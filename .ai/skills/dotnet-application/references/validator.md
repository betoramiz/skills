# Validation Pattern

Validation is performed in the Application layer using FluentValidation. A validator class is typically defined inside the UseCase class itself.

```csharp
using FluentValidation;
using Floreria.Application.Common.Validation;

public class Create : IUseCase
{
    public record Command(string Name);
    public record Response(int Id);
    
    private class ValidateCommand : AbstractValidator<Command>
    {
        public ValidateCommand()
        {
            RuleFor(x => x.Name)
                .NotEmpty()
                .NotNull()
                .MaximumLength(100);
        }
    }

    private readonly IBackendDbContext _context;
    private readonly IValidator<Command> _validator;
    
    public Create(IBackendDbContext context, IValidator<Command> validator)
    {
        _context = context;
        _validator = validator;        
    }

    public async Task<ErrorOr<Response>> ExecuteAsync(Command request, CancellationToken cancellationToken)
    {
        if (_validator.IsNotValid(request, out var errors))
            return errors.Errors();
            
        // ... execution logic
    }
}
```
