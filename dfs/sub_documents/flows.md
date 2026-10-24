This document describes the flows (actions) among components within the system as well as in-charge packets. Packet formats: [packets.md](packets.md).

# Initialization

## 1. Associated nodes

- DNS
- Master
- Data

## 2. Packets

- `Notify`: Master → DNS, Data → Master
- `AskIP`: Data → DNS
- `AskIPAck`: DNS → Data

## 3. Flows

### 3.1. Master

1. Master sends Notify to DNS [`Notify`]

### 3.2. Data

1. Data sends AskIP to DNS [`AskIP`]
2. DNS replies with the socket address of the current Master [`AskIPAck`]
3. Data sends Notify to Master [`Notify`]

## 4. Special cases

### 4.1. DNS does not know the Master yet

- DNS replies with an empty `AskIPAck` payload

### 4.2. Backup Master starts

- New Master sends Notify to the current Master [`Notify`]

# Read data

## 1. Associated nodes

- Client
- DNS
- Current Master
- 1 Data node

## 2. Packets

- `AskIP`: Client → DNS
- `AskIPAck`: DNS → Client
- `RequestFromClient`: Client → Current Master, Client → Data node
- `ResponseNodeIp`: Current Master → Client
- `DataNodeSendData`: Data node → Client
- `ClientRequestAck`: Data node → Current Master

## 3. Flows

1. Client connects to DNS to ask the IP of current Master [`AskIP`]
2. DNS responds with the IP [`AskIPAck`]
3. Client connects to the current Master to ask to read a specific file [`RequestFromClient`]
4. Master responds with the IP of the Data node holding a file replica [`ResponseNodeIp`]
5. Client connects to the Data node to get the file [`RequestFromClient`]
6. Data node responds with the file binary [`DataNodeSendData`]
7. Data node sends ACK to the Master for finishing the reading process [`ClientRequestAck`]

## 4. Special cases

### 4.1. File is being written

- No client can read a file while another client is writing it ([Consistency guarantee](../specification.md#consistency-guarantee))

# Send data

## 1. Associated nodes

- Client
- DNS
- Current Master
- 2 Data nodes

## 2. Packets

- `AskIP`: Client → DNS
- `AskIPAck`: DNS → Client
- `RequestFromClient`: Client → Current Master
- `ResponseNodeIp`: Current Master → Client
- `ClientUpload`: Client → Receiving node
- `ClientRequestAck`: Receiving node → Current Master
- `ClientUploadAck`: Receiving node → Client
- [Replication](#replication) packets

## 3. Flows

1. Client connects to DNS to ask the IP of current Master [`AskIP`]
2. DNS responds with the IP [`AskIPAck`]
3. Client connects to the current Master to ask to write a specific file [`RequestFromClient`]
4. Master responds with the IP of a receiving node (either a Data node or the Master) that is ready to receive the file [`ResponseNodeIp`]
5. Client connects to the receiving node to write the file [`ClientUpload`]
6. When the client finishes writing, the receiving node:
   1. notifies the Master that the writing process is done [`ClientRequestAck`]
   2. sends ACK to the client [`ClientUploadAck`]
7. The current Master selects a suitable Data node, different from the one that just received the file, and triggers [Replication](#replication)

## 4. Special cases

### 4.1. Master designates itself

- In step 4, Master can designate itself or another Data node inside the network and embeds that node's address in `ResponseNodeIp`

### 4.2. File is being read or written by another client

- Master permits a single writer and no readers while a file is being written ([Consistency guarantee](../specification.md#consistency-guarantee))

# Heartbeat

## 1. Associated nodes

- Current Master
- Backup Master
- All Data nodes

## 2. Packets

- `Heartbeat`: Current Master → Data node, Current Master → Backup Master, Backup Master → Current Master
- `HeartbeatAck`: Data node → Current Master, Current Master → Backup Master, Backup Master → Current Master

## 3. Flows

Periodically, the current Master checks the health status of each Data node, and the current and backup Master check each other.

1. Node A sends heartbeat to B [`Heartbeat`]
2. B sends ACK back to A [`HeartbeatAck`]

## 4. Special cases

### 4.1. No response to 2 consecutive heartbeats

- Data node down: the current Master triggers [Fault tolerance on Data node](#fault-tolerance-on-data-node)
- Current Master down: the backup Master triggers [Fault tolerance on Master node](#fault-tolerance-on-master-node)

# Fault tolerance on Data node

## 1. Associated nodes

- Current Master
- 2 Data nodes

## 2. Packets

- `Heartbeat`, `HeartbeatAck`: detect the down node
- [Replication](#replication) packets

## 3. Flows

1. After 2 heartbeats from the current Master that the Data node (denote A) doesn't respond to, the current Master looks up, for each file X held by A, which Data node (denote B) is holding a replica
2. For each X and B, the current Master triggers [Replication](#replication)

# Fault tolerance on Master node

## 1. Associated nodes

- Current Master
- Backup Master
- All Data nodes
- DNS

## 2. Packets

- `Heartbeat`, `HeartbeatAck`: detect the down Master
- `Notify`: new Master → DNS

## 3. Flows

1. After 2 heartbeats from the backup Master that the current Master doesn't respond to, the backup Master notifies all Data nodes that it is now the current Master [packet not specified]
2. New current Master updates its IP to the DNS server [`Notify`]

# State synchronization

## 1. Associated nodes

- Current Master
- Backup Master

## 2. Packets

- `StateSync`: Current Master → Backup Master
- `StateSyncAck`: Backup Master → Current Master

## 3. Flows

1. Every **30 seconds**, the current Master synchronizes its current state to the backup Master [`StateSync`]
2. After receiving, the backup Master sends ACK [`StateSyncAck`]

# Replication

## 1. Associated nodes

- Current Master
- `N` Data nodes: `N` is applied universally across the network and is specified at start-up

## 2. Packets

- `RequestSendReplica`: Current Master → Data node
- `SendReplica`: Data node → Data node
- `SendReplicaAck`: Data node → Current Master

## 3. Flows

Triggered in 2 scenarios:

- A file is newly written (uploaded) by a client
- A Data node is down

Assume 2 Data nodes, A and B, where A holds the file X to be replicated:

1. Current Master selects a new suitable Data node to store the file
2. Current Master notifies A to send X to B [`RequestSendReplica`]
3. A sends X to B [`SendReplica`]
4. Both A and B send ACK to the current Master [`SendReplicaAck`]
5. Master updates the in-memory database about which nodes are holding X

Data node selection: Master chooses the Data node holding the fewest files. The same strategy applies in both Send data and Replication.

## 4. Special cases

### 4.1. Fewer nodes than `N`

- Re-run Replication when a new node joins (planned, see [BACKLOG](../BACKLOG.md))

# Graceful shutdown

## 1. Associated nodes

- Every node (Master, Data) and client

## 2. Packets

- `GracefulShutdown`: Front-end → Master / Data node

## 3. Flows

Each node (Master, Data) and client holds 3 threads, initialized at start-up:

- Processor thread (**P**): handles every logic
- Receiver thread (**R**): receives packets sent from other nodes/clients
- Sender thread (**S**): delivers packets crafted by the Processor thread

Graceful shutdown is triggered by **P**, e.g. when the front-end sends `GracefulShutdown`. A shutdown flag **F** is shared among the 3 threads: only **P** can modify it, the other threads read it. In parallel, **S** and **R** each shut down in their own way.

### 3.1. Receiver thread

The main loop of **R** receives incoming packets with `listener.incoming()`:

1. **P** sets **F** to true
2. **P** connects to **R** with `TcpStream::connect`
3. The connection from **P** wakes **R** from the `listener.incoming()` loop; **R** reads **F** and breaks the loop

### 3.2. Sender thread

The main loop of **S** is an infinite loop; at the beginning of the loop body, **S** checks with a timeout whether **P** has sent a packet via the channel. Therefore, **P** only needs to set the flag:

1. **P** sets **F** to true
2. **S** reads **F** at the beginning of the loop body and breaks the loop
