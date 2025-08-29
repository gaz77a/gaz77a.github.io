---
layout: post
title: '🧹 Refactoring in C#: Practical Examples for Clean and Maintainable Code 🧹'
date: 2025-08-27 21:52:30 +1000
categories: Software Development
tags: .NET, Refactoring, Readability
---

{% include
  fancy_banner.html
    image_url="/assets/images/refactoring-in-csharp/title.png"
    alt="Title Banner"
%}

> By Gary Butler | 15 min read

<br/>

## 🚩 Introduction

Refactoring is the art of improving code without changing its external behavior. It helps make code cleaner, more efficient, and easier to maintain.
In this blog, I will explore common refactoring patterns with examples in C#.

## 1. Replace Nested Conditionals with Guard Clauses

### Problem

Deeply nested conditionals make code hard to read and maintain.

### Solution

Use guard clauses to handle exceptional cases upfront.

### Before

```csharp
public void ProcessOrder(Order? order)
{
    if (order != null)
    {
        if (order.Status == OrderStatus.Pending)
        {
           // Process the order
        }
    }
}
```

### After

```csharp
public void ProcessOrder(Order? order)
{
    if (order == null) return;
    if (order.Status != OrderStatus.Pending) return;
    // Process the order
}
```

### Benefit

Guard clauses simplify the structure by reducing nesting.

---

<br/>

## 2. Extract Method

### Problem

A single method is doing too much, making it hard to test or understand.

### Solution

Extract logically distinct operations into smaller methods.

### Before

```csharp
public void GenerateReport()
{
    Console.WriteLine("Connecting to database...");
    // Database connection logic

    Console.WriteLine("Fetching data...");
    // Data fetching logic

    Console.WriteLine("Generating report...");
    // Report generation logic
}
```

### After

```csharp
private void ConnectToDatabase() => Console.WriteLine("Connecting to database...");
private void FetchData() => Console.WriteLine("Fetching data...");
private void Generate() => Console.WriteLine("Generating report...");

public void GenerateReport()
{
    ConnectToDatabase();
    FetchData();
    Generate();
}
```

### Benefit

Each method has a single responsibility, improving readability and testability.

---

<br/>

## 3. Replace Magic Numbers with Constants

### Problem

Magic numbers obscure the purpose of the code.

### Solution

Replace them with meaningful constants.

### Before

```csharp
public double CalculateArea(double radius) =>
  3.14159 _ radius _ radius;
```

## After

```csharp
private const double Pi = 3.14159;

public double CalculateArea(double radius) =>
  Pi _ radius _ radius;
```

### Benefit

Constants make the code self-explanatory and easier to maintain.

---

<br/>

## 4. Introduce Parameter Object / DTO

### Problem

A method takes too many parameters, making it hard to understand and at risk of positional errors.

### Solution

Group related parameters into a single record when the number of parameters exceeds two.

### Before

```csharp
public void CreateOrder(string customerName, string productName, int quantity, double price)
{
  // Order creation logic
}
```

### After

```csharp
public record OrderDetails
{
  public required string CustomerName { get; init; }
  public required string ProductName { get; init; }
  public required int Quantity { get; init; }
  public required double Price { get; init; }
}

public void CreateOrder(OrderDetails orderDetails)
{
  // Order creation logic
}
```

### Benefit

Parameter objects reduce method complexity and improve reusability.

---

<br/>

## 5. Replace Loops with LINQ

### Problem

Loops can be verbose and less expressive.

### Solution

Use LINQ with Lambdas; `Method Syntax`; for concise and declarative expressions.

### Before

```csharp
List<string> activeUsers = new List<string>();
foreach (var user in users)
{
  if (user.IsActive)
  {
    activeUsers.Add(user.Name);
  }
}
```

### After

```csharp
var activeUsers = users
  .Where(user => user.IsActive)
  .Select(user => user.Name)
  .ToList();
```

### Benefit

LINQ expressions are more concise and easier to read.

---

<br/>

## 6. Fluent Style with Custom Builders

### Problem

Complex object creation often involves multiple steps, leading to verbose, repetitive, and error-prone code.

### Solution

Introduce a fluent builder pattern for more readable and maintainable object construction.

### Before

```csharp
var report = new Report();
report.SetTitle("Annual Report");
report.SetAuthor("Jane Doe");
report.AddSection("Introduction", "This is the introduction.");
report.AddSection("Summary", "This is the summary.");
report.Publish();
```

### After

```csharp
var report = new ReportBuilder()
  .WithTitle("Annual Report")
  .WithAuthor("Jane Doe")
  .AddSection("Introduction", "This is the introduction.")
  .AddSection("Summary", "This is the summary.")
  .Build();

report.Publish();
```

### Benefit

The fluent interface improves readability, supports chaining, and simplifies object creation while encapsulating complexity in the builder.

---

<br/>

## 7. Using the Adapter Pattern with Fluent Style

### Problem

When working with data from external sources, it’s common to encounter mismatches between the expected structure and the available data.

### Solution

By using an adapter pattern you can convert one interface or data structure into another, making the integration seamless. Readability can be further enhanced by using a Fluent style for the adaptors.

### Before (without Adapter pattern):

```csharp
public record Category
{
  public required string Name { get; init; }
  public required string Description { get; init; }
}

public record UploadCategoryModel
{
  public required string CategoryName { get; init; }
  public required string CategoryDescription { get; init; }
}

public class CategoryProcessor
{
  public List<UploadCategoryModel> ConvertCategoriesToUploadModels(List<Category> categories)
  {
    var uploadCategories = new List<UploadCategoryModel>();

    foreach (var category in categories)
    {
      uploadCategories.Add(
        new UploadCategoryModel()
        {
          CategoryName = category.Name,
          CategoryDescription = category.Description
        });
    }
    return uploadCategories;
  }

  public void ProcessCategories()
  {
    var categories = View.GetCategories();
    var uploadCategories = ConvertCategoriesToUploadModels(categories);
    Repository.SaveCategories(uploadCategories);
  }
}
```

### After (using Adapter pattern with fluent style)

You can implement an adapter that transforms the `Category` to `UploadCategoryModel`. This allows us to separate concerns, where the adapter handles the conversion logic. Additionally, we use fluent style to streamline the process.

```csharp
public static class CategoryExtensions
{
  // Adapter to convert a Category to UploadCategoryModel
  public static UploadCategoryModel ToUploadCategoryModel(this Category category) =>
    new()
    {
      CategoryName = category.Name,
      CategoryDescription = category.Description
    };

  // Adapter method to convert a list of categories to a list of upload models
  public static IEnumerable<UploadCategoryModel> ToUploadCategoryModels(this IEnumerable<Category> categories) =>
    categories.Select(ToUploadCategoryModel);
  }

public class CategoryProcessor
{
  public void ProcessCategories()
  {
    var uploadCategories = View
      .GetCategories()
      .ToUploadCategoryModels();

    Repository.SaveCategories(uploadCategories);
  }
}
```

### Explanation

-   **Adapter**: The extension method `ToUploadCategoryModel` converts a `Category` record to an `UploadCategoryModel`.

-   **Fluent Style**: The `ToUploadCategoryModels` extension method is chained to directly transform a collection of `Category` objects into a collection of `UploadCategoryModel` objects.
    Note the use of a `Method Group` in the calling of` ToUploadCategoryModel`

### Benefit

-   **Separation of Concerns**: The conversion logic is abstracted into the adapter, making the `CategoryProcessor` class cleaner and easier to maintain.

-   **Fluent Style**: The fluent methods make the transformation process more readable and concise.

---

<br/>

## Refactoring Patterns in .NET 8

.NET 8 brings powerful language enhancements like advanced pattern matching and switch expressions, which streamline complex control flows and improve code readability. This section explores how to refactor C# code using these modern features.

### 1. Replace Nested Conditionals with Pattern Matching and Switch expressions

#### Problem

Deeply nested `if-else` statements can be verbose and error-prone.

#### Solution

Use pattern matching and `switch expressions` to simplify conditionals.

#### Before

```csharp
public string GetDiscountMessage(Customer? customer)
{
  if (customer == null)
  {
    return "Customer not found.";
  }

  if (customer.IsActive)
  {
    if (customer.Type == CustomerType.Premium)
    {
      return "You get a 20% discount!";
    }

    if (customer.Type == CustomerType.Regular)
    {
      return "You get a 10% discount.";
    }
  }
  return "No discount available.";
}
```

#### After

```csharp
public string GetDiscountMessage(Customer? customer) =>
  customer switch
  {
    null => "No Customer supplied.",
    { Type: CustomerType.Premium, IsActive: true } => "You get a 20% discount!",
    { Type: CustomerType.Regular, IsActive: true } => "You get a 10% discount.",
    _ => "No discount available."
  };
```

#### Benefit

Pattern matching and switch expressions simplifies code by concisely handling multiple cases in a single expression.

---

<br/>

### 2. Combine Type Checking with Pattern Matching

#### Problem

Manual type checking is verbose and can lead to boilerplate.

#### Solution

Use type patterns to directly match and cast.

#### Before

```csharp
public void Process(Entity entity)
{
  if (entity is Customer customer)
  {
    Console.WriteLine($"Processing customer: {customer.Name}");
  }
  else if (entity is Order order)
  {
      Console.WriteLine($"Processing order: {order.Id}");
  }
  else
  {
    Console.WriteLine("Unknown type.");
  }
}
```

#### After

```csharp
public void Process(Entity entity) =>
  entity switch
  {
    Customer customer => Console.WriteLine($"Processing customer: {customer.Name}"),
    Order order => Console.WriteLine($"Processing order: {order.Id}"),
    _ => Console.WriteLine("Unsupported entity.")
  };
```

#### Benefit

Type patterns eliminate the need for explicit casting, reducing boilerplate.

---

<br/>

### 3. Simplify Complex If Statements with Property Patterns

#### Problem

`If statements` handling nested conditions can be hard to follow.

#### Solution

Use property patterns to directly evaluate object properties.

#### Before

```csharp
public string GetShippingCost(Order order)
{
  if (order.Customer.Type == CustomerType.Premium && order.TotalAmount > 100)
  {
    return "Free Shipping";
  }
  else if (order.Customer.Type == CustomerType.Regular && order.TotalAmount > 50)
  {
    return "$5 Shipping";
  }

  return "$10 Shipping";
}
```

#### After

```csharp
public string GetShippingCost(Order order) =>
  order switch
  {
    { Customer: { Type: CustomerType.Premium }, TotalAmount: > 100 } => "Free Shipping",
    { Customer: { Type: CustomerType.Regular }, TotalAmount: > 50 } => "$5 Shipping",
    _ => "$10 Shipping"
  };
```

#### Benefit

Property patterns make conditions clearer and more concise.

---

<br/>

### 4. Use List Patterns for Collection Matching

#### Problem

Iterating over collections to match patterns is verbose.

#### Solution

Use list patterns to directly match collection shapes.

#### Before

```csharp
public string AnalyzeNumbers(int[] numbers)
{
  if (numbers.Length == 0)
  {
    return "Empty array.";
  }

  if (numbers.Length == 1 && numbers[0] == 42)
  {
    return "The answer to life.";
  }

  if (numbers[0] == 1 && numbers[^1] == 10)
  {
    return "Starts with 1, ends with 10.";
  }

  return "No match.";
}
```

#### After

```csharp
public string AnalyzeNumbers(int[] numbers) => numbers switch
  {
    [] => "Empty array.",
    [42] => "The answer to life.",
    [1, .., 10] => "Starts with 1, ends with 10.",
    _ => "No match."
  };
```

#### Benefit

List patterns simplify working with arrays and collections.

---

<br/>

### 5. Using `and`, `or` and `is` for Cleaner Logic

#### Problem

Logical operators like `&&` and `\|\|` can clutter conditions and make logic harder to read, especially in complex expressions.

#### Solution

Replace `&&` and `\|\|` with the new `and` and `or` keywords for improved clarity and integration with modern C# pattern matching.

#### Before

```csharp
public void ProcessValue(int value)
{
  if (value > 0 && value < 100)
  {
    Console.WriteLine("Value is within range.");
  }
  else if (value <= 0 || value >= 100)
  {
    Console.WriteLine("Value is out of range.");
  }
  else
  {
    Console.WriteLine("Unknown value.");
  }
}
```

#### After

```csharp
public void ProcessValue(int value)
{
  if (value is > 0 and < 100)
  {
    Console.WriteLine("Value is within range.");
  }
  else if (value is <= 0 or >= 100)
  {
    Console.WriteLine("Value is out of range.");
  }
  else
  {
    Console.WriteLine("Unknown value.");
  }
}
```

#### Benefit

`and`, `or` and `is` improve readability by making conditions more intuitive and expressive, particularly when combined with range patterns.

---

<br/>

### 6. Using `and` and `or` in Switch Logical Patterns

#### Problem

Traditional logical operators in `switch` expressions require verbose `when` clauses, reducing clarity.

#### Solution

Directly embed `and` and `or` into switch expressions for concise logic.

#### Before

```csharp
public string GetValueDescription(int value)
{
  return value switch
  {
    int n when n > 0 && n < 100 => "Value is within range.",
    int n when n <= 0 || n >= 100 => "Value is out of range.",
    _ => "Unknown value."
  };
}
```

#### After

```csharp
public string GetValueDescription(int value) =>
  value switch
  {
    > 0 and < 100 => "Value is within range.",
    <= 0 or >= 100 => "Value is out of range.",
    _ => "Unknown value."
  };
```

#### Benefit

Using `and` and `or` in logical patterns eliminates the need for when clauses, leading to cleaner and more readable expressions.

---

<br/>

## 🎯 Conclusion

Refactoring is an ongoing process that improves code quality and developer productivity. By applying patterns like guard clauses, method extraction, and pattern matching, you can write cleaner, more maintainable C# code.

Having unit tests with a high code coverage will ensure that refactoring can be done while ensuring the stability of the code.

Start small, refactor incrementally, and always prioritize readability and simplicity.

## Resources

-   [Refactoring: Improving the Design of Existing Code by Martin Fowler](https://martinfowler.com/books/refactoring.html)
-   [C# Documentation](https://learn.microsoft.com/en-us/dotnet/csharp)
-   [Clean Code Principles](https://cleancoders.com/)
    -   This site has a number of paid courses but I suggest looking at the topics covered as a conversation starter.
