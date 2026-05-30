# Rich Domain Entity Pattern

Entities should encapsulate behavior and data. Avoid anemic models. Use private setters for internal state management, public methods for business operations, and the `ErrorOr` library to handle validation and execution errors.

```csharp
public class Product
{
    public Guid Id { get; private set; }
    public string Name { get; private set; }
    public decimal Price { get; private set; }

    private Product(Guid id, string name, decimal price)
    {
        Id = id;
        Name = name;
        Price = price;
    }

    public static ErrorOr<Product> Create(string name, decimal price)
    {
        if (string.IsNullOrWhiteSpace(name))
            return Error.Validation("Name is required");
        if (price <= 0)
            return Error.Validation("Price must be positive");

        return new Product(Guid.NewGuid(), name, price);
    }

    public ErrorOr<Success> UpdatePrice(decimal newPrice)
    {
        if (newPrice <= 0)
            return Error.Validation("Price must be positive");
            
        Price = newPrice;
        return Result.Success;
    }
}
```
