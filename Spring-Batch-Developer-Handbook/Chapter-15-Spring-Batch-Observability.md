# Chapter 15 - Spring Batch Observability

## Introduction
Modern cloud-native applications lo, application em chesthondi ani bayata nunchi monitor cheyadaniki Observability vadatharu. Spring Batch lona pedda jobs ekkada time theeskuntunnai, em jarugutundi anedi thelusukovadam mukhyam.

Spring Batch mukhya ga rendu observability options isthundi:
1. **Micrometer** (Metrics and Tracing for Spring Boot applications)
2. **JFR** (Java Flight Recorder for deep JVM-level profiling)

## 1. Micrometer Support
Micrometer anedi Prometheus leda Datadog lanti systems ki metrics ni pamputhundi. Idi by default disable ayi untundi. Deenni enable cheyadaniki `ObservationRegistry` bean create cheyali.

```java
@Bean
public ObservationRegistry observationRegistry(MeterRegistry meterRegistry) {
    ObservationRegistry observationRegistry = ObservationRegistry.create();
    observationRegistry.observationConfig()
        .observationHandler(new DefaultMeterObservationHandler(meterRegistry));
    return observationRegistry;
}
```

### Built-in Metrics
Spring Batch specific metrics anni `spring.batch` prefix tho untayi. Mukhya maina metrics:
- `spring.batch.job` (TIMER): Job execution entha sepu pattindi. Tags: `name`, `status`.
- `spring.batch.job.active` (LONG_TASK_TIMER): Present active unna job.
- `spring.batch.step` (TIMER): Step execution duration.
- `spring.batch.step.active` (LONG_TASK_TIMER): Present active unna step.
- `spring.batch.item.read` (TIMER): Item read duration.
- `spring.batch.item.process` (TIMER): Item process duration.
- `spring.batch.chunk.write` (TIMER): Chunk write duration.
- `spring.batch.job.launch.count` (COUNTER): Enni sarlu job launch ayyindi.

Ivi dashboards lo visualise cheskovachu.

## 2. JFR (Java Flight Recorder) Support
Spring Batch v6.0 nunchi JFR support vachindi. JFR anedi JVM lopala vunde event-based profiling tool, deenitho chala low overhead (performance impact lekunda) deep insights theskovachu.
Job start ainapudu kindhi argument pass cheste, JVM exit ayinapudu oka `.jfr` file create avthundi. Daanni manam *Java Mission Control (JMC)* lanti tools tho open chesi analyze cheyyachu.

```bash
java -XX:StartFlightRecording:filename=my-batch-job.jfr,dumponexit=true -jar my-batch-job.jar
```

Ila cheste Spring Batch automatically jobs, steps, item reads, writes, transaction boundaries ki JFR events generate chesthundi.

## Summary
Spring Batch ni Prometheus lanti modern monitoring systems tho connect cheyyadaniki `Micrometer` metrics vaadatharu. JVM level lo low-level performance issues (like memory leaks, lock contentions) thelusukodaniki Spring Batch v6.0 nunchi vachina JFR support vadatharu.
