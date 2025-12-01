<p style="text-align: center;font-size: 40px;font-weight: 900">xButler app</p>

# Feature

## Feat: Fetch DFS status

<p align="center">
    <img src="res/feat-fetch-status.png" width="800">
</p>

**_Front-end_** periodically invokes native function to fetch DFS status. **_Native module_** then sends health check packet to DFS service. DFS protocol is used, particularly packet `StatusService` and `StatusServiceAck`.

Periodic invocation in **_Front-end_** is done by hook `useInterval` with _5-second_ interval. Additionally, **_Front-end_** only runs this flow when the screen is DFS only.
