# Chapter 13 - Common Batch Patterns

## Introduction
Spring Batch lo off-the-shelf components (like standard ItemReaders inka ItemWriters) tho ne chala panulu aipothayi. Kani enterprise batch systems lo konni recurring problems untayi - like error logging, manual job stopping, multi-line records parsing lanti vi. Ee chapter lo Spring Batch lo ivi ela handle cheyalo konni Common Batch Patterns gurinchi thelusukundam.

---

## 1. Logging Item Processing and Failures

Oka step lo bad record vachinapudu, adhi skip cheyyadame kakunda, daani gurinchi log lo (leda inko error table lo) detail ga rayali anukuntam. Daaniki `ItemReadListener` leda `ItemWriteListener` implement chesi Step ki register cheyali.

### Behind the Scenes: `ItemListenerSupport`
- **Package Name:** `org.springframework.batch.core.listener.ItemListenerSupport`
- **Important Methods:** `onReadError(Exception ex)`, `onWriteError(Exception ex, List items)`
- **Internal Caller Flow:**
```text
TaskletStep#execute()
  ↓
ChunkOrientedTasklet#execute()
  ↓
SimpleChunkProcessor#process() / doWrite()
  ↓ (If Exception occurs)
ItemWriteListener#onWriteError()
```

```java
@Bean
public Step simpleStep(JobRepository jobRepository, PlatformTransactionManager txManager) {
    return new StepBuilder("simpleStep", jobRepository)
            .<String, String>chunk(10, txManager)
            .reader(reader())
            .writer(writer())
            .listener(new ItemFailureLoggerListener()) // Custom logger
            .build();
}
```
*Best Practice Note:* `onError` lopala db lanti transactional resource vaduthunte, thappakunda propagation `REQUIRES_NEW` undali. Leda parent rollback valla idi kuda fail avtundi.

---

## 2. Stopping a Job Manually for Business Reasons

Business logic prakaram job ni aapeyyali (Example: Oka "Poison Pill" record vachindi, inka processing continue avvakudadu) anukunte.

**Approach 1: Throw Runtime Exception**
ItemProcessor nunchi oka custom `RuntimeException` (which is not configured to be skipped/retried) throw cheste, Job FAILED status tho end aipothundi.

**Approach 2: Return Null from Reader**
`ItemReader` nunchi `null` return cheste, adi Step ki data aipoindi ani cheppi gracefully COMPLETED state loki thestundi.

**Approach 3: Using StepExecution flag**
Meeru Listener dwara `StepExecution` object theeskuni `stepExecution.setTerminateOnly()` ani pilithe, next check lo framework `JobInterruptedException` throw chesi job ni kill chestundi.

---

## 3. Adding a Footer Record

Flat file ki data rasepudu, last lo "Total records=500" lanti footer record rayali anukunte, `FlatFileFooterCallback` vadathamu.

### Behind the Scenes: `FlatFileFooterCallback`
- **Package Name:** `org.springframework.batch.item.file.FlatFileFooterCallback`
- **Important Methods:** `writeFooter(Writer writer)`
- **Lifecycle:** `FlatFileItemWriter` loni `close()` method pilichinappudu (Step end aypoyetappudu), stream close chese mundu ee footer callback ni pilustundi.

```java
public void writeFooter(Writer writer) throws IOException {
    writer.write("Total Amount Processed: " + this.totalAmount);
}
```

*State Persistence Warning:* Ikkada `totalAmount` calculate chese time lo class stateful aipotundi. Restart scenario lo ee value pothundi. Anduke, ilanti writers ki thappakunda `ItemStream` interface implement chesi, `update()` method lona aa value ni `ExecutionContext` lo pettali.

---

## 4. Driving Query Based ItemReaders

Database nunchi chala pedda data theesku ravalante Cursor leda Paging readers vadathamu ani mundu chapter lo chusamu. Kani konni DB vendor issues valla (like aggressive locking) motham objects memory loki theravadam kastam ayyidi.

Appudu **"Driving Query Pattern"** vadatharu.
1. `ItemReader` lo chinna query vadi (e.g. `SELECT ID from CUSTOMER`) kevalam IDs (primary keys) matrame read chestaru.
2. `ItemProcessor` loki aa ID vastundi. Appudu processor inkoka explicit query leda DAO call (e.g. `SELECT * from CUSTOMER where ID = ?`) vadi full object thechukuni process chestundi.

---

## 5. Handling Step Completion When No Input is Found

Spring Batch lo oka input file lekapoina, leda empty unna, default ga `COMPLETED` exit code isthundi. Idi business prakaram error ayyundochu. Idi detect chesi fail cheyyadaniki `NoWorkFoundStepExecutionListener` vadatharu.

### API Insight
- **Package Name:** `org.springframework.batch.core.step.NoWorkFoundStepExecutionListener`
```java
public ExitStatus afterStep(StepExecution stepExecution) {
    if (stepExecution.getReadCount() == 0) {
        return ExitStatus.FAILED; // Stops the job execution indicating failure
    }
    return null;
}
```

---

## 6. Passing Data to Future Steps

Oka step lo kanukkunna data ni (e.g. Generated output file name) tharuvata velle step ki ivvali anukunte.

**Behind the Scenes:** Step `ExecutionContext` kevalam ah step varake untundi. Job `ExecutionContext` motham job cycle untundi. Step run ayye loop lo Job context ki rayakudadu. Anduke step lopala `stepContext.put()` chesi, Step aipoyaka daanni job context loki promote cheyali.

Deeniki `ExecutionContextPromotionListener` vadatharu.

```java
@Bean
public ExecutionContextPromotionListener promotionListener() {
    ExecutionContextPromotionListener listener = new ExecutionContextPromotionListener();
    listener.setKeys(new String[] {"someKey"}); // "someKey" will be pushed from Step to Job context
    return listener;
}
```

Next step (Step 2) lo ah key ni `@BeforeStep` lo thecchukovachu.
```java
@BeforeStep
public void retrieveData(StepExecution stepExecution) {
    this.someObject = stepExecution.getJobExecution().getExecutionContext().get("someKey");
}
```

## Summary
Ee chapter lo real-world enterprise batch jobs lo vache chinna chinna samasyalaku (like passing data between steps, terminating explicitly, handling empty inputs, maintaining stateful footer callbacks) elanti established patterns Spring Batch isthundo nerchukunnnamu. Next chapter lo `Spring Batch Integration` gurinchi thelusukundam.