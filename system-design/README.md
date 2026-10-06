# System Design

An interview-prep guide, not a book — three parts: what things are and why (`basics/`), the
same but for more advanced topics (`advanced/`), and real designs actually practiced for
interviews (`designs/`).

## Basics

### Fundamentals

- [x] [Fundamentals](<./basics/fundamentals/fundamentals.md>) — scaling, load balancing, DNS, CDN, caching, Redis, and terms worth knowing

### Databases

- [x] [SQL vs NoSQL](<./basics/databases_part1/sql_vs_nosql.md>)
- [x] [Database Indexing](<./basics/databases_part1/database_indexing.md>)
- [x] [Database Partitioning](<./basics/databases_part1/database_partitioning.md>)
- [x] [Database Replication](<./basics/databases_part1/database_replication.md>)
- [x] [Database Sharding](<./basics/databases_part1/database_sharding.md>)
- [x] [Consistent Hashing](<./basics/databases_part2/consistent_hashing.md>)
- [x] [CAP Theorem](<./basics/databases_part2/cap_theorem.md>)
- [x] [ACID vs BASE](<./basics/databases_part2/acid_base.md>)
- [x] [Strong vs Eventual Consistency](<./basics/databases_part2/strong_vs_eventual_consistency.md>)
- [x] [Distributed Transactions](<./basics/databases_part2/distributed_transactions.md>)

### Communications

- [x] [REST vs GraphQL vs gRPC](<./basics/communications/rest_graphql_grpc.md>)
- [x] [Websockets](<./basics/communications/websockets.md>)
- [x] [Messaging Queues](<./basics/communications/messaging_queues.md>)
- [x] [Kafka](<./basics/communications/kafka.md>)

### API Infrastructure

- [x] [API Gateway](<./basics/API_infrastructure/API_Gateway.md>)
- [x] [Rate Limiting](<./basics/API_infrastructure/rate_limiting.md>)
- [x] [Authentication](<./basics/API_infrastructure/authentication.md>)
- [x] [Bloom Filter](<./basics/API_infrastructure/bloom_filter.md>)
- [x] [Monolith vs Microservices](<./basics/API_infrastructure/monolith_vs_microservice.md>)

## Advanced

### Architectural Patterns

- [x] [Event-Driven Architecture](<./advanced/architectural_patterns/event_driven_architecture.md>)
- [x] [CQRS](<./advanced/architectural_patterns/CQRS.md>)
- [x] [Event Sourcing](<./advanced/architectural_patterns/event_sourcing.md>)
- [x] [Saga Pattern](<./advanced/architectural_patterns/saga_pattern.md>)
- [x] [Circuit Breaker](<./advanced/architectural_patterns/circuit_breaker.md>)

### Reliability & Observability

- [x] [High Availability](<./advanced/reliability_observability/high_availability.md>)
- [x] [Fault Tolerance](<./advanced/reliability_observability/fault_tolerance.md>)
- [x] [Distributed Locking](<./advanced/reliability_observability/distributed_locking.md>)
- [x] [Monitoring and Alerting](<./advanced/reliability_observability/monitoring_alerting.md>)
- [x] [Service Discovery](<./advanced/reliability_observability/service_discovery.md>)

### Data Partitioning & Storage

- [x] [Data Partitioning Strategies](<./advanced/data_partitioning/data_partitioning_strategies.md>)
- [x] [Time Series Databases](<./advanced/data_partitioning/time_series_DBs.md>)
- [x] [Data Lakes vs Warehouses](<./advanced/data_partitioning/data_lakes_and_warehouses.md>)
- [x] [Search Infrastructure](<./advanced/data_partitioning/search_infrastructure.md>)
- [x] [Geospatial Data](<./advanced/data_partitioning/geospatial_data.md>)

## Designs

Real system designs practiced for interviews — how the pieces above actually get put together,
plus how to talk through them out loud. Each design is one markdown file, diagram(s) alongside
it in this folder.

- [x] [URL Shortener](<./designs/url-shortener.md>)
