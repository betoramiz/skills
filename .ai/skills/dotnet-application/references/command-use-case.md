# Command Use Case Pattern

Commands are used to mutate state. They are immutable records that should be validated and return `ErrorOr<T>`.

```csharp
public class Create : IUseCase
{
    public record Command(string Name);

    public record Response(int Id);

    private class ValidateCommand : AbstractValidator<Command>
    {
        public ValidateCommand()
        {
            RuleFor(x => x.Name).NotEmpty().NotNull().MaximumLength(100);
        }
    }
    
    private readonly IBackendDbContext _context; // reference to the database context
    public Create(IBackendDbContext context) => _context = context;

    public async Task<ErrorOr<Response>> ExecuteAsync(Command request, CancellationToken cancellationToken)
    {
        var validator = new ValidateCommand();
        var validate = await validator.ValidateAsync(request, cancellationToken);
        if (!validate.IsValid)
            return ErrorHelper.GetValidationErrors(validate.Errors);

        var category = Domain.Category.Category.Create(request.Name);
        if (category.IsError)
            return category.FirstError;

        var result = await _context.SaveDataAsync(cancellationToken);
        if (result.IsError)
        {
            _logger.LogError("No se pudo crear el costo: {Errors}", string.Join(", ", result.Errors.Select(x => x.Description)));
            return result.Errors;
        }
                    
        return new Response(category.Value.Id);
    }
}
```
