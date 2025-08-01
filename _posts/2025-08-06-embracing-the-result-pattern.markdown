---
layout: post
title: '🏆 Embracing the Result Pattern in .NET: A Modern Alternative to Exceptions 🏆'
date: 2025-08-06 21:52:30 +1000
categories: Software Development
tags: .NET, Result Pattern, Error Handling, Performance, Readability
---

{% include
  fancy_banner.html
    image_url="/assets/images/embracing-the-result-pattern/title.png"
    alt="Title Banner"
%}

> By Gary Butler | 10 min read

<br/>

## 🚩 TL;DR;

Effective error handling is vital in software development. While exceptions have been the standard in .NET, they can add complexity, hurt performance, and make code harder to read and maintain.

The `Result Pattern` offers an elegant, efficient alternative by making success and failure states explicit.

## 📖 Introducing the Result Pattern

The `Result pattern` is a modern approach in .NET for handling errors. Instead of throwing exceptions, methods return a `Result` object that explicitly indicates either success or failure, along with any relevant error information. This leads to clearer, more predictable, and maintainable code compared to traditional exception-based error handling.

```csharp
public Result<float> Divide(int numerator, int denominator) =>
    (denominator is 0)
        ? Result<float>.Error("Cannot divide by zero.")
        : Result<float>.Success((float) numerator / denominator);
```

## 🏆 Why the Result Pattern?

1. 📝 **Improved Readability and Maintainability**:

    - Results make the flow of success and failure explicit.
    - Encourages developers to handle both cases explicitly, reducing unhandled exceptions.

2. ⚡️ **Performance Benefits**:

    - Exceptions are expensive due to the cost of unwinding the stack.
    - In high-performance scenarios, avoiding exceptions for expected conditions can significantly improve throughput.

3. 🎯 **Predictability in APIs**:
    - The Result pattern clarifies which methods can fail and what errors to expect, leading to safer and more predictable APIs.
    - Errors are processed at the time the method call is made rather than being handled many levels above.

## 🚨 Before: Using Exceptions

```csharp
public Order GetOrder(int orderId)
{
  if (orderId <= 0)
  {
    throw new ArgumentException("Order ID must be positive", nameof(orderId));
  }

  var order = _orderRepository.Find(orderId);
  if (order == null)
  {
    throw new NotFoundException($"Order {orderId} not found.");
  }

  return order;
}

public void ProcessOrder(int orderId)
{
  try
  {
      var order = GetOrder(orderId);
      // Process the order...
  }
  catch (Exception ex)
  {
      Console.WriteLine($"Error processing order: {ex.Message}");
  }
}
```

### 🚧 Problems

-   🕳️ **Hidden Control Flow**: The flow of execution is not clear, as exceptions jump to a different context.
-   ⏳ **Performance Cost**: Throwing and catching exceptions is expensive.
-   🤔 **Unnecessary Generalization**: Many exceptions are used for predictable, non-exceptional conditions.

## ✅ After: Using the Result Pattern

```csharp
public Result<Order> GetOrder(int orderId)
{
  if (orderId <= 0)
  {
    return Result<Order>.Error("Order ID must be positive.");
  }

  var order = _orderRepository.Find(orderId);
  return order != null
    ? Result<Order>.Success(order)
    : Result<Order>.Error($"Order {orderId} not found.");
}

public void ProcessOrder(int orderId)
{
  var result = GetOrder(orderId);

  if (result.IsError)
  {
    Console.WriteLine($"Error processing order: {result.ErrorMessage}");
    return;
  }

  var order = result.Value;
  // Process the order...
}
```

### 🌟 Benefits

-   🔎 **Explicit Flow**: The flow of success and failure is straightforward and clear.
-   📈 **Predictable Costs**: No stack unwinding improves performance in predictable failure cases.
-   🐞 **Easier Debugging**: Errors are explicitly carried as part of the result, making it easier to trace and log.

## 🛡️ Result Pattern using .NET

In .NET, the result pattern can be implemented using generics to handle success and error scenarios in a more elegant and type-safe manner. This pattern eliminates the need for exception handling in many cases, improving performance and readability.

Here's how you can define the `Result<T>` class using generics:

```csharp
public class Result<T>(T? value, string? errorMessage)
{
  public string ErrorMessage => (IsError ? errorMessage : string.Empty) ?? string.Empty;

  public T Value => IsError ? default! : value!;

  public bool IsError => !IsSuccess;

  public bool IsSuccess => errorMessage is null;

  public static Result<T> Success(T value) => new(value, null);

  public static Result<T> Error(string errorMessage) => new(default, errorMessage);
}
```

## ⏱️ Benchmark: Exceptions vs. Result Pattern

This benchmark measures the performance difference between exception-based and result-based error handling in .NET. It simulates a series of method calls where calls fail on even numbers and succeed on odd numbers (a `50%` failure rate).

For each pattern:

-   `Exception-based` methods throw and catch exceptions on failure.
-   `Result-based` methods return a Result object indicating success or error.

For a range of iteration counts, the code times how long it takes to process all calls using each approach. This allows you to directly compare their performance as the workload increases.

```csharp
private static int ThrowingMethod(int i)
{
  if (i % 2 == 0)
  {
    throw new Exception("Error");
  }
  return i;
}

private static Result<int> ResultMethod(int i) =>
  i % 2 == 0
    ? Result<int>.Error("Error")
    : Result<int>.Success(i);

private static void ProcessWithExceptions(int i)
{
  // Exception-based
  try
  {
    var result = ThrowingMethod(i);
    // Handle Success
    // further process result;
  }
  catch (Exception ex)
  {
    // Handle Error
    var errorMessage = ex.Message;
  }
}

private static void ProcessWithResultPattern(int i)
{
  var result = ResultMethod(i);
  if (result.IsError)
  {
    // Handle Error
    var errorMessage = result.ErrorMessage;
    return;
  }

  // Handle Success
  var value = result.Value;
}

private void MeasureExecutionTime(string prefix, Action action)
{
  var stopwatch = Stopwatch.StartNew();
  action();
  stopwatch.Stop();

  var elapsed = stopwatch.ElapsedMilliseconds;
  _testOutputHelper.WriteLine($"{prefix}: {elapsed}ms");
}

[Fact]
public void Benchmark()
{
  // Simulate n calls, 50% fail rate
  var maxIterations = new List<int> {1, 10, 100, 1000, 10000, 100000, 1000000};

  maxIterations.ForEach(maxIteration =>
    {
      var iterations = Enumerable.Range(0, maxIteration).ToList();

      MeasureExecutionTime($"Exceptions {maxIteration}",
        () => iterations.ForEach(ProcessWithExceptions));

      MeasureExecutionTime($"Result Pattern {maxIteration}",
        () => iterations.ForEach(ProcessWithResultPattern));
    });
}
```

### 📊 Typical Results

The results below show elapsed time, in milliseconds, for the given pattern and number of iterations.

| Pattern    |   1 |  10 | 100 | 1,000 | 10,000 | 100,000 | 1,000,000 |
| ---------- | --: | --: | --: | ----: | -----: | ------: | --------: |
| Exceptions |   0 |   0 |   0 |     2 |     23 |     228 |     2,335 |
| Result     |   0 |   0 |   0 |     7 |      0 |       2 |        35 |

#### 🥡 Key takeaways

-   🚨 **Exception-Based**: Noticeably higher runtime as the number of iterations increases, mainly due to stack unwinding and added control flow complexity.
-   ✅ **Result-Based**: Consistently faster execution, benefiting from predictable control flow and simpler, cleaner code paths.

## 🎯 Conclusion

The `Result Pattern` in .NET enables explicit error handling, improves code readability, and offers performance benefits for predictable failure scenarios.

By using `Result<T>`, you make APIs safer, less error-prone, and more maintainable. This modern pattern is particularly advantageous in scenarios where exceptions are overused for non-exceptional conditions.

Adopting this pattern isn't just about avoiding exceptions; it's about writing better, clearer, and faster code.
