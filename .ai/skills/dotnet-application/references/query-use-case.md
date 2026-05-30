# Query Use Case Pattern

Queries are used to read state. They are immutable, should return `ErrorOr<T>` (or `PaginationResponse<T>` for paginated reads), and can use either EF Core or Dapper.

### Standard Query Example
```csharp
public class GetById : IUseCase
{
    public record Query(int Id);

    public record Response(int Id, string Name, string PhotoUrl, bool Published, int ProductsCount = 0);

    private readonly IBackendDbContext _context;
    public GetById(IBackendDbContext context) => _context = context;

    public async Task<ErrorOr<Response>> ExecuteAsync(Query request, CancellationToken cancellationToken)
    {
        var category = await _context.Categories
            .Include(x => x.Photo)
            .Include(x => x.Products)
            .Where(category => category.Id == request.Id)
            .Select(category => new Response(category.Id, category.Name, category.Photo.Url, category.Published, category.Products.Count))
            .FirstOrDefaultAsync(cancellationToken);

        if (category is null)
            return Error.NotFound();

        return category;
    }
}
```

### Paginated Query Example
```csharp
public class GetById : IUseCase
{
    public class Query : PaginationRequest
    {
        public Query(int page, int itemsPerPage)
        {
            ItemsPerPage = itemsPerPage;
            Page = page;
        }
    }

    public record Response(int Id, string Name, string PhotoUrl, bool Published, int ProductsCount = 0);

    private readonly IBackendDbContext _context;
    public GetById(IBackendDbContext context) => _context = context;

    public async Task<PaginationResponse<Response>> ExecuteAsync(Query request, CancellationToken cancellationToken)
    {
        return await _context.Categories
            .Include(x => x.Photo)
            .Include(x => x.Products)
            .Where(category => category.Id == request.Id)
            .Select(category => new Response(category.Id, category.Name, category.Photo.Url, category.Published, category.Products.Count))
            .ToPaginatedResponseAsync(request.Offset, request.ItemsPerPage, request.CurrentPage);
    }
}
```
