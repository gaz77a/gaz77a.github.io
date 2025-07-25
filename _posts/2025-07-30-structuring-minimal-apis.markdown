---
layout: post
title: '🛠️ Structuring Minimal APIs in .NET 8: A Modern Approach with Dependency Injection, Handlers, and Endpoint Filters 🛠️'
date: 2025-07-30 21:52:30 +1000
categories: Software Development
tags: .NET, Minimal APIs, Dependency Injection, Handlers, Endpoint Filters
---

<div style="
  max-width: 1100px;
  width: 100%;
  margin: 42px auto 44px auto;
  border-radius: 28px;
  box-shadow:
    0 20px 80px -4px rgba(44,60,180,0.72),             /* bold shadow */
    0 28px 128px 0 rgba(44,50,100,0.19);               /* halo */
  overflow: visible;        /* shadow spills out, not clipped */
  background: none;
  position: relative;
  display: block;
">
  <div style="
    border-radius: 28px;
    overflow: hidden;       /* ensures image corners are rounded */
    width: 100%;
    height: 136px;          /* Flat banner */
  ">
    <img
      src="/assets/images/structuring-minimal-apis/title.png"
      alt="Title image"
      style="
        display: block;
        width: 100%;
        height: 100%;
        object-fit: cover;
        background: #dde8fc;
        border: 0;
      "
    />
  </div>
</div>

> By Gary Butler | 8 min read

<br/>

## 🚩 Introduction

Minimal APIs; first introduced in .NET 6 have become a powerful tool for building lightweight, efficient, and clean web applications. Unlike traditional MVC endpoints, Minimal APIs focus on simplicity, allowing developers to achieve more with less boilerplate while maintaining flexibility and testability. In this blog, I will show how

to structure a Minimal API project using:

-   🟢 Fluent-style Dependency Injection (DI)
-   🟦 Handlers with associated services/repositories
-   🟡 Endpoint filters for reusable logic; available from .NET 7 onwards

Finally, we’ll compare this approach to popular alternatives like FastEndpoints and traditional MVC endpoints to understand its unique advantages.

<br/>

## 👋 Hello World

Using top-level commands and minimal APIs a `Hello World` example can be created using only a few lines of code:

```csharp
var app = WebApplication
   .CreateBuilder(args)
   .Build();

app.MapGet("/", () => "Hello World!");

app.Run();
```

<br/>

## 🚧 Feature Based Example

Let’s explore how this works in a real-world scenario. We need to create an API for managing various features, focusing on the Products feature. The requirements are:

-   📌 Endpoints
    -   A `GetProducts` endpoint that returns a `List<Product>`.
-   🛡️ Security
    -   All requests must include a valid API key.
-   🗄️ Data Source
    -   The data will be retrieved from a `ProductRepository`.
    -   The `ProductRepository` can source data from multiple backends, such as:
        -   A database.
        -   An external API.

The endpoints for a feature will be split into a number of distinct layers to allow for a clear separation of concerns and enhancing maintainability. These layers are:

-   👉 Handler
    -   Acts as the entry point for HTTP requests.
    -   Validates inputs and maps request data to models.
    -   Delegates business logic to the Service layer.
    -   Returns appropriate HTTP responses with status codes.
-   ⚙️ Service
    -   Contains core business logic and applies rules/validations.
    -   Orchestrates data retrieval and transformation processes.
    -   Ensures modularity by interacting with repositories instead of data sources directly.
-   🗃️ Repository
    -   Manages data access and communication with databases or external APIs.
    -   Encapsulates complex queries to abstract data layer details.
    -   Provides a clean interface for the Service layer.

<br/>

## Step 1️⃣: Feature-Based Handlers

Each feature will have a handler that maps endpoints for its specific domain. This keeps routing logic modular and feature-focused.

### ProductFeature/Handlers/ProductsHandler.cs

```csharp
public static class ProductsHandler
{
   private static async Task<IResult> GetAllProducts(IProductService productService)
   {
      var products = await productService.GetAllProductsAsync();
      return Results.Ok(products);
   }

   public static IEndpointRouteBuilder MapProductEndpoints(this IEndpointRouteBuilder endpoints)
   {
      endpoints.MapGet("/products", () => GetAllProducts)
         .AddEndpointFilter<ProductsEndpointFilter>()
         .Produces<List<Product>>()
         .Produces(StatusCodes.Status400BadRequest)
         .WithName("GetProducts")
         .WithOpenApi();

      return endpoints;
   }
}
```

<br/>

## Step 2️⃣: Feature Service

Services encapsulate business logic with each feature maintaining its own service for a clear separation of responsibilities.

### ProductFeature/Services/IProductService.cs

```csharp
public interface IProductService
{
   Task<IEnumerable<Product>> GetAllProductsAsync();
}
```

### ProductFeature/Service/ProductService.cs

```csharp
public class ProductService(IProductRepository repository) : IProductService
{
   public async Task<IEnumerable<Product>> GetAllProductsAsync() =>
      await repository.GetAllProductsAsync();
}
```

<br/>

## Step 3️⃣: Feature Repository

Repositories handle data access. Each feature maintains its own repository for a clear separation of responsibilities. The repository will also adapt the underlying data to its own model so as to not leak the underlying data structures.

### ProductFeature/Repository/IProductRepository.cs

```csharp
public interface IProductRepository
{
   Task<IEnumerable<Product>> GetAllProductsAsync();
}
```

### ProductFeature/Repository/IProductRepository.cs

```csharp
public interface IProductRepository
{
   Task<IEnumerable<Product>> GetAllProductsAsync();
}
```

### ProductFeature/Repository/ProductRepository.cs

```csharp
public class ProductRepository : IProductRepository
{
   public async Task<IEnumerable<Product>> GetAllProductsAsync()
   {
      // Mocked data return
      var products = new List<Product>
      {
         new() { Id = 1, Name = "Laptop", Price = 1200.24m },
         new() { Id = 2, Name = "Smartphone", Price = 800.56m }
      };

      return await Task.FromResult(products);
   }
}
```

### ProductFeature/Repository/Models/Product.cs

```csharp
public record Product
{
   public required int Id { get; init; }
   public required string Name { get; init; }
   public required decimal Price { get; set; }
}
```

<br/>

## Step 4️⃣: Using Endpoint Filters

Endpoint filters enable reusable logic for Minimal APIs, such as validation or authentication.

### EndpointFilters/ValidationFilter.cs

```csharp
public class ProductsEndpointFilter : IEndpointFilter
{
   public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext context, EndpointFilterDelegate next) =>
      context.HttpContext.Request.Query.ContainsKey("apiKey")
         ? await next(context) // Proceed to the next filter or endpoint
         : Results.BadRequest("API Key is missing");
}
```

<br/>

## Step 5️⃣: Fluent Endpoint Registration

Register each feature's handler in Program.cs for a clean, modular setup.

### Program.cs

```csharp
var app = WebApplication
   .CreateBuilder(args)
   .AddCoreConfiguration()
   .AddUserSecretsIfNeeded()
   .AddProductConfiguration()
   .Build()
   .UseMiddleware()
   .MapEndpoints();

app.Run();
```

<br/>

## Step 6️⃣: Fluent Dependency Injection

Create an extension for each features dependency injection configuration.

### Configuration/ProductConfigurationExtensions.cs

```csharp
public static WebApplicationBuilder AddProductConfiguration(this WebApplicationBuilder builder)
{
   builder.Services.AddScoped<IProductRepository>(
      _ => new ProductRepository()
   );

   builder.Services.AddScoped<IProductService>(
      x => new ProductService(
         x.GetRequiredService<IProductRepository>())
   );

   return builder;
}
```

## 🗂️ Project Structure

Here’s how a feature-based folder structure might look:

```
src/
├── 📁 Configuration/
│   ├── 📝 CoreConfigurationExtensions.cs
│   ├── 📝 EndPoints.cs
│   ├── 📝 UserSecretsExtensions.cs
│   ├── 📝 WebApplicationBuilderExtensions.cs
│
├── 📁 Features/
│   ├── 📁 Configuration/
│        ├── 📝 FeaturesConfiguration.cs
│
├── 📁 HealthCheckFeature/
│   ├── 📁 Handlers/
│        ├── 📝 HealthCheckHandler.cs
│
├── 📁 ProductFeature/
│   ├── 📁 Configuration/
│        ├── 📝 ProductsConfiguration.cs
│   ├── 📁 EndpointFilters/
│        ├── 📝 ProductsEndpointFilter.cs
│   ├── 📁 Handlers/
│        ├── 📝 ProductsHandler.cs
│   ├── 📁 Services/
│        ├── 📝 IProductService.cs
│        ├── 📝 ProductService.cs
│   ├── 📁 Repository/
│        ├── 📁 Models/
│              ├── 📝 Product.cs
│        ├── 📝 IProductRepository.cs
│        ├── 📝 ProductRepository.cs
│
├── 📝 Program.cs

```

<br/>

## 🏆 Benefits of This Approach

### 1️⃣ ⭐ **Feature-Based Modularity**

Each feature (e.g., Product, Order) is self-contained with its own handlers, services, and repositories. This makes it easier to:

-   Navigate the codebase.
-   Add new features without affecting others.
-   Scale the application as it grows.

### 2️⃣ 🧪 **Testability**

-   Handlers, services, and repositories are decoupled and easily testable.
-   Dependency Injection ensures mock implementations can be injected during unit testing.

### 3️⃣ 🧩 **Flexibility with EndpointFilters**

EndpointFilters, such as `ProductsEndpointFilter`, allows separation of middleware between endpoints.

### 4️⃣ 🧹 **Cleaner Program.cs**

Using fluent-style configuration keeps the `Program.cs` file clean and focused on application-level setup. Feature-specific logic is delegated to handlers.

### 5️⃣ ⚖️ **Comparison**

Below is a comparison of three popular approaches for building APIs in .NET — MVC, Minimal APIs, and FastEndpoints — each offering distinct patterns and capabilities.

  <table style="border-collapse:collapse; width:100%;">
    <thead style="position:sticky; top:0; background:#fafafa; z-index:1;">
      <tr>
        <th>✨ <strong>Feature</strong></th>
        <th>🔷 <strong>MVC</strong></th>
        <th>🟢 <strong>Minimal APIs</strong></th>
        <th>🟠 <strong>FastEndpoints</strong></th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td><strong>🎯 Use Case</strong></td>
        <td>Full-featured web applications (Views, Controllers, Models)</td>
        <td>Lightweight, small to medium APIs</td>
        <td>Lightweight APIs with structured approach</td>
      </tr>
      <tr>
        <td><strong>🧩 Framework</strong></td>
        <td><a href="http://asp.net/">ASP.NET Core MVC</a></td>
        <td><a href="http://asp.net/">ASP.NET Core Minimal APIs</a></td>
        <td><a href="http://asp.net/">ASP.NET Core with FastEndpoints library</a></td>
      </tr>
      <tr>
        <td><strong>🛣️ Routing</strong></td>
        <td>Attribute-based (e.g., <code>[HttpGet]</code>) or conventional</td>
        <td>Fluent, inline routing or via Extensions</td>
        <td>Declarative, endpoint-based with attributes</td>
      </tr>
      <tr>
        <td><strong>📝 Boilerplate Code</strong></td>
        <td>Moderate to high, requires controllers and models</td>
        <td>Minimal, customizable for separation of concerns</td>
        <td>Minimal, but enforces endpoint structures</td>
      </tr>
      <tr>
        <td><strong>🚀 Performance</strong></td>
        <td>Moderate (slightly more overhead)</td>
        <td><strong>Extremely high</strong>, possibly faster than FastEndpoints</td>
        <td>Very high, close to Minimal APIs</td>
      </tr>
      <tr>
        <td><strong>⚡ Async Support</strong></td>
        <td>Fully supports <code>async</code></td>
        <td>Fully supports <code>async</code></td>
        <td>Fully supports <code>async</code></td>
      </tr>
      <tr>
        <td><strong>🔒 Type Safety</strong></td>
        <td>Strong, compile-time type safety</td>
        <td>Strong, compile-time type safety</td>
        <td>Strong, compile-time type safety</td>
      </tr>
      <tr>
        <td><strong>🧩 Dependency Injection</strong></td>
        <td>Built-in, well-integrated</td>
        <td>Built-in, well-integrated</td>
        <td>Built-in, well-integrated</td>
      </tr>
      <tr>
        <td><strong>✅ Validation</strong></td>
        <td>Manual or <code>FluentValidation</code></td>
        <td>Manual or <code>FluentValidation</code></td>
        <td>Built-in request and response validation</td>
      </tr>
      <tr>
        <td><strong>📚 OpenAPI Documentation</strong></td>
        <td>Requires Swashbuckle/NSwag integration</td>
        <td>Requires Swashbuckle/NSwag integration</td>
        <td>Built-in OpenAPI documentation</td>
      </tr>
      <tr>
        <td><strong>🔀 Separation of Concerns</strong></td>
        <td>Enforced via Controllers, Models, Views</td>
        <td>Fully achievable through custom structure</td>
        <td>Encouraged via structured endpoints</td>
      </tr>
      <tr>
        <td><strong>📈 Scalability</strong></td>
        <td>High, suitable for large, complex applications</td>
        <td>Suitable for small to medium-sized applications</td>
        <td>Suitable for small to medium-sized applications</td>
      </tr>
      <tr>
        <td><strong>🌏 Community</strong></td>
        <td>Large, long-established <a href="http://asp.net/">ASP.NET MVC</a> community</td>
       <td>Growing popularity with Minimal APIs users</td>
       <td>Smaller but active and focused community</td>
       </tr>
       <tr>
       <td><strong>🎓 Learning Curve</strong></td>
       <td>Moderate to high, especially for beginners</td>
       <td>Low to moderate, customizable for use cases</td>
       <td>Low to moderate, requires understanding FastEndpoints</td>
       </tr>
       <tr>
       <td><strong>🏅 Key Differentiators</strong></td>
       <td>Separation of concerns with Views, Controllers, Models</td>
       <td>Inline, fast, customizable for large-scale APIs</td>
       <td>Adds validation and structure to Minimal APIs</td>
       </tr>
       </tbody>
 </table>

## 🔑 Key Features of Minimal API’s

### 1️⃣ Separation of Concerns in Minimal APIs:

Minimal APIs allow separation of concerns by moving logic into services, handlers, or dedicated classes, mimicking MVC-like architecture while remaining lightweight.

### 2️⃣ Performance:

Minimal APIs are extremely fast because of their lightweight architecture and reduced middleware pipeline, often faster than frameworks like FastEndpoints.

### 3️⃣ Use Cases:

Minimal APIs can scale to more complex applications with proper organization (e.g., using service classes, extensions for modularity).

## 🎯 Conclusion

By structuring Minimal APIs with a feature-based organization, DI, and reusable endpoint filters, you create a clean and scalable architecture that is easy to maintain and test. This approach leverages the strengths of .NET 8, providing a modern, performant alternative to MVC and competing frameworks like FastEndpoints.

Adopting this modular design ensures your Minimal APIs remain efficient and maintainable as your application grows.
