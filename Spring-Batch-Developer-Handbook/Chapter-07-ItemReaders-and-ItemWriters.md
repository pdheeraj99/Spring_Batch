# Chapter 07 - ItemReaders and ItemWriters

## Introduction
Batch processing lo mukhya maina pani: pedda amount lo data ni chadavadam (read), daanni transform cheyadam leda calculate cheyadam (process), inka result ni ekkadaina rayadam (write). Spring Batch deenikosam moodu key interfaces ni istundi: `ItemReader`, `ItemProcessor`, mariyu `ItemWriter`. Ee chapter lo manam veeti gurinchi, inka veetiki avasaramaina `ItemStream` gurinchi deeply nerchukuntamu.

## 1. ItemReader
`ItemReader` anedi okko item ni input nunchi retrieval cheyadaniki vaade abstraction. Idi database nunchi row kavachu, flat file nunchi line kavachu, leda XML nunchi oka element kavachu.

### API Insight
```java
public interface ItemReader<T> {
    T read() throws Exception;
}
```
**Behind the Scenes:**
`read()` method okko item ni return chestundi. Input lo inka items emi lekapothe, idi **`null`** return chestundi. Idi chusake Spring Batch "Oh, inka data aipoindi" ani commit phase ki vellipotundi.
- `ItemReader` by default "forward only". Ante okasari chadivina record malli chadavadu. (Kani JMS queue lanti transactional resources lo rollback ayinapudu malli same item raavachu).
- Input lo asalu items lekapothe (e.g. empty database table), first `read()` call lone `null` vastundi. Exception emi raadu.

## 2. ItemWriter
`ItemWriter` anedi `ItemReader` ki inverse. Input resource ni open chesi chadivinatte, output resource ni open chesi data rayali. Ikkada difference enti ante, `ItemReader` okko item read chesthe, `ItemWriter` okesari oka **batch leda chunk** of items ni write chestundi.

### API Insight
```java
public interface ItemWriter<T> {
    void write(Chunk<? extends T> items) throws Exception;
}
```
**Behind the Scenes:**
`write()` method ki `Chunk` object (list of items) vastundi. Ee items anni `commit-interval` ki taggattu ga `ItemReader` chadivi, `ItemProcessor` process chesina tarwata vasthayi.
- Oka vela `ItemProcessor` lo konni items filter (null return chesthe) ayipothe, leda skip logic dwara items skip ayipothe, e `Chunk` empty ga kuda raavachu. `ItemWriter` empty chunks ni kuda gracefully handle cheyali.
- DB ki write chese tappudu, anni items okesari insert chesi, last lo hibernate session unte flush cheyadam ikkade jarugutundi.

## 3. ItemStream
Reader leda writer file ni leda database connection ni open cheyyali inka close cheyyali. Alage restart kosam state ni save cheyali. Deenikosame `ItemStream` interface vachindi.

### API Insight
```java
public interface ItemStream {
    void open(ExecutionContext executionContext) throws ItemStreamException;
    void update(ExecutionContext executionContext) throws ItemStreamException;
    void close() throws ItemStreamException;
}
```
**Behind the Scenes Execution Flow:**
1. **`open()`**: Step start ayinapudu call avtundi. Ikkada file streams open avthayi. Oka vela job restart avthunte, `ExecutionContext` lo unna patha state (e.g. `lines.read.count=400`) chusi, file tondaraga aa line varaku forward avtundi.
2. **`update()`**: Prathi chunk commit ayye **mundu** call avtundi. Ikkade `ItemReader` "nenu 500 lines chadivesanu" ani `ExecutionContext` lo count update chestundi, so that database lo ah state persist avtundi.
3. **`close()`**: Step end ayinapudu call avtundi. Resources anni safe ga release avthayi.

## 4. The Delegate Pattern and Registering with the Step
Spring Batch lo chala sarlu okate component inkoka component ki panulu delegate chestundi (e.g. `CompositeItemWriter` multiple writers ki delegate chestundi).

**Common Mistake:**
Manam `ItemReader` leda `ItemWriter` ni `StepBuilder` lo pass chesinapudu, vatiki `ItemStream` (leda `StepListener`) implement chesi unte, Spring Batch vaatini **automatically register** chestundi.
Kani `CompositeItemWriter` lanti vi vadutunappudu, lopaliki pass chese sub-writers ni Spring Batch choodaledu!

**Solution:**
Veetini manually register cheyali `stream()` method vadukuni.

```java
@Bean
public Step step1(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
    return new StepBuilder("step1", jobRepository)
                .<String, String>chunk(2, transactionManager)
                .reader(fooReader())
                .writer(compositeItemWriter())
                // Manual registration of delegate!
                .stream(barWriter())
                .build();
}

@Bean
public CustomCompositeItemWriter compositeItemWriter() {
    CustomCompositeItemWriter writer = new CustomCompositeItemWriter();
    writer.setDelegate(barWriter());
    return writer;
}
```

## Interview Questions
1. **`ItemReader` null return cheste emavutundi?**
   - Spring Batch ki items anni poorthi ayyayi ani ardhamaipothundi. Ventane aa chunk ni process/write chesi, step ni complete chestundi. Exception emitlu raadu.
2. **`ItemWriter` okkosari empty list enduku receive chesukuntundi?**
   - Endukante chunk loni items anni `ItemProcessor` lono leda skip logic dwara filter/skip aipoyi undochu. Writer deenni error ga chudakunda gracefully handle cheyali.
3. **`ItemStream` interface yokka mukhya uddesham enti?**
   - Execution resources (files, DB connections) ni open/close cheyadaniki, inka commit ki mundu `ExecutionContext` lo state ni save cheyadaniki, deeni vallane fail aina job malli ekkadnunchi apindo akkada nunchi start (restart) avvagaludu.
4. **Delegate readers/writers vadinappudu elanti care theesukovali?**
   - Parent writer ne framework chustundi kabatti, sub-writers/delegates ni Step builder lo explicitly `.stream(subWriter)` vadi register cheyali. Leda valla `open()` / `update()` methods call avvavu, state fail avtundi.

## Summary
`ItemReader` and `ItemWriter` lu Spring Batch lo base components, ivi chadavadaniki, rayadaniki vaadatharu. Veetitho patu `ItemStream` kalavadam valla framework ki resource management inka restartability thelisosthundi. Delegations vadinappudu jagrattaga stream registration cheyyadam chala mukhyam. Next chapter lo specific ga Flat files, XML, inka Database Readers/Writers gurinchi deep ga velludam.