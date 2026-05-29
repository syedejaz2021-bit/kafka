# Wiring the Trino Kafka connector

Trino's Kafka connector lets you query Kafka topics as SQL tables. The bootstrap
servers point at the Strimzi bootstrap Service. Because the Kafka cluster is in
the `kafka` namespace and Trino is in `trino-stack`, use the cross-namespace DNS:

    events-kafka-kafka-bootstrap.kafka.svc.cluster.local:9092

## What to add to the Trino `trino-config` ConfigMap

Add a new catalog properties block alongside the existing mysql/postgresql/mongodb ones:

```yaml
  kafka.properties: |
    connector.name=kafka
    kafka.nodes=events-kafka-kafka-bootstrap.kafka.svc.cluster.local:9092
    kafka.table-names=log-events,deployments,incidents
    kafka.hide-internal-columns=false
    # Where Trino looks for table description JSON files (optional but recommended)
    kafka.table-description-dir=/etc/trino/kafka
```

If you want schemas/decoders for the JSON messages, also add table description
files (see `kafka-table-descriptions/` in this bundle) and mount them at
`/etc/trino/kafka`.

## Volume mount additions to the Trino Deployment

In the `volumeMounts` list of the trino container, add:

```yaml
        - { name: catalogs, mountPath: /etc/trino/catalog/kafka.properties, subPath: kafka.properties }
        - { name: kafka-tables, mountPath: /etc/trino/kafka }
```

In the `volumes` list, add the kafka.properties key to the existing `catalogs`
configMap items, and add a new volume for table descriptions:

```yaml
      - name: catalogs
        configMap:
          name: trino-config
          items:
          - { key: mysql.properties,      path: mysql.properties }
          - { key: postgresql.properties, path: postgresql.properties }
          - { key: mongodb.properties,    path: mongodb.properties }
          - { key: kafka.properties,      path: kafka.properties }   # <-- ADD
      - name: kafka-tables                                            # <-- ADD whole block
        configMap:
          name: trino-kafka-tables
          optional: true
```

## After applying

Restart Trino so it picks up the new catalog:

```bash
kubectl rollout restart deploy/trino -n trino-stack
```

Then verify in the Trino tab of your dashboard:

```sql
SHOW CATALOGS;
-- should now include 'kafka'

SHOW TABLES FROM kafka.default;
-- should list log-events, deployments, incidents

SELECT COUNT(*) FROM kafka.default."log-events";
-- counts messages currently in the topic (within retention window)
```

## Important caveats

1. **Kafka is a stream, not a table.** Each query reads the messages currently
   retained in the topic (last 7 days per our retention config). It is NOT a
   persistent store like the other catalogs. Old messages age out.

2. **Cross-namespace network policy.** If you have NetworkPolicies restricting
   traffic, allow trino-stack → kafka on port 9092. Most clusters allow all
   namespace-to-namespace traffic by default; only an issue if you've locked it down.

3. **Message format.** By default Trino reads Kafka messages as raw bytes. To
   query fields (e.g. service_id, level), you need a table description JSON that
   tells Trino the message is JSON and maps fields to columns. See the
   `kafka-table-descriptions/` folder.
