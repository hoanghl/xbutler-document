# 1. What the rewrite needs to address compared with the existing implementation

- **Use TypeScript**: simpler than Rust, one codebase for Node (CLI) and React Native (Android); performance is not a goal
- **Importable library**: `dfs-core` is imported by `dfs-cli` and `xbutler-fe`
- **Keep the existing protocol as implemented in Rust**; update [packets.md](../dfs/sub_documents/packets.md) where it differs:
  - Heartbeat carries a 2-byte port (doc: empty payload)
  - Notify is `<role><port>`, IP taken from the socket (doc: `<role><IP><port>`)
  - HeartbeatAck goes to the Master, not the sender
- **Android only** for the mobile node; no iOS
- **Fix problems in the Rust code** (paths in the Rust `dfs` repo, under `src/components/`):
  - Master and Data each have a copy of the upload and replication handlers, and the copies have drifted: Master's `SendReplica` re-reads its own disk instead of using the received bytes (`entity/master.rs:329-342`)
  - `file_info` keys on `filename` with one `node` column, so the Master cannot record two replicas (`inmem_db/types.rs:40-45`)
  - `panic!` / `exit(1)` on bad input (BACKLOG: "Remove all panic")
  - `RequestFromClient` parser drops the last filename byte (`packets.rs:516`)
  - Protocol duplicated in Kotlin (`Packet.kt`, `TCP.kt`); JNI `libdfs.so` + NDK-built `libsqlite3.so`
- **Current features of the Rust implementation**:
  - Done: initialization (DNS, Notify), heartbeat Master → Data, write, replication after write, graceful shutdown, Data node in an Android foreground service
  - Not done: read, read/write consistency, missed-heartbeat detection, Data / Master fault tolerance, backup Master, StateSync

# 2. Design

## 2.1. Repo design

- One package with a mostly flat `src/`: no monorepo, no layer folders, no adapter packages

```
dfs-core/
├── package.json          "type": "module", "exports" (types + default), "files": ["dist"], "sideEffects": false
├── tsconfig.json         strict, "lib": ["ES2022"], "types": []
├── README.md, CHANGELOG.md, LICENSE
├── .github/workflows/ci.yml   typecheck → test → build → publint + attw
├── src/
│   ├── index.ts          only public entry: createNode, createClient, types
│   ├── platform.ts       Transport, BlobStore, Clock, Logger
│   ├── types.ts          NodeAddress, Role, NodeConfig
│   ├── protocol/         packets.ts, codec.ts, bytes.ts (big-endian ints, UTF-8; no Buffer)
│   ├── cluster.ts        members, heartbeats, 2 missed → down
│   ├── catalog.ts        file → replica holders, least-loaded placement
│   ├── roles/            storage.ts (Master + Data), master.ts, data.ts, dns.ts
│   ├── node.ts           createNode: frame → decode → queue → handler
│   └── client.ts         createClient: put / get
└── test/
    ├── memory-platform.ts   fake network + manual clock
    ├── codec.test.ts        reference bytes from the Rust build, inline hex
    └── cluster.test.ts      multi-node flows in one process
```

- Build with `tsc` (JS + `.d.ts`, no bundler); test with `bun test`
- Platform code lives in the apps, ~100 lines each:
  - `dfs-cli`: `node:net`, `node:fs/promises`
  - `xbutler-fe`: `react-native-tcp-socket`, `expo-file-system`
  - Move to a shared package only when a second app needs the same one
- Distribution: versioned builds on npm / GitHub Packages; `bun link` while co-developing (Metro may need `watchFolders`)
- Grow per file: a file past ~400 lines, or a new backlog area (auth, dynamic roles), becomes a folder

## 2.2. Architecture: sans-I/O with injected I/O

- Core owns encoding, cluster state, handlers and the frame loop; it never implements I/O
- App supplies I/O through 4 interfaces in `platform.ts`: `Transport`, `BlobStore`, `Clock`, `Logger`
- Rules:
  - Core reaches outside only through `platform`; tsconfig makes `Buffer`, `node:*`, `setTimeout`, `console` compile errors
  - Interfaces carry bytes and plain values, never packets; protocol knowledge lives only in core
  - Time is injected (`clock.every`, `clock.now`), so timeouts are testable
  - One frame at a time through a queue, so handlers cannot interleave at `await`
  - Adapters fail by rejecting; core decides (log, retry, mark node down); no panics
- Roles are handler sets: Master = storage + master, Data = storage + data, DNS = dns
- Transport contract:
  - One TCP connection per frame, read to EOF, report the peer's IPv4 (strip `::ffff:`)
  - Every participant listens, including CLI client and phone: replies arrive on a new connection to the sender's port
- Rejected alternatives:
  - Strict sans-I/O (`receive() → actions`): flows do I/O mid-way (read → send, save → ack), so each step becomes an action + result event
  - Full DDD (bounded contexts × layers) or full hexagonal layout: too much structure for a small library
  - Kept from DDD: value objects (`NodeAddress`, `FileName`), file → replica-set model; kept from hexagonal: `platform.ts`

## 2.3. Android node

- Run the node as a foreground service + Headless JS task
  - Service alone keeps the process, not JS timers: RN stops timers on `onHostPause` unless a Headless JS task runs (bridgeless fix in RN 0.76+)
- Reuse `DFSService.kt`: extend `HeadlessJsTaskService`, `allowedInForeground = true`, timeout `0` (believed to mean no timeout; verify)
- Register the task in a custom entry file that then imports `expo-router/entry`
- Service type `specialUse` (no time limit); avoid `dataSync` (6 h / 24 h for targetSdk 35+) and `shortService` (~3 min)
  - Google Play needs `PROPERTY_SPECIAL_USE_FGS_SUBTYPE`; sideloading does not
- Start from the UI (Android 12+ blocks starts from background)
- Node and UI share the JS thread: keep socket and file I/O native, avoid copying large byte arrays in JS
- Removes `UDS.kt`, `TCP.kt`, `Packet.kt`, `libdfs.so`, `libsqlite3.so`, NDK build; status, logs, stop become JS calls and events
- Ready-made alternative: `react-native-background-actions` ≥ 4.1.0
- Validate on a real phone:
  - Screen off 10+ min (`adb shell dumpsys deviceidle force-idle`)
  - Swipe app from recents
  - Partial wake lock held (`adb shell dumpsys power`)
  - Inbound Wi-Fi connections with screen off (else add a `WifiLock`)
  - OEM battery killers

# 3. Milestones

- Each function adds the packets, handlers, state and `dfs-cli` command it needs; each packet lands with a reference-byte test and its `packets.md` fix
- **M1 – Basic functions** (Node + `dfs-cli`):
  1. Initialization: `AskIp`, `AskIpAck`, `Notify`; codec, `node.ts` loop, first handlers in `roles/dns.ts`, `master.ts`, `data.ts`, members in `cluster.ts`, Node + memory platforms
  2. Heartbeat: `Heartbeat`, `HeartbeatAck`; Master timer (`clock.every`), `lastSeen` in `cluster.ts`
  3. Write: `RequestFromClient`, `ResponseNodeIp`, `ClientUpload`, `ClientUploadAck`, `ClientRequestAck`; `roles/storage.ts`, `catalog.ts` + placement, `client.put`
  4. Replication: `RequestSendReplica`, `SendReplica`, `SendReplicaAck`; replica sets in `catalog.ts`
  5. Read: `DataNodeSendData`, read side of `RequestFromClient`; `client.get`
  6. Graceful shutdown: `GracefulShutdown`; `node.stop()`
  - Done when: an all-TS cluster writes, replicates and reads a file; TS and Rust nodes interoperate on 1–4 and 6
- **M2 – Advanced functions**:
  1. Read/write consistency: many readers or one writer, in `catalog.ts`; no new packets
  2. Data fault tolerance: 2 missed heartbeats → down → re-replicate; reuses replication packets
  3. State sync: `StateSync`, `StateSyncAck`; serialize `cluster.ts` + `catalog.ts`
  4. Master fault tolerance: backup Master role, Master ↔ backup heartbeat, promotion + `Notify` to DNS
  - Done when: every flow in [flows.md](../dfs/sub_documents/flows.md) and the consistency rule in [specification.md](../dfs/specification.md) pass in an all-TS cluster
- **M3 – Android**: RN transport in a Headless JS foreground service; `xbutler-fe` runs the TS Data node; delete `libdfs.so`, `libsqlite3.so`, `UDS.kt`, `TCP.kt`, `Packet.kt`
  - Done when: the phone checks in 2.3 pass

# References

- [Headless JS](https://reactnative.dev/docs/headless-js-android)
- [Fix timers in headless tasks on bridgeless mode](https://github.com/react/react-native/pull/47496)
- [Foreground service types](https://developer.android.com/develop/background-work/services/fgs/service-types)
- [react-native-background-actions](https://github.com/Rapsssito/react-native-background-actions)
- [Network protocols, sans I/O](https://snarky.ca/network-protocols-sans-i-o/)
