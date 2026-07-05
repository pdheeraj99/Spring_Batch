# Chapter 10 - Repeat

## Introduction
Batch processing lo oka task ni multiple times execute cheyalsina avasaram vasthundi. Chunk processing lona idi internal ga jaruguthundi, kani Spring Batch deenni oka general abstraction laaga `RepeatOperations` interface dwara provide chestundi. Ee chapter lo `Repeat` ela panichestundi inka completion policies enti ani deep ga nerchukuntamu.

## 1. RepeatOperations and RepeatTemplate
Oka pani ni malli malli cheyadaniki Spring Batch lo vaade interface `RepeatOperations`. Deeniki mukhya maina method `iterate`. Deeni default implementation `RepeatTemplate`.

### API Insight
```java
public interface RepeatOperations {
    RepeatStatus iterate(RepeatCallback callback) throws RepeatException;
}
```

- **RepeatCallback**: Ikkade mana actual logic untundi. Idi `RepeatStatus` return chestundi (`CONTINUABLE` leda `FINISHED`). `CONTINUABLE` isthe malli execute avuthundi, `FINISHED` isthe aagipothundi.

**Behind the Scenes: RepeatTemplate**
- **Package Name:** `org.springframework.batch.repeat.support.RepeatTemplate`
- `RepeatTemplate` oka while loop laaga panachestundi. Prathi iteration taruvatha `CompletionPolicy` ni adigi aagipovala leda continue avvala ani decide avuthundi.

```java
RepeatTemplate template = new RepeatTemplate();
template.setCompletionPolicy(new SimpleCompletionPolicy(2));

template.iterate(new RepeatCallback() {
    public RepeatStatus doInIteration(RepeatContext context) {
        // Do stuff in batch...
        System.out.println("Executing repeating logic...");
        return RepeatStatus.CONTINUABLE;
    }
});
```

## 2. Completion Policies
Loop eppudu aagali ani decide chesedi `CompletionPolicy`.

1. **SimpleCompletionPolicy**: Oka fixed number of times run ayyaka aagipothundi (e.g. `commit-interval` kosam idi vadatharu).
2. **TimeoutTerminationPolicy**: Certain amount of time (timeout) daatina taruvatha aagipothundi. Idi long-running jobs eppudu stop avvalo decide cheyyadaniki upayogapadutundi.
3. Manam custom completion policy kuda rayochu (e.g., specific batch window daatithe stop chesela).

## 3. Exception Handling (ExceptionHandler)
`RepeatCallback` lo exception vaste em jaragali anedi `ExceptionHandler` chusukuntundi.
- `SimpleLimitExceptionHandler`: Konni exceptions ni ignore chesi, oka limit dataka re-throw chestundi. Idi `RepeatTemplate` lo configure cheyyochu.

## 4. Listeners
Repeat loop life cycle lo hooks add cheyadaniki `RepeatListener` vadatharu.
- `open`, `before`, `after`, `onError`, `close` methods untayi. Prathi iteration mundu/tarvatha logs veyyadaniki leda monitoring kosam vadachu.

## 5. Parallel Processing (TaskExecutorRepeatTemplate)
Standard `RepeatTemplate` oke thread (Synchronous) lo iterations run chestundi. Kani okosari manam parallel ga run cheyali anukunte `TaskExecutorRepeatTemplate` vadi, oka Spring `TaskExecutor` isthe, multiple threads lo loop execute avuthundi.

## 6. Declarative Iteration (AOP)
Oka normal Spring service method ni intercept chesi, dani paina retry leda repeat logic ni apply cheyadaniki Spring Batch `RepeatOperationsInterceptor` ni isthundi. Idi AOP (Aspect Oriented Programming) vaadi intercept chestundi.

## Summary
`RepeatOperations` anedi Spring Batch lona internal chunk loop ki gunde kaaya lantidi. `RepeatTemplate` mariyu `CompletionPolicy` kalisi `ItemReader` ni eppatidaka pilavali ani decide chesthayi.
