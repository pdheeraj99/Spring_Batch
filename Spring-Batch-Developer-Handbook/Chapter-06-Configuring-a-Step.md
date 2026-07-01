# Chapter 06 - Configuring a Step

## Introduction
Mundu chapters lo `Step` anedi oka job lo independent, sequential phase ani nerchukunnnamu. Ee chapter lo `Step` ni elaa configure cheyyali, chunk-oriented processing ante enti, tasklets, inka step flow (next, conditions) elaa control cheyyali anedi deep ga chusthamu.

## 1. Chunk-oriented Processing

Spring Batch lo ekkuva vadedi "chunk-oriented" processing style. Deeni artham, data ni okkokkati (one at a time) read chesi, oka 'chunk' (list of items) ga kalipi, okesari transaction boundary lona write cheyyadam.

### Behind the Scenes: Chunk Execution Flow
1. Transaction start avuthundi.
2. `ItemReader.read()` call avuthundi. Idi null oche varaku leda `commit-interval` reach ayye varaku items ni list lo add chesthundi.
3. Oka vela `ItemProcessor` unte, aa list lo unna prathi item meeda `process()` method call avuthundi. (Ikkada invalid items ni null return chesi filter cheyochu).
4. Aa processed list ni `ItemWriter.write(items)` ki pass chesthundi.
5. Transaction commit avuthundi. Emaina exceptions vasthe mottam chunk rollback avuthundi.

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

### Configuring Skip and Retry Logic
Chala sarlu konni bad records valla poorthi chunk / step fail avvakudadu. Alanti time lo **Skip Logic** vadatharu.
Atlane konni "retryable" errors (e.g. `DeadlockLoserDataAccessException`) vasthe, wait chesi malli try cheste pass avvachu. Alanti time lo **Retry Logic** vadatharu.

```java
// Java configuration for skip and retry
SkipPolicy skipPolicy = new LimitCheckingExceptionHierarchySkipPolicy(Set.of(FlatFileParseException.class), 10);
RetryPolicy retryPolicy = RetryPolicy.builder().maxRetries(3).includes(Set.of(DeadlockLoserDataAccessException.class)).build();

return new StepBuilder("step1", jobRepository)
        .<String, String>chunk(10, transactionManager)
        .reader(reader()).writer(writer())
        .faultTolerant() // Enables skip/retry
        .skipPolicy(skipPolicy)
        .retryPolicy(retryPolicy)
        .build();
```
*API Insight:* Skip and Retry logic `faultTolerant()` enable chesinapude pani chestundi.

## 2. TaskletStep
Chunk-oriented processing okkate kadu, konnisarlu stored procedure call cheyadam, file lu delete cheyadam lanti panulu cheyali. Deeniki `ItemReader/Writer` vadadam kante `Tasklet` vadadam best.

**Behind the Scenes:** `TaskletStep` anedi `Tasklet` interface loni `execute()` method ni repeatedly call chesthundi. Adi `RepeatStatus.FINISHED` return chese varaku leda exception vache varaku run avuthundi. Idi okka transaction lo jarugutundi.

```java
@Bean
public Step deleteFilesStep(JobRepository jobRepository, PlatformTransactionManager txManager) {
    return new StepBuilder("deleteFiles", jobRepository)
                .tasklet(myFileDeletingTasklet(), txManager)
                .build();
}
```

## 3. Intercepting Step Execution (Listeners)
Step execution lo various phases lo (e.g. chunk start ayye mundu, file read ayyina taruvatha) log cheyyadaniki leda custom logic execute cheyadaniki `StepListener` vadatharu.
Mukhya maina listeners:
- `StepExecutionListener` (`@BeforeStep`, `@AfterStep`): Step start, end ki.
- `ChunkListener` (`@BeforeChunk`, `@AfterChunk`): Prathi transaction (chunk) start, end ki.
- `ItemReadListener`, `ItemProcessListener`, `ItemWriteListener`: Prathi item chadivetappudu, process/write ayyetappudu.
- `SkipListener`: Error vachi item skip ayinappudu alert ivvadaniki.

## 4. Controlling Step Flow
Oka job lo steps varusaga run avvalani ledu. Oka step success aithe inkoka step ki, fail aithe inko step ki vellela "Conditional Flow" rayochu.

### Behind the Scenes: BatchStatus vs ExitStatus
Flow control eppudu `ExitStatus` meeda aadharapadi untundi, `BatchStatus` meeda kaadu. `BatchStatus` (STARTED, COMPLETED, FAILED) framework use chestundi. `ExitStatus` (e.g. "COMPLETED WITH SKIPS") anedi caller ki pampinche custom string. Listener dwara manam `ExitStatus` ni override cheyochu.

**Sequential Flow:**
```java
return new JobBuilder("job", jobRepository)
        .start(stepA).next(stepB).next(stepC).build();
```

**Conditional Flow:**
```java
return new JobBuilder("job", jobRepository)
        .start(stepA)
        .on("FAILED").to(stepC)   // if stepA fails, go to stepC
        .from(stepA).on("*").to(stepB) // otherwise go to stepB
        .end().build();
```

**Programmatic Flow Decisions:**
Conditions mari complex ga unte `JobExecutionDecider` vadachu. Idi `FlowExecutionStatus` ni return chestundi, daani batti next etu vellalo determine cheyochu.

## 5. Late Binding of Job and Step Attributes
Chala sarlu file names compiler time lo kaakunda, run-time lo Job parameters dwara vasthayi. Alanti attributes ni inject cheyadaniki Spring Batch `Late Binding` support isthundi.

**Behind the Scenes: Scope**
Late binding jaragalante bean ki `@StepScope` leda `@JobScope` thappakunda undali. Ee scopes valla Spring context aa bean ni application start ayyinapudu kaakunda, Step leda Job start ayinappudu initialize chesthundi. Appude `JobParameters` available ga untayi.

```java
@Bean
@StepScope
public FlatFileItemReader<Foo> reader(@Value("#{jobParameters['input.file.name']}") String name) {
    return new FlatFileItemReaderBuilder<Foo>()
                .resource(new FileSystemResource(name))
                //...
                .build();
}
```

## Interview Questions
1. **Chunk-oriented processing ante enti? Adi ela pani chestundi?**
   - Reader dwara items ni one-by-one chadivi, oka chunk ga list lo petti, commit-interval reach ayyaka motham chunk ni oke transaction lo writer ki pampisthundi.
2. **TaskletStep ki, Chunk-oriented step ki theda enti?**
   - File reads/writes lanti bulk data processing ki Chunk vadatharu. Script execution, DB cleanup, file zipping lanti one-off tasks ki Tasklet vadatharu.
3. **`@StepScope` enduku vadatharu?**
   - Runtime lo pass chese JobParameters leda ExecutionContext attributes ni bean loki inject cheyadaniki late binding kosam vadatharu.

## Summary
Ee chapter lo chunk processing, skip/retry logic elaa configure cheyali, tasklets epudu vadali, step flow ni conditions inka deciders vadi elaa control cheyyali ani detail ga chusamu. Alage late binding (StepScope) yokka avasaram inka importance nerchukunnnamu. Next chapter lo main workers ayina ItemReaders and ItemWriters gurinchi deep ga discuss cheddamu.