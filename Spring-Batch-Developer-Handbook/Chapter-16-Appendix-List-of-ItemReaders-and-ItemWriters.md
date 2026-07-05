# Chapter 16 - Appendix: List of ItemReaders and ItemWriters

## Introduction
Spring Batch lo `ItemReader` and `ItemWriter` interfaces chala central role play chesthayi. Data ekkada nunchi vastondi, ela parse avtondi and ekkadiki velli save avtondi anedi ee implementations meede depend ayyi untundi. Spring Batch box bayata chala default readers and writers ni provide chestundi. Ee appendix lo manam unna anni important item readers and writers list, vati descriptions, and vi thread-safe or kaada ani table format lo chuddam.

## Why this feature exists
Prathi developer flat file reading, database paging, xml parsing lanti common problems ki malli malli code raayakunda undadaniki Spring Batch velle robust ga test chesina standard implementations ni provide chestundi. Idi telusukovadam valla development time chala varaku save avthundi.


### Note on Modern File Formats
With the rise of modern data formats, Spring Batch provides extensive support for JSON and Avro out of the box.
- `JsonItemReader` parses a JSON array of objects using either Jackson or Gson.
- `AvroItemReader` and `AvroItemWriter` serialize and deserialize data efficiently using the Apache Avro framework, providing compact binary formats.

---

## 1. Item Readers

Kindha unna table lo Spring Batch provide chese Item Readers list and vaati attributes unnai:

| Item Reader | Description | Thread-safe |
| :--- | :--- | :--- |
| **AbstractItemStreamItemReader** | `ItemStream` and `ItemReader` ni combine chese base class. | Yes |
| **AbstractItemCountingItemStreamItemReader** | Number of items ni count chesi basic restart capabilities iche abstract base class. | No |
| **AbstractPagingItemReader** | Basic paging features iche base class. | No |
| **AbstractPaginatedDataItemReader** | Spring Data paginated facilities meeda base ayyi unna paging class. | No |
| **AggregateItemReader** | Oka list of items ni group chesi oka single item (collection) ga pampisthundi (custom record boundaries). | Yes |
| **AmqpItemReader** | Spring `AmqpTemplate` vaadi AMQP (RabbitMQ) nunchi synchronous ga messages read chestundi. | Yes |
| **KafkaItemReader** | Apache Kafka topic loni partitions nunchi read chestundi. Offsets ni execution context lo save chestundi (restarts kosam). | No |
| **FlatFileItemReader** | CSV, TXT lanti flat files nunchi read chestundi. Restart and skip properties untayi. | No |
| **ItemReaderAdapter** | Ye class ni aina `ItemReader` interface ki adapt chestundi. | Yes |
| **JdbcCursorItemReader** | JDBC `ResultSet` (cursor) vaadi data ni stream chestundi. | No |
| **JdbcPagingItemReader** | SQL statement tho large datasets ni memory lopu pages laaga read chestundi. | Yes |
| **JmsItemReader** | `JmsOperations` vaadi JMS queues nunchi messages theeskuntundi. | Yes |
| **JpaCursorItemReader** | JPQL query execute chesi result set meeda iterate avthundi. | No |
| **JpaPagingItemReader** | JPQL query ni use chesi database nunchi paging format lo read chestundi. | Yes |
| **ListItemReader** | Memory lo unna `List` nunchi okko item isthundi. (Mainly for testing). | No |
| **MongoPagingItemReader** | `MongoOperations` vaadi MongoDB nunchi paginated data theeskuntundi. | Yes |
| **MongoCursorItemReader** | MongoDB cursor dwara items stream chestundi. | Yes |
| **RepositoryItemReader** | Spring Data `PagingAndSortingRepository` vaadi DB nunchi read chestundi. | Yes |
| **StoredProcedureItemReader** | Database stored procedure execute chesi vachina cursor tho data read chestundi. | No |
| **StaxEventItemReader** | StAX vaadi XML files ni read chestundi. | No |
| **JsonItemReader** | JSON document loni objects ni read chestundi. | No |
| **AvroItemReader** | Serialized Avro objects unna resource nunchi read chestundi. | No |
| **LdifReader** / **MappingLdifReader**| LDAP LDIF files nunchi data thechi attributes or Domain objects loki map chestundi. | No |

---

## 2. Item Writers

ItemWriters data ni final target ki push cheyadaniki help avthayi.

| Item Writer | Description | Thread-safe |
| :--- | :--- | :--- |
| **AbstractItemStreamItemWriter** | `ItemStream` and `ItemWriter` interfaces ni combine chese abstract base class. | Yes |
| **AmqpItemWriter** | Spring `AmqpTemplate` vaadi AMQP/RabbitMQ broker ki messages send chestundi. | Yes |
| **CompositeItemWriter** | Oka item ni theesukoni multiple injected item writers ki pampisthundi (e.g., both DB and File ki okesari rayadam). | Yes |
| **FlatFileItemWriter** | Items ni flat files (CSV, TXT) ki raasthundi. | No |
| **ItemWriterAdapter** | Vere edaina service class method ni `ItemWriter` laaga work ayyela adapt chestundi. | Yes |
| **JdbcBatchItemWriter** | `PreparedStatement` batching features use chesi SQL queries dwara database ki items raasthundi. | Yes |
| **JmsItemWriter** | `JmsOperations` vaadi JMS destination ki items raasthundi. | Yes |
| **JpaItemWriter** | JPA `EntityManager` aware. Items ni `merge` or `persist` chestundi. | Yes |
| **KafkaItemWriter** | `KafkaTemplate` vaadi Apache Kafka topic ki data push chestundi. | No |
| **MimeMessageItemWriter** | `MimeMessage` type unna items ni JavaMailSender dwara emails ga send chestundi. | Yes |
| **MongoItemWriter** | `MongoOperations` vaadi items ni MongoDB lo save chestundi. | Yes |
| **PropertyExtractingDelegatingItemWriter**| Item loni specific properties ni extract chesi, vere oka method ki arguments laaga delegate chestundi. | Yes |
| **RepositoryItemWriter** | Spring Data `CrudRepository` vaadi items ni DB lo save chestundi. | Yes |
| **StaxEventItemWriter** | `Marshaller` vaadi items ni XML file loki raasthundi. | No |
| **JsonFileItemWriter** | `JsonObjectMarshaller` vaadi Json file loki data raasthundi. | No |
| **AvroItemWriter** | Avro formatting use chesi WritableResource loki data serialize chestundi. | No |
| **ListItemWriter** | In-memory `List` loki items push chestundi (Testing or internal tracking kosam). | No |

---

## Best Practices
- **Multithreading**: Thread-safe "No" ani unna (e.g., `FlatFileItemReader`) components ni partitioning leda multi-threaded steps lo direct ga vaadakudadu. Vaadi theerali ante `SynchronizedItemStreamReader` wrapper vaadali.
- **Paging vs Cursor**: Thread-safe environments lo `JdbcPagingItemReader` chala safe and fast compare to `JdbcCursorItemReader` which is not thread-safe. Database load drushtya kooda paging chala cases lo better.

## Interview Questions
1. **Spring Batch lo FlatFileItemReader thread-safe aa kaada?**
   **Ans:** Kaadu. Endukante adi file loni line pointers (state) ni maintain chestundi. Multi-threaded step lo deenni as it is vaadithe reading corrupted ayye risk undi.

2. **Oka batch job output ni oke saari DB ki and File ki ela rayali?**
   **Ans:** `CompositeItemWriter` ni vaadi, daantlo `JdbcBatchItemWriter` and `FlatFileItemWriter` ni list laaga inject chesthe, spring batch automatic ga item ni rendu writers ki sequentially pampistundi.

## Enhanced List
Official list prakaram `JpaPagingItemReader`, `HibernateCursorItemReader`, `MongoItemReader`, `KafkaItemReader` inka chala custom implementations Spring Batch natively support chesthundi. Veetini vadi manam boilerplate code thagginchachu.
