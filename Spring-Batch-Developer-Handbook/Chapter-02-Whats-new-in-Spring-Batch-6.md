# Chapter 02 - What’s new in Spring Batch 6.0

## Introduction
Prati major release laaga, Spring Batch 6.0 kooda chala kotha features, dependencies upgrades, mariyu architecture level improvements tho vachindi. Ee chapter lo manam Spring Batch 6.0 lo vachina mukhya maina maarupulu enduku vachayi, vaati valla upayogam enti, inka vaatini ela vadalo chusthamu.

## 1. Dependencies Upgrade
Spring Batch 6.0 ee kindi latest Spring inka third-party libraries ki upgrade aindi:
- **Spring Framework 7.0**
- **Spring Integration 7.0**
- **Spring Data 4.0**
- Spring LDAP 4.0, Spring AMQP 4.0, Spring Kafka 4.0
- **Micrometer 1.16**

*Idi enduku mukhyam?* Latest performance improvements inka security updates kosam base framework versions ni upgrade cheyadam chaala avasaram.

## 2. Batch Infrastructure Configuration Improvements

### New annotations and classes
V6 kante mundu `@EnableBatchProcessing` annotation anedi eppudu JDBC-based infrastructure ki tie aiyyi undedi. Kani ipudu adi change aindi. Rendu kotha annotations ni introduce chesaru:
- `@EnableJdbcJobRepository`
- `@EnableMongoJobRepository`

Ipudu `@EnableBatchProcessing` ni common attributes configure cheyadaniki vaadi, store-specific attributes ki paina unna kotha annotations ni vadukochu.

**Code Example:**
```java
@EnableBatchProcessing(taskExecutorRef = "batchTaskExecutor")
@EnableJdbcJobRepository(dataSourceRef = "batchDataSource", transactionManagerRef = "batchTransactionManager")
class MyJobConfiguration {
    @Bean
    public Job job(JobRepository jobRepository) {
        return new JobBuilder("job", jobRepository)
            // job flow omitted
            .build();
    }
}
```

### Resourceless Batch Infrastructure by default
Mundu (prior to V6), `DefaultBatchConfiguration` metadata storage kosam compulsory ga in-memory database (like H2 or HSQLDB) ni require chesedi. Kani ipudu, default ga "resourceless" batch infrastructure (based on `ResourcelessJobRepository`) vaadutunnaru.

*Performance Note:* Deenivalla metadata vadani applications ki memory footprint taggutundi inka database connections/transactions undav kabatti default performance perugutundi.

### Configuration Simplification
Chala mandi beans ni configure chese pani taggicharu. For example:
- `JobRepository` ippudu `JobExplorer` interface ni extend chestundi, so separate ga `JobExplorer` bean avasaram ledu.
- `JobOperator` ippudu `JobLauncher` ni extend chestundi.
- `JobRegistry` ippudu optional inka auto-registration chestundi.
- Transaction manager optional, okavela ivvakapote `ResourcelessTransactionManager` use avtundi.

## 3. New Implementation of Chunk-Oriented Processing
Idi kotha feature kadu kani kotha implementation. `ChunkOrientedTasklet` / `TaskletStep` badulu ipudu `ChunkOrientedStep` vachindi (idi 5.1 lo experimental ga vachi 6.0 lo stable aindi).

Fault-tolerance features lo kooda maarupulu vachayi:
- **Retry:** Spring Retry library badulu, ippudu Spring Framework 7 lo unna retry functionality vadutunnaru.
- **Skip:** Skip feature ippudu poortiga `SkipPolicy` interface meeda matrame aadharapadi undi.

## 4. New Concurrency Model
Mundu unna "parallel iteration" model lo throttling inka backpressure valla performance issues inka confusing transaction semantics vachedi.

Ippudu, **Producer-Consumer pattern** tho kotha model vachindi. Oka internal bounded queue ni producer inka consumer threads madyalo pedutaru. Items ready avvagane queue lo petti, consumer threads vaatini process chestayi. Chunk write ayye time ki producer pause ayyi, write ayyaka malli produce chestundi. Idi chala efficient.

## 5. Graceful Shutdown Support
Spring Batch 6.0 lo Job executions ni controlled manner lo stop chese "Graceful Shutdown" vachindi. Ee feature interrupt signals ni active steps ki pampinchi, repository ni consistent state lo update chestundi (restartability kosam).

## 6. Observability with Java Flight Recorder (JFR)
Micrometer metrics ki toduga, JVM lo in-built ga unde Java Flight Recorder (JFR) support ni add chesaru. Dini valla minimal performance overhead tho batch job executions (read, write, transaction boundaries) profiling cheyochu.

## 7. Lambda Style Configuration
Builders ni vaade tapudu, contextual lambda expressions vadi configuration ni inka concise ga rayochu.

**Example:**
```java
// V6 nunchi ilaa rayochu
var reader = new FlatFileItemReaderBuilder()
 .resource(...)
 .delimited(config -> config.delimiter(',').quoteCharcter('"'))
 .build();
```

## Other Notable Features
- **Null Safety:** APIs ki `JSpecify` annotations add chesaru for better code quality.
- **Local Chunking:** JVM lopalane multiple threads vaadi local ga chunk items ni parallel ga process cheyadaniki support vachindi.
- **SEDA Style with Spring Integration:** Staged Event-Driven Architecture ni Spring Integration message channels vaadi scale cheyochu.
- **Jackson 3 Support:** Jackson 3.x ki upgrade ayyaru, Jackson 2.x ni deprecate chesaru.
- **Remote Step Support:** Oka batch job yokka steps ni remote machines leda clusters lo execute cheyadaniki Spring Integration messaging dwara support icharu.

## Deprecations
- JUnit 4 support in `spring-batch-test` deprecated.
- XML configuration (`batch:...` namespace) deprecated.

## Interview Questions
1. **Spring Batch 6.0 lo `@EnableBatchProcessing` elaa marindi?**
   - Idi varaku idi JDBC infrastructure ki tied ayyi undedi. Ippudu idi only common attributes isthundi. Store specific ki `@EnableJdbcJobRepository` leda `@EnableMongoJobRepository` vaadali.
2. **Spring Batch 6.0 lo Resourceless batch infrastructure ante enti?**
   - Default ga metadata kosam in-memory DB avasaram ledu. `ResourcelessJobRepository` vadutaru, deeni valla performance inka memory taggutundi.
3. **Kotha Concurrency model ela pani chestundi?**
   - Producer-Consumer pattern inka bounded queue vadi items ni process chestundi, idi prathi thread state sync kante chala efficient ga untundi.

## Summary
Spring Batch 6.0 anedi configuration ni chala simplify chesindi, unnecessary beans ni tagginchindi, kotha modern concurrent execution model ni thechindi, mariyu Observability (JFR) inka null safety (JSpecify) dwara enterprise readiness ni inka penchindi. Next chapters lo manam Architecture gurinchi deeply chusthamu.