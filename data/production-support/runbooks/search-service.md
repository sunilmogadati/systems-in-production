# Search Service Runbook

## Overview
Elasticsearch-based product search and filtering.
Cluster: 3 data nodes, 2 master nodes.

## Common Issues

### Slow Search Response
1. Check cluster health: `curl http://es-cluster.internal:9200/_cluster/health`
2. Green = OK, Yellow = degraded (missing replicas), Red = data loss risk
3. Check JVM heap: `curl http://es-cluster.internal:9200/_nodes/stats/jvm`
4. If heap > 85%, GC pressure is likely causing latency spikes.

### OOM (Out of Memory) Crashes
1. Check JVM heap settings: should be 50% of available RAM, max 31GB.
2. Common cause: large aggregation queries from analytics dashboard.
3. Restart procedure: `kubectl rollout restart statefulset/search-es-data`
4. After restart, verify shard allocation: `curl http://es-cluster.internal:9200/_cat/shards?v`
5. Reindex may be needed if shards are unassigned.

### Memory Leak Pattern
- Symptom: Memory usage climbs steadily over 3-5 days, then OOM crash.
- Root cause: Elasticsearch field data cache not bounded.
- Fix: Set `indices.fielddata.cache.size: 40%` in elasticsearch.yml
- This was identified in Q4 2025 but the fix was reverted during a cluster upgrade.

## Escalation
- P1/P2: Page Discovery oncall + SRE
- P3/P4: Create ticket, assign to Discovery team
