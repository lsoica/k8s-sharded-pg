# Overview
```sh
                       ┌─────────────────────┐
                       │                     │
                       │        App          │
                       │                     │
                       └──────────┬──────────┘
                                  │
                                  │
                       ┌──────────▼──────────┐
                       │                     │
                       │     Load Balancer   │
                       │                     │
        ┌──────────────┴───────────┬─────────┴─────────────────┐
        │                          │                           │
        │                          │                           │
        │                          │                           │
        │                          ▼                           │
┌───────▼────────┐        ┌────────────────┐          ┌────────▼───────┐
│                │        │                │          │                │
│                │        │                │          │                │
│ Coordinator 1  │        │  Coordinator 2 │          │  Coordinator 3 │
│                │        │                │          │                │
│                │        │                │          │                │                
└──────┬──┬─┬────┴────────┴─────────┬──┬─┬─┴──────────┴───────┬┬─┬─────┘
       │  │ │                       │  │ │                    ││ │
       │  │ │                       │  │ │                    ││ │
       │  │ │                       │  │ │                    ││ │
  ┌────▼──▼─▼┌─┐               ┌────▼──▼─▼┌─┐              ┌──▼▼─▼────┬─┐
┌─┼──────────┤ │             ┌─┼──────────┤ │            ┌─┼──────────┤ │
│ │          │ │             │ │          │ │            │ │          │ │
│ │  Data +  │ │             │ │  Data +  │ │            │ │  Data +  │ │
│ │  Replica │ │             │ │  Replica │ │            │ │  Replica │ │
│ │          │ │             │ │          │ │            │ │          │ │
│ ├──────────┼─┘             │ ├──────────┼─┘            │ ├──────────┼─┘
└─┴──────────┘               └─┴──────────┘              └─┴──────────┘

```
# Options

## PL Proxy

## Postgres-XL

## Built-in sharding with postgres_fdw and partitioning
### Declarative partitioning
#### Limitations
- Unique constraints (and hence primary keys) on partitioned tables must include all the partition key columns. This limitation exists because the individual indexes making up the constraint can only directly enforce uniqueness within their own partitions; therefore, the partition structure itself must guarantee that there are not duplicates in different partitions.
- There is no way to create an exclusion constraint spanning the whole partitioned table. It is only possible to put such a constraint on each leaf partition individually. Again, this limitation stems from not being able to enforce cross-partition restrictions.
 - BEFORE ROW triggers on INSERT cannot change which partition is the final destination for a new row.
- Mixing temporary and permanent relations in the same partition tree is not allowed. Hence, if the partitioned table is permanent, so must be its partitions and likewise if the partitioned table is temporary. When using temporary relations, all members of the partition tree have to be from the same session.

### Partitioning Using Inheritance

#### Limitations
- There is no automatic way to verify that all of the CHECK constraints are mutually exclusive. It is safer to create code that generates child tables and creates and/or modifies associated objects than to write each by hand.
- Indexes and foreign key constraints apply to single tables and not to their inheritance children, hence they have some caveats to be aware of.
- The schemes shown here assume that the values of a row's key column(s) never change, or at least do not change enough to require it to move to another partition. An UPDATE that attempts to do that will fail because of the CHECK constraints. If you need to handle such cases, you can put suitable update triggers on the child tables, but it makes management of the structure much more complicated.
- INSERT statements with ON CONFLICT clauses are unlikely to work as expected, as the ON CONFLICT action is only taken in case of unique violations on the specified target relation, not its child relations.
- Triggers or rules will be needed to route rows to the desired child table, unless the application is explicitly aware of the partitioning scheme. Triggers may be complicated to write, and will be much slower than the tuple routing performed internally by declarative partitioning.

Missing features:
- GTM - global transaction manager
- GSM - global snapshot manager

## Deploy PostgreSQL with zalando's operator

```sh
git clone https://github.com/zalando/postgres-operator.git
cd postgres-operator
helm install postgres-operator ./charts/postgres-operator
helm install postgres-operator-ui ./charts/postgres-operator-ui
kubectl port-forward svc/postgres-operator-ui 8081:80
```

Script to connect to first cluster:
```sh
#!/bin/sh

kubectl port-forward pod/acid-cluster1-0 5432:5432 &
P_F_PID=$!

export PGPASSWORD=$(kubectl get secret postgres.acid-cluster1.credentials.postgresql.acid.zalan.do -o jsonpath={.data.password} | base64 -d)
export PGSSLMODE=require

psql -U postgres
kill $P_F_PID
```

Script to connect to second cluster:
```sh
#!/bin/sh

kubectl port-forward pod/acid-cluster2-0 5432:5432 &
P_F_PID=$!

export PGPASSWORD=$(kubectl get secret postgres.acid-cluster2.credentials.postgresql.acid.zalan.do -o jsonpath={.data.password} | base64 -d)
export PGSSLMODE=require

psql -U postgres
kill $P_F_PID
```

PostgreSQL hostnames:
```sh
acid-cluster1-0.acid-cluster1.default.svc.cluster.local
acid-cluster2-0.acid-cluster2.default.svc.cluster.local
acid-cluster3-0.acid-cluster3.default.svc.cluster.local
acid-cluster4-0.acid-cluster4.default.svc.cluster.local

INSERT INTO shard.shard_config values(1,'server_0','acid-cluster2-0.acid-cluster2.default.svc.cluster.local',5432,'postgres', 'RkpSw9WFTUSq090GCtXDLVCeJQCO8Xv4EyglRc3WNS2ZRgjLn21iZybnuRC8tR5c');
INSERT INTO shard.shard_config values(2,'server_1','acid-cluster3-0.acid-cluster3.default.svc.cluster.local',5432,'postgres', 'YYAZ08YweH6CdVNYp35QypAWtKpuGW3ELmf664Uda2lBZyGFp32Qn39B45VwRdS2');
INSERT INTO shard.shard_config values(3,'server_2','acid-cluster4-0.acid-cluster4.default.svc.cluster.local',5432,'postgres', 'fKhbEBhlZie7Sp6nmrO747X3vjPLJ0IFy8XJGavaa0pJmDZrZpGauxJbRWg2pilz');

```

## Cluster CRD
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: shardedpostgresqls.formuladb.io
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: formuladb.io
  # list of versions supported by this CustomResourceDefinition
  versions:
    - name: v1
      # Each version can be enabled/disabled by Served flag.
      served: true
      # One and only one version must be marked as the storage version.
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                shards:
                  type: integer
  # either Namespaced or Cluster
  scope: Namespaced
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: shardedpostgresqls
    # singular name to be used as an alias on the CLI and for display
    singular: shardedpostgresql
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: ShardedPostgreSQL
    # shortNames allow shorter string to match your resource on the CLI
    shortNames:
    - spg
```
## References

- [pgDash on sharding](https://pgdash.io/blog/postgres-11-sharding.html)
