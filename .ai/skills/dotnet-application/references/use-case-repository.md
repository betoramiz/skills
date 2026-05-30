# Use Case Repository Pattern

Each feature folder must contain a sealed record implementing `IUseCaseRepository` that lists references to all Use Cases belonging to that feature.

```csharp
public sealed record ProductUseCases(
    Add Add,
    Delete Delete,
    List List,
    Update Update,
    GetById GetById,    
    DeletePhoto DeletePhoto,
    GetInitialData GetInitialData
) : IUseCaseRepository;
```
