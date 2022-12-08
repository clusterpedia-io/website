---
title: "Standalone TCP for Health Checker"
weight: 3
---
When client-go creates any number of Clients with the same configuration, such as certificates, it reuses the same TCP connection.
https://github.com/kubernetes/kubernetes/blob/3f823c0daa002158b12bfb2d53bcfe433516659d/staging/src/k8s.io/client-go/transport/transport.go#L54

This results in the cluster health check interface using the same TCP connection as the resource synchronized informer, which may cause TCP blocking and increased health check latency if a large number of informers are started for the first time.
We add a feature gate - `HealthCheckerWithStandaloneTCP` to allow users to use a standalone tcp for health checks

```bash
./clustersynchro-manager --feature-gates=HealthCheckerWithStandaloneTCP=true
```

|desc|feature gates|default
|-----|-------------| ------ |
| Use standalone tcp for health checker | `HealthCheckerWithStandaloneTCP` |false|

**Note: When this feature is turned on, the TCP long connections to member clusters will change from 1 to 2.**
If 1000 clusters are imported, then ClusterSynchro Manager will keep 2000 TCP connections.
