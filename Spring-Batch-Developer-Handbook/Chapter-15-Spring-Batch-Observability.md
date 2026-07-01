# Chapter 15: Spring Batch Observability

## Introduction
Modern cloud-native applications lo, systems ni monitor cheyadam, troubleshoot cheyadam chala important. Daanni manam **Observability** antamu. Spring Batch version 4.2 nunchi metrics collection ni support chestundi, and version 5.0 and 6.0 lo **Micrometer** and **Java Flight Recorder (JFR)** support tho observability ni inka enhance chesaru. Ee chapter lo batch jobs ni ela monitor cheyali and metrics ela capture cheyali ani discuss chesthamu.

## Why this feature exists
- **Performance Tuning**: Oka batch job entha sepu run avtundi? Ye step daggara time ekkuva teeskuntundi? Leda oka item read cheyadaniki entha time padtundi? Ilanti questions ki accurate answers ivvadaniki.
- **Monitoring & Alerts**: Production lo job fail aina leda anukunna time kante ekkuva run avthunna (SLA breach), Prometheus + Grafana lanti monitoring tools dwara alerts generate cheyadaniki.
- **Tracing**: Microservices architecture lo oka job multiple systems tho interact ainappudu, distributed tracing (Trace & Spans) dwara request flow ni track cheyadaniki.
- **Low-Overhead Profiling**: Production system meeda minimum impact tho deep JVM level profiling (JFR) chesi bottlenecks ni identify cheyadaniki.

---

## 1. Micrometer Support

Micrometer anedi Java ki vendor-neutral application metrics facade. SLF4J logging ki ela panichestundo, Micrometer metrics ki ala panichestundi.

### Enabling Metrics Collection
Default ga metrics collection disable ayyi untundi. Danni enable cheyadaniki, manam `ObservationRegistry` bean ni define chesi `DefaultMeterObservationHandler` ni add cheyali.

```java
import io.micrometer.observation.ObservationRegistry;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.observation.DefaultMeterObservationHandler;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ObservabilityConfig {

    @Bean
    public ObservationRegistry observationRegistry(MeterRegistry meterRegistry) {
        ObservationRegistry observationRegistry = ObservationRegistry.create();
        observationRegistry.observationConfig()
            .observationHandler(new DefaultMeterObservationHandler(meterRegistry));
        return observationRegistry;
    }
}
```

Idi configure chesina taravata, Spring Batch automatic ga `spring.batch.*` prefix tho metrics ni record chestundi.

### Built-in Metrics
Micrometer support dwara Spring Batch provide chese default metrics:

| Metric Name | Type | Description | Tags |
| :--- | :--- | :--- | :--- |
| `spring.batch.job` | TIMER | Duration of job execution | name, status |
| `spring.batch.job.active` | LONG_TASK_TIMER | Currently active job | name |
| `spring.batch.step` | TIMER | Duration of step execution | name, job.name, status |
| `spring.batch.step.active` | LONG_TASK_TIMER | Currently active step | name |
| `spring.batch.item.read` | TIMER | Duration of item reading | job.name, step.name, status |
| `spring.batch.item.process` | TIMER | Duration of item processing | job.name, step.name, status |
| `spring.batch.chunk.write` | TIMER | Duration of chunk writing | job.name, step.name, status |
| `spring.batch.job.launch.count`| COUNTER | Job launch count | N/A |

*Note:* Ikkada `status` tag values success/failure ni indicate chestayi. `Job` and `Step` timers ki exit status untundi. Reading/Processing/Writing ki `SUCCESS` or `FAILURE` status untundi.

### Custom Metrics in Custom Components
Mana sontha `Tasklet` or custom readers lo specific logic ni time cheyyali anukunte, Micrometer API ni direct ga vaadochu.

```java
import io.micrometer.observation.Observation;
import io.micrometer.observation.ObservationRegistry;
import org.springframework.batch.core.StepContribution;
import org.springframework.batch.core.scope.context.ChunkContext;
import org.springframework.batch.core.step.tasklet.Tasklet;
import org.springframework.batch.repeat.RepeatStatus;

public class MyTimedTasklet implements Tasklet {

    private ObservationRegistry observationRegistry;

    public MyTimedTasklet(ObservationRegistry observationRegistry) {
        this.observationRegistry = observationRegistry;
    }

    @Override
    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) {
        Observation observation = Observation.start("my.tasklet.step", this.observationRegistry);
        try (Observation.Scope scope = observation.openScope()) {
            // Business logic ikkada rastamu
            Thread.sleep(1000); // Simulate work
            return RepeatStatus.FINISHED;
        } catch (Exception e) {
            observation.error(e);
            throw new RuntimeException(e);
        } finally {
            observation.stop();
        }
    }
}
```

### Tracing
Spring Batch 5 nunchi Micrometer Observation API dwara Tracing support kooda vachindi. Oka Job run avthunnappudu oka Trace create avtundi, and dani loni prathi Step execution ki oka Span create avtundi.

Deenni enable cheyadaniki, `TracingAwareMeterObservationHandler` ni configure cheyyali:

```java
@Bean
public ObservationRegistry observationRegistry(MeterRegistry meterRegistry, Tracer tracer) {
    DefaultMeterObservationHandler observationHandler = new DefaultMeterObservationHandler(meterRegistry);
    ObservationRegistry observationRegistry = ObservationRegistry.create();
    observationRegistry.observationConfig()
            .observationHandler(new TracingAwareMeterObservationHandler<>(observationHandler, tracer));
    return observationRegistry;
}
```

---

## 2. Java Flight Recorder (JFR) Support

### What is JFR?
Java Flight Recorder (JFR) anedi JVM lo unna built-in, low-overhead event-based profiling tool. Production application performance ni monitor and troubleshoot cheyadaniki idi chala useful. Spring Batch 6.0 nunchi JFR support out-of-the-box vachindi.

### Enabling JFR for Batch Jobs
JVM arguments dwara manam JFR ni enable cheyochu. External configurations avasaram ledu.

```bash
java -XX:StartFlightRecording:filename=my-batch-job.jfr,dumponexit=true -jar my-batch-job.jar
```

JFR enable chesinappudu, Spring Batch automatic ga ee kindi vatiki JFR events create chestundi:
- Job Executions
- Step Executions
- Item Reads
- Item Writes
- Transaction boundaries

Idi complete ayyaka generated `.jfr` file ni manam **Java Mission Control (JMC)** lanti tools lo load chesi detailed ga analyze cheyochu.

---

## Best Practices
- **Do not reinvent the wheel**: Job metrics ni collect cheyadaniki custom `JobExecutionListener` and `StopWatch` vaadakandi. Micrometer already built-in metrics isthundi.
- **Tagging properly**: Micrometer tags ekkuva vadithe Prometheus/Datadog lo cardinalities perigipothayi (e.g. tag ga prathi run time dynamic id ivvakandi). Use static tags like job name or step name.
- **Use JFR in Production**: JFR overhead is usually < 1%. Kabatti production lo heavy batch jobs run ayyetappudu `StartFlightRecording` enable cheyadam valla deep CPU/Memory and lock profiling data dorukutundi.

---

## Common Mistakes
- **Missing @EnableBatchProcessing impact**: Spring Boot vaade vallu `@EnableBatchProcessing` leda `DefaultBatchConfiguration` vaadakapothe, observation registry automatically inject avvadu. Appudu maname `BatchObservabilityBeanPostProcessor` ni application context lo register cheyyali.

---

## Interview Questions

1. **Spring Batch lo execution metrics (time taken for a step) ela monitor chestharu?**
   **Ans:** Spring Batch 4.2+ (and mainly 5.0+) nunchi Micrometer support undi. Manam `ObservationRegistry` bean create chesthe, automatic ga `spring.batch.step`, `spring.batch.item.read` lanti timers register avthayi. Vatini Prometheus lanti registry ki export chesi Grafana lo monitor cheyochu.

2. **Spring Batch JFR Support (Spring Batch 6.0 feature) valla upayogam enti?**
   **Ans:** Java Flight Recorder (JFR) anedi JVM level low-overhead profiler. Spring Batch 6 nunchi job/step executions and item read/write time ki JFR events produce chestundi. JFR enable cheyadam valla external agents or code changes lekunda production lo memory, locks, and batch behavior ni Java Mission Control (JMC) dwara analyze cheyochu.

3. **Job and Step logging ki, Tracing ki theda enti?**
   **Ans:** Logging logs ni lines ga dump chestundi. Kani Tracing (using Micrometer Tracing API) anedi oka Job request ni unique `TraceId` tho wrap chestundi, and aa job loni prathi step ki `SpanId` isthundi. Deeni valla Zipkin or Jaeger lanti systems lo entha time ekkada spend ayyindho oka hierarchical tree laaga visualise cheyochu.
