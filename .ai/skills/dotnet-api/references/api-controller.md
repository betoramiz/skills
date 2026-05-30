# API Controller Pattern

API controllers in the FloreriaStore project inherit from the `ApiController` base class and coordinate with application layer UseCase repositories.

```csharp
public class CategoryController(CategoryUseCases categoryUseCases) : ApiController
{
    [HttpGet]
    public async Task<IActionResult> Get(CancellationToken cancellationToken)
    {
        var result = await _categoryUseCases.GetById.ExecuteAsync(new GetById.Query(id), cancellationToken);
        return result.Match(Ok, Problem);
    }
}
```
