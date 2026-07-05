# Chapter 06 - Configuring a Step

## Introduction
Mundu chapters lo `Step` anedi oka job lo independent, sequential phase ani nerchukunnnamu. Ee chapter lo `Step` ni elaa configure cheyyali, chunk-oriented processing ante enti, tasklets, inka step flow (next, conditions) elaa control cheyyali anedi deep ga chusthamu.

## 1. Chunk-oriented Processing

Spring Batch lo ekkuva vadedi "chunk-oriented" processing style. Deeni artham, data ni okkokkati (one at a time) read chesi, oka 'chunk' (list of items) ga kalipi, okesari transaction boundary lona write cheyyadam. `commit-interval` entha set cheste, anni items chadivina taruvatha (read), ItemProcessor dwara process ayyi, ItemWriter dwara okesari write avuthayi.

### Behind the Scenes: ChunkOrientedTasklet
Chunk-oriented step ane daaniki gunde kaaya ee `ChunkOrientedTasklet`.

*   **Package Name:** `org.springframework.batch.core.step.item.ChunkOrientedTasklet`
*   **Important Methods:** `execute(StepContribution contribution, ChunkContext chunkContext)`
*   **Who calls it internally:** `TaskletStep` (Lopala idi `Tasklet` laaga behave chestundi)
*   **What it calls next:** `ChunkProvider.provide()` (which calls `ItemReader.read()`) and `ChunkProcessor.process()` (which calls `ItemProcessor.process()` and `ItemWriter.write()`)
*   **Lifecycle:** Transaction lopala `execute()` call avagane idi `ItemReader` nunchi commit-interval varaku records thechukuntundi. Tharuvaatha aa list of records ni processing and writing kosam `ChunkProcessor` ki isthundi.

```mermaid
sequenceDiagram
    participant TaskletStep
    participant ChunkOrientedTasklet
    participant ChunkProvider
    participant ItemReader
    participant ChunkProcessor
    participant ItemProcessor
    participant ItemWriter

    TaskletStep->>ChunkOrientedTasklet: execute()
    ChunkOrientedTasklet->>ChunkProvider: provide()
    loop until commit-interval
        ChunkProvider->>ItemReader: read()
        ItemReader-->>ChunkProvider: returns Item
    end
    ChunkProvider-->>ChunkOrientedTasklet: returns Chunk (List)

    ChunkOrientedTasklet->>ChunkProcessor: process(Chunk)
    loop for each Item
        ChunkProcessor->>ItemProcessor: process(Item)
        ItemProcessor-->>ChunkProcessor: returns processed Item
    end
    ChunkProcessor->>ItemWriter: write(Processed Chunk)
    ChunkOrientedTasklet-->>TaskletStep: RepeatStatus
```

**Code Example:**
```java
@Bean
public Step step1(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
    return new StepBuilder("step1", jobRepository)
                .<String, String>chunk(10, transactionManager) // commit-interval is 10
                .reader(itemReader())
                .processor(itemProcessor())
                .writer(itemWriter())
                .build();
}
```

### 1.1 Configuring Skip and Retry Logic
Chala sarlu konni bad records valla poorthi chunk / step fail avvakudadu. Alanti time lo **Skip Logic** vadatharu.
Atlane konni "retryable" errors (e.g. `DeadlockLoserDataAccessException`) vasthe, wait chesi malli try cheste pass avvachu. Alanti time lo **Retry Logic** vadatharu. `faultTolerant()` configure chesinapude idi enable avtundi.

```java
@Bean
public Step skipAndRetryStep(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
    return new StepBuilder("skipAndRetryStep", jobRepository)
                .<String, String>chunk(10, transactionManager)
                .reader(itemReader())
                .writer(itemWriter())
                .faultTolerant() // Enables skip and retry
                .skipLimit(10) // Allow skipping up to 10 items
                .skip(FlatFileParseException.class) // Skip items if this exception occurs
                .retryLimit(3) // Retry up to 3 times
                .retry(DeadlockLoserDataAccessException.class) // Retry on deadlock
                .build();
}
```

### 1.2 Transaction Attributes
Step transaction attributes (isolation level, propagation behavior, timeout) kooda customize cheyochu.
```java
DefaultTransactionAttribute attribute = new DefaultTransactionAttribute();
attribute.setPropagationBehavior(Propagation.REQUIRES_NEW.value());
attribute.setIsolationLevel(Isolation.SERIALIZABLE.value());
attribute.setTimeout(30); // seconds

return new StepBuilder("step1", jobRepository)
            .<String, String>chunk(10, transactionManager)
            .reader(itemReader())
            .writer(itemWriter())
            .transactionAttribute(attribute)
            .build();
```

### 1.3 Registering ItemStreams
ItemStream anedi ItemReader leda ItemWriter yokka execution state (e.g. current line number) ni ExecutionContext lo save cheyadaniki vadatharu. Framework automatically `ItemStream` implement chese readers/writers ni register chestundi. Kani, oka ItemStream implement chese component ItemReader leda ItemWriter ki delegate aithe, daanni explicit ga register cheyyali.
```java
return new StepBuilder("step1", jobRepository)
            .<String, String>chunk(10, transactionManager)
            .reader(itemReader())
            .writer(itemWriter())
            .stream(fileItemWriter1())
            .stream(fileItemWriter2())
            .build();
```

---

## 2. Configuring Step Restart (Start Limit & allowStartIfComplete)

Job restart ayinappudu steps ela behave cheyyalo manam control cheyochu.
- `startLimit(int limit)`: Oka step ni enni sarlu start cheyochu ane limit. (Default: Integer.MAX_VALUE). Limit exceed aithe `StartLimitExceededException` vastundi.
- `allowStartIfComplete(boolean allow)`: Oka step already 'COMPLETED' status tho unna kooda, job restart ayinappudu malli run avvalante idi `true` set cheyyali. Validation steps ki idi chala upayogapadtundi.

```java
@Bean
public Step stepWithLimits(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
    return new StepBuilder("stepWithLimits", jobRepository)
                .<String, String>chunk(10, transactionManager)
                .reader(itemReader())
                .writer(itemWriter())
                .startLimit(3) // Maximum 3 times this step can be started
                .allowStartIfComplete(true) // Always run even if it was previously COMPLETED
                .build();
}
```

---

## 3. TaskletStep
Chunk-oriented processing okkate kadu, konnisarlu stored procedure call cheyadam, file lu delete cheyadam lanti panulu cheyali. Deeniki `ItemReader/Writer` vadadam kante `Tasklet` vadadam best.

### Behind the Scenes: TaskletStep
*   **Package Name:** `org.springframework.batch.core.step.tasklet.TaskletStep`
*   **Default Implementation:** This is the default implementation of `org.springframework.batch.core.Step` for both Tasklets and Chunks.
*   **Important Methods:** `doExecute()`, `execute()`
*   **Who calls it internally:** `SimpleStepHandler.handleStep()` (Which is called by `SimpleJob`)
*   **What it calls next:** `TransactionManager` (to start transaction) and `Tasklet.execute()`
*   **Lifecycle:** Framework nunchi step execute call vachinappudu, idi Spring `TransactionManager` vaadi kotha transaction begin chestundi. Aa transaction lona `Tasklet.execute()` call chestundi. Exception vasthe rollback, success aithe commit chestundi.

```java
@Bean
public Step deleteFilesStep(JobRepository jobRepository, PlatformTransactionManager txManager) {
    return new StepBuilder("deleteFiles", jobRepository)
                .tasklet(myFileDeletingTasklet(), txManager)
                .build();
}
```


## 4. Inheriting from a Parent Step
Java config lo builders use chestunnapudu prathi sari oke options (listeners, skip limit, etc.) rasedani badulu, okesari master (parent) definition create chesukuni, daanni multiple steps (child steps) ki apply cheyochu. Spring Batch v5+ lo XML-style abstract step inheritance kante composition leda helper methods vadatam best practice. Kani, basic properties set chese common builder ni thayaaru cheyachu.


## 5. Intercepting Step Execution (Listeners)
Step execution lo various phases lo (e.g. chunk start ayye mundu, file read ayyina taruvatha) log cheyyadaniki leda custom logic execute cheyadaniki `StepListener` vadatharu.
Mukhya maina listeners:
- `StepExecutionListener` (`@BeforeStep`, `@AfterStep`): Step start, end ki.
- `ChunkListener` (`@BeforeChunk`, `@AfterChunk`): Prathi transaction (chunk) start, end ki.
- `ItemReadListener`, `ItemProcessListener`, `ItemWriteListener`.

```java
@Bean
public Step listenerStep(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
    return new StepBuilder("listenerStep", jobRepository)
                .<String, String>chunk(10, transactionManager)
                .reader(itemReader())
                .writer(itemWriter())
                .listener(new MyStepExecutionListener())
                .build();
}
```

## 6. Controlling Step Flow
Oka job lo steps varusaga run avvalani ledu. Oka step success aithe inkoka step ki, fail aithe inko step ki vellela "Conditional Flow" rayochu. Flow control kosam manam `ExitStatus` ni vadathamu.

### Behind the Scenes: BatchStatus vs ExitStatus
Flow control eppudu `ExitStatus` meeda aadharapadi untundi, `BatchStatus` meeda kaadu.
- **BatchStatus**: Framework internal status (e.g., STARTED, COMPLETED, FAILED, ABANDONED).
- **ExitStatus**: Step execution complete ayyaka job ki return iche string. (e.g., "COMPLETED WITH SKIPS"). Listener dwara manam `ExitStatus` ni override cheyochu.

**Conditional Flow Example:**
```java
@Bean
public Job conditionalJob(JobRepository jobRepository, Step stepA, Step stepB, Step stepC) {
    return new JobBuilder("conditionalJob", jobRepository)
            .start(stepA)
            .on("FAILED").to(stepC)   // if stepA fails, go to stepC
            .from(stepA).on("*").to(stepB) // otherwise go to stepB
            .end()
            .build();
}
```

## 7. Late Binding of Job and Step Attributes
Chala sarlu file names compiler time lo kaakunda, run-time lo Job parameters dwara vasthayi. Alanti attributes ni inject cheyadaniki Spring Batch `Late Binding` support isthundi.

**Behind the Scenes: Scope**
Late binding jaragalante bean ki `@StepScope` leda `@JobScope` thappakunda undali. Ee scopes valla Spring context aa bean ni application start ayyinapudu kaakunda, Step leda Job start ayinappudu initialize chesthundi. Appude `JobParameters` available ga untayi.

```java
@Bean
@StepScope
public FlatFileItemReader<Foo> reader(@Value("#{jobParameters['input.file.name']}") String name) {
    return new FlatFileItemReaderBuilder<Foo>()
                .resource(new FileSystemResource(name))
                .name("fooReader")
                .targetType(Foo.class)
                .delimited()
                .names("name", "age")
                .build();
}
```

## Interview Questions
1. **`TaskletStep` lona `ChunkOrientedTasklet` elaa fit avuthundi?**
   - Chunk processing lo unde reader, processor, writer logic antha `ChunkOrientedTasklet` lo wrap ayyi untundi. `TaskletStep` transaction theesukuni ee tasklet ni `execute()` ani pilustundi. So Chunk processing is just a special type of Tasklet.
2. **`@StepScope` enduku vadatharu?**
   - Runtime lo pass chese `JobParameters` leda `ExecutionContext` attributes ni bean loki inject cheyadaniki late binding kosam vadatharu. Application start ki badulu step start ayinappude aa bean create avtundi.
3. **`BatchStatus` mariyu `ExitStatus` ki difference enti?**
   - `BatchStatus` anedi Step execution yokka state (e.g., COMPLETED, FAILED) ni represent chestundi. Idi framework maintain chestundi. `ExitStatus` anedi Step execution ayyaka final result ga return iche custom string (e.g., "COMPLETED WITH SKIPS"). Conditional flow routing antha `ExitStatus` meedane aadharapadi untundi.
4. **`allowStartIfComplete` use case enti?**
   - Job restart ayinappudu already success ayina steps malli run avvavu. Kani okavela step (e.g., file existence validation leda temporary cleanup) prathi execution lo jaragalante `allowStartIfComplete(true)` vadathamu.

## Best Practices & Enterprise Notes
- **Transaction Manager**: Prathi Chunk transaction boundary lo run avuthundi, kabatti correct TransactionManager ni provide cheyyadam mukhyam.
- **Commit Interval Tuning**: Commit interval size chala peddaga pedithe Memory Issues (OOM) raavachu, chala chinnaga pedithe performance taggipothundi. Usually 100 to 1000 bounds lo untundi depending on data size.
- **Idempotency**: Restart logic use chestunnapudu steps idempotent ga undela chusukovali, ante malli run ayina system state corrupt avvakudadu.

## Summary
Ee chapter lo `TaskletStep` inka `ChunkOrientedTasklet` madyalo unna internal relationships chusamu. Sequence diagrams lo reader/processor/writer ela call avuthayo, transaction attributes ela work avuthayo, restart configuration (start limit) ela set cheyyalo, flow control lo `ExitStatus` patra, mariyu late binding gurinchi deep ga discuss cheddamu. Next chapter lo main workers ayina ItemReaders and ItemWriters gurinchi deep ga discuss cheddamu.
