# Chapter 14 - Spring Batch Integration

## Introduction
Spring Batch ekkuva data ni batch mode lo process cheyyadaniki design cheyabadindi. Kani real-world lo data anedi asynchronous ga external systems (like queues, files, APIs) nunchi ravachu. Ee data streams ni batch jobs tho link cheyadaniki `Spring Integration` framework ni vadaniki Spring Batch Integration module upayogapaduthundi.

## 1. Launching Jobs through Messages (Job-Launching Gateway)
File ochi directory lo padagane leda message broker (JMS/RabbitMQ) loki message ragane, automatically oka batch job start cheyali anukunte manam `JobLaunchingGateway` vaduthamu.
Spring Integration loni `Message` payload ga `JobLaunchRequest` ni theeskuni, adi job ni execute chesi result (`JobExecution`) ni inko channel ki pamputhundi.

### Available Attributes of the Job-Launching Gateway
`JobLaunchingGateway` ni configure chesetappudu konni mukhya maina attributes untayi:
- `id`: Spring bean identifier.
- `auto-startup`: Startup lo automatic ga start avvala leda (default: `true`).
- `request-channel`: Input channel ekkadanunchi message vasthundi.
- `reply-channel`: Job complete ayyaka `JobExecution` payload ni pampinche channel.
- `reply-timeout`: Reply channel ki pampetapudu timeout (milli seconds lo). Default `-1` (infinite wait).
- `job-launcher`: Optional. Custom `JobLauncher` unte pass cheyachu. Leda default bean `jobLauncher` vaduthundi.
- `order`: Multiple subscribers unnapudu invocation order.

**Code Example:**
```java
@Bean
@ServiceActivator(inputChannel = "jobRequestsChannel")
public JobLaunchingGateway sampleJobLaunchingGateway(JobLauncher jobLauncher) {
    JobLaunchingGateway gateway = new JobLaunchingGateway(jobLauncher);
    gateway.setOutputChannelName("jobRepliesChannel");
    return gateway;
}
```

## 2. Asynchronous Processors
Chala sarlu ItemProcessor lo mana logic REST API call leda DB call cheyalsi osthundi. Ivi slow ga jaruguthayi. Appudu antha chunk process ayyevaraku thread block aipothundi.
Deenni solve cheyadaniki `AsyncItemProcessor` mariyu `AsyncItemWriter` vadatharu. Idi prathi item processing ni inko thread ki isthundi (returning `Future<O>`), tarvatha writer ah futures resolve ayyaka write chestundi.

```java
@Bean
public AsyncItemProcessor<String, String> asyncProcessor(ItemProcessor<String, String> itemProcessor, TaskExecutor taskExecutor) {
    AsyncItemProcessor<String, String> asyncProcessor = new AsyncItemProcessor<>();
    asyncProcessor.setDelegate(itemProcessor);
    asyncProcessor.setTaskExecutor(taskExecutor);
    return asyncProcessor;
}

@Bean
public AsyncItemWriter<String> asyncWriter(ItemWriter<String> itemWriter) {
    AsyncItemWriter<String> asyncWriter = new AsyncItemWriter<>();
    asyncWriter.setDelegate(itemWriter);
    return asyncWriter;
}
```

## 3. Remote Chunking
Pedda files unnappudu (e.g. 10GB file), oke machine meeda process chesthe memory/cpu problem vastundi. **Remote Chunking** vadithe:
- **Master (Manager) Node:** File ni read chestundi. Cunks ni create chesi message queue ki pamputhundi. (No Processing, No Writing).
- **Worker Node:** Queue nunchi chunk (data) theeskuni, processor dwara transform chesi DB lo write chestundi. (Worker node lone writer execute avtundi).

Idi heavy processing unte chala use avtundi kani network latency problem undachu (endukante actual data antha wire meeda travel avvali).

## 4. Remote Partitioning
Remote chunking kante chala sarlu **Remote Partitioning** better. Ikkada master node actual data ni pampadu, data loni bounds (e.g., Row 1-1000 oka worker ki, Row 1001-2000 inko worker ki) pamputhundi.
- **Master Node:** Partitioning rules create chesi, metadata maatrame message broker dwara workers ki pamputhundi.
- **Worker Node:** Aa metadata theeskuni, own ga read, process, write chestundi. Network meeda just instructions velthayi, payload velladu!

## Summary
Spring Integration vaadi Batch jobs ni event-driven (file arrival) ga eppudu kavalsina appudu start cheyochu. Asynchronous Processing petti single machine lo scalability thechukovachu. Inka multiple machines lo scale avvali ante Remote Chunking leda Remote Partitioning architecture ni implement cheyochu.
