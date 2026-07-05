# Chapter 12 - Unit Testing

## Introduction
Spring Batch jobs ni unit test cheyadam kante, "end-to-end" integration test cheyadam chala mukhyam. Endukante transaction boundaries, chunks, item processor logic evi normal method call laaga check cheyyalem. Deenikosam Spring Batch `spring-batch-test` project isthundi.

**Note:** Spring Batch 6.0 nunchi `JobLauncherTestUtils` badulu `JobOperatorTestUtils` vaduthunnaru and JUnit 4 support theesesaru. JUnit Jupiter (JUnit 5) is mandatory.

## 1. Test Setup

Batch tests ni run cheyadaniki, mana test class ki `@SpringBatchTest` annotation ivvali. Idi mana kosam `JobOperatorTestUtils` mariyu `JobRepositoryTestUtils` beans ni inject chestundi.

Alage application context load cheyadaniki `@SpringJUnitConfig` (JUnit 5) vadathamu.

```java
@SpringBatchTest
@SpringJUnitConfig(MyJobConfig.class)
public class MyJobFunctionalTests {

    @Autowired
    private JobOperatorTestUtils jobOperatorTestUtils;

    @Autowired
    private JobRepositoryTestUtils jobRepositoryTestUtils;

    @BeforeEach
    public void setup(@Autowired Job job) {
        // Okavela multiple jobs unte setJob vadali,
        // single job unte automatically wire avuthundi.
        this.jobOperatorTestUtils.setJob(job);
    }

    // ... tests ...
}
```

## 2. End-to-End Testing (Testing the Entire Job)
Motham job ni test cheyadaniki `JobOperatorTestUtils.startJob()` vadatharu. Idi `JobExecution` ni return chestundi. Ah tharvatha manam BatchStatus `COMPLETED` vachinda leda ani check cheyochu.

```java
@Test
public void testJob() throws Exception {
    // Execute job
    JobExecution jobExecution = jobOperatorTestUtils.startJob();

    // Assert status
    assertEquals("COMPLETED", jobExecution.getExitStatus().getExitCode());

    // Optional: Assert DB state after job run
}
```
Job parameters kavali ante `startJob(JobParameters)` use cheyochu.

## 3. Testing Individual Steps
Chala pedda job unnapudu, okko step fail avvakunda test cheyyadam kastam. Anduke `JobOperatorTestUtils.startStep("stepName")` vadukuni oka specific step ni matrame test cheyochu.

```java
@Test
public void testStep1() {
    JobExecution jobExecution = jobOperatorTestUtils.startStep("loadFileStep");
    assertEquals(BatchStatus.COMPLETED, jobExecution.getStatus());
}
```

## 4. Step Scope Testing (StepScopeTestUtils)
Manam ItemReader leda ItemProcessor ki `@StepScope` petti, test chestunte, direct ga `reader.read()` pilisthe error vastundi (endukante scope ledu kabatti).
Alanti situations lo `StepScopeTestUtils` vaadutharu.

```java
@Test
public void testReaderWithStepScope() throws Exception {
    // 1. Create a mock execution
    StepExecution stepExecution = MetaDataInstanceFactory.createStepExecution();

    // 2. Put context into execution
    stepExecution.getExecutionContext().putString("input.file.name", "test-data.csv");

    // 3. Run the logic inside the scope
    int count = StepScopeTestUtils.doInStepScope(stepExecution, () -> {
        int items = 0;
        reader.open(stepExecution.getExecutionContext());
        while (reader.read() != null) {
            items++;
        }
        reader.close();
        return items;
    });

    assertEquals(5, count);
}
```

## 5. JobRepositoryTestUtils
Test run ayyaka prathisari patha data database lo unte, malli next test fail avvachu. Anduke `JobRepositoryTestUtils` loni `removeJobExecutions()` vadi database clean chesukovachu. Leda temporary in-memory database H2 vaadadam best practice.

## Summary
`@SpringBatchTest` and `JobOperatorTestUtils` vadatam valla end-to-end integration tests chala easy avuthayi. `@StepScope` components ni test cheyyadaniki `StepScopeTestUtils` upayogapaduthundi.
