# Repository Interface Pattern

Define repository interfaces in the Domain layer to abstract complex queries or persistence actions. We do not use the generic repository pattern; instead, define specific operations returning dedicated read/write DTO records.

```csharp
public interface IProductRepository
{
    Task<ProductByIdDto?> GetByIdAsync(Guid id);
    Task<GetProductsWithFiltersDto> GetProductsWithFiltersAsync(GetProductsWithFiltersRequest request);
}

public record ProductByIdDto(Guid Id, string Name, decimal Price);
public record GetProductsWithFiltersRequest(string? Name, decimal? MinPrice, decimal? MaxPrice);
public record GetProductsWithFiltersDto(List<ProductByIdDto> Products, int TotalCount);
```
