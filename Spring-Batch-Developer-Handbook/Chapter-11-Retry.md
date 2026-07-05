# Chapter 11 - Retry

## Introduction
Network issues leda database deadlocks vachinappudu ah particular operation ventane fail avvakunda malli oka rendu sarlu try cheyadam chala common requirement. Spring Batch lo idi retry policies dwara chesthamu.

**Important Note (Spring Batch 6.0+):** Spring Batch v6.0 nunchi, `Spring Retry` project paina unna dependency ni theesesaru. Ipudu idi direct ga Spring Framework 7.0 loni **core retry feature** ni vaduthondi.

## 1. RetryOperations and RetryTemplate
Oka pani fail ayite daanni malli cheyadaniki `RetryOperations` interface vaadatharu. Deeniki default implementation `RetryTemplate`.

### API Insight
```java
public interface RetryOperations {
    <T, E extends Throwable> T execute(RetryCallback<T, E> retryCallback) throws E;
}
```

**Behind the Scenes: RetryTemplate**
- Prathi try kosam `RetryCallback` ni call chesthundi. Exception vasthe, `RetryPolicy` ni aduguthundi (malli try cheyacha ani), inka `BackOffPolicy` ni aduguthundi (entha sepu aagi try cheyali ani).

## 2. RetryPolicy
RetryPolicy anedi mukhya maina rule engine. Idi `true` return cheste `RetryTemplate` malli execute chestundi.
- **SimpleRetryPolicy**: Fixed number of attempts (e.g. 3 attempts) kosam vadataru. Specific exceptions ni mathrame retry chesela (e.g., `DeadlockLoserDataAccessException`) configure cheyochu.
- **TimeoutRetryPolicy**: Oka limit time (timeout) tharvatha inka try cheyadu.
- **AlwaysRetryPolicy**: Infinite ga try chesthune untundi (idi jagrathaga vadali).

## 3. BackOffPolicy
Malli try chese mundu entha sepu aagali anedi `BackOffPolicy` chuskuntundi. Ventane try cheste malli fail ayye chance untundi, anduke `Thread.sleep` laaga konchem gap ivvadam better.
- **FixedBackOffPolicy**: Prathi attempt ki fixed time aaguthundi (e.g., 1000ms).
- **ExponentialBackOffPolicy**: Time penchukuntu velladam (e.g., first 1 sec, tharuvatha 2 secs, aa tharuvatha 4 secs). Idi chala enterprise systems lo best practice.

## 4. RetryPolicy Builder
Spring Batch / Spring Core lo retry rules ni configure cheyadaniki Builders vadatam common.

```java
RetryTemplate template = RetryTemplate.builder()
        .maxAttempts(3)
        .fixedBackoff(1000) // wait for 1 second before retrying
        .retryOn(DeadlockLoserDataAccessException.class)
        .build();

template.execute(context -> {
    // Business logic that might fail
    System.out.println("Trying to access DB...");
    // throw new DeadlockLoserDataAccessException(...);
    return null;
});
```

## 5. Stateful vs Stateless Retry
- **Stateless Retry:** `RetryTemplate` lopale while loop untundi. Loop jaruguthunna sepu thread block avuthundi. (Idi most common).
- **Stateful Retry:** Asynchronous messaging vishayam lo thread ni block cheyyaleru. Exception vachinapudu ventane return aipotharu, malli message vachinapudu idhe patha attempt ani gurtu pattadaniki "state" maintain chestharu.

## Best Practices
- Non-idempotent operations ki retry pettakudadu. E.g. email send cheseppudu fail ayithe, retry chesthe 2-3 sarlu email vellachu.
- Retry eppudu external systems integration leda concurrent DB updates ki limit cheyyali.
- `ExponentialBackOffPolicy` eppudu better. Endukante downstream system over-load kakunda kapadtundi.

## Summary
`RetryTemplate` anedi transient errors ni handle cheyadaniki oka super utility. Step fault tolerance lo retry enable chesinappudu Spring Batch deenne vaduthundi.
