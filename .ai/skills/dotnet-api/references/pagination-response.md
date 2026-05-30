# Pagination Response Pattern

When returning paginated lists from an API controller, invoke the paginated use case and return the `Ok` response.

```csharp
[HttpGet]
public async Task<IActionResult> Get(int page, int pageSize, CancellationToken cancellationToken)
{
    var result = await _categoryUseCases.Get.ExecuteAsync(new Get.Query(page, pageSize), cancellationToken);
    return Ok(result);
}
```
