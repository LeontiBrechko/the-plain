---
title: Use .join()/.get() Judiciously in Java Async Chains
updated: 2022-03-07 17:50
---

# Problem
You have a `CompletableFuture` chain (let’s call it the **main chain**) with an exception handling stage. In this handling stage, you would like to make another async call (e.g. free up a dependency resource) and do some actions after it (e.g. throw the original exception). That second call and actions should block the completion of the main chain

Example pseudo-code:
```
makeFirstAsyncRequest()
  .handleException(exception -> 
    makeSecondAsyncRequest()
    
    wait for makeSecondAsyncRequest to complete

    throw exception
  )
```

# **Solution**
## Java 8-11
In case of older Java versions (8-11), there are at least 2 possible solutions:

- Call [.join()](https://docs.oracle.com/en/java/javase/12/docs/api/java.base/java/util/concurrent/CompletableFuture.html#join()) or [.get()](https://docs.oracle.com/en/java/javase/12/docs/api/java.base/java/util/concurrent/CompletableFuture.html#get()) methods of `CompletableFuture` interface to block on completion of `makeSecondAsyncRequest` method call
- Create a separate `CompletableFuture` to complete the main chain after all actions are done in the `handleException` stage

### Call .join()/.get()
This is a **sub-optimal** (and may be even **breaking**) solution as [.join()](https://docs.oracle.com/en/java/javase/12/docs/api/java.base/java/util/concurrent/CompletableFuture.html#join()) or [.get()](https://docs.oracle.com/en/java/javase/12/docs/api/java.base/java/util/concurrent/CompletableFuture.html#get()) method could cause a thread used for the first async request to sleep and wait until the completion of the second request. This will block the first thread from doing work on other concurrent computations, resulting in performance degradation. Moreover, if the same thread is chosen to complete the second async call, this will produce a deadlock.

Here is a Java code example:
```java
private static CompletableFuture<Void> runWithJoin() {
  return makeFirstAsyncRequest()
      .exceptionally(exception -> {
        makeSecondAsyncRequest().join();
        throw new CompletionException(exception);
      });
}
```

### Create a Separate CompletableFuture
Use a separate `CompletableFuture`, which will complete once the second async call finishes. This approach will not block the execution of the first thread at the level of the `exceptionally` method definition.

Example:
```java
private static CompletableFuture<Void> runWithSeparateFuture() {
  final CompletableFuture<Void> resultFuture = new CompletableFuture<>();

  makeFirstAsyncRequest()
      .exceptionally(exception -> {
        makeSecondAsyncRequest()
            .whenComplete((voidResult, innerException) -> {
              if (innerException != null) resultFuture.completeExceptionally(innerException);
              else resultFuture.completeExceptionally(exception);
            });

        return null;
      });

  return resultFuture;
}
```

## Java 12+
For Java 12+, it’s easy and elegant by using [exceptionallyCompose()](https://docs.oracle.com/en/java/javase/12/docs/api/java.base/java/util/concurrent/CompletionStage.html#exceptionallyCompose(java.util.function.Function)) method of `CompletionStage` interface.

Example:
```java
private static CompletableFuture<Void> runWithExceptionallyCompose() {
  return makeFirstAsyncRequest()
      .exceptionallyCompose(exception -> makeSecondAsyncRequest()
          .thenAccept(voidResult -> {
            throw new CompletionException(exception);
          }));
}
```

# Solution Performance Comparison
## Test Setup
In order to compare the performance of the solutions, I used [AWS StepFunctions](https://aws.amazon.com/step-functions/) Java SDK with an invalid endpoint to simulate delayed (due to connection timeout) network requests and [Netty HTTP Client](https://netty.io/) to delegate the handling of concurrent requests to a non-blocking event-driven framework.

A single solution test consists of **20 runs** with **1000 main chain executions** each (2 async invocations per chain execution) and measures basic statistics of execution time.

To demonstrate the deadlock scenario, I run the tests using **a single SDK client** shared between the first and second async calls. Also, to compare actual execution time in a happy case, I run a set of tests with **separate SDK clients**.

The tests code could be found [here](https://github.com/LeontiBrechko/join-vs-future/blob/master/src/main/java/net/leontibrechko/blog/test/joinvsfuture/JoinVsFutureRunner.java). You can check environment configurations in [this gist](https://gist.github.com/LeontiBrechko/7b3ca5d72e674627839e30fa6b823fd0).

## Test Results
Based on the results presented in the tables below, usage of `.join()`/`.get()` solution produced a deadlock when a single SDK client is shared between `makeFirstAsyncRequest()` and `makeSecondAsyncRequest()`. This is explained by the fact that the same thread could be allocated to run both async invocations.

For the happy case, `.join()`/`.get()` is much slower than both separate CompletableFuture and `exceptionallyCompose()` solutions. Furthermore, both later solutions are comparable the same in performance.

### Single SDK Client

|  | .join()/.get() | Separate CompletableFuture | exceptionallyCompose() |
| --- |:---:|:---:|:---:|
| Total run time (ms) | Infinite due to deadlock | 2125 | 2111 |
| Average (ms) | Infinite due to deadlock | 106 | 105 |
| Max (ms) | Infinite due to deadlock | 289 | 300 |
| Min (ms) | Infinite due to deadlock | 77 | 73 |

### Separate SDK Clients

|  | .join()/.get() | Separate CompletableFuture | exceptionallyCompose() |
| --- |:---:|:---:|:---:|
| Total run time (ms) | 33182 | 2399 | 2076 |
| Average (ms) | 1659 | 119 | 103 |
| Max (ms) | 3257 | 433 | 294 |
| Min (ms) | 1524 | 76 | 80 |