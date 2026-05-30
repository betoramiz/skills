# Domain Unit Test Pattern

Unit tests focused on testing business logic and invariants in isolation, with no database dependencies, using xUnit and FluentAssertions.

```csharp
[Fact]
public void Product_UpdatePrice_ValidPrice_UpdatesPrice()
{
    // Arrange
    var productResult = Product.Create("Test Product", 100m);
    var product = productResult.Value;

    // Act
    var result = product.UpdatePrice(150m);

    // Assert
    result.IsError.Should().BeFalse();
    product.Price.Should().Be(150m);
}

[Fact]
public void Product_UpdatePrice_InvalidPrice_ReturnsValidationError()
{
    // Arrange
    var productResult = Product.Create("Test Product", 100m);
    var product = productResult.Value;

    // Act
    var result = product.UpdatePrice(-50m);

    // Assert
    result.IsError.Should().BeTrue();
    result.FirstError.Type.Should().Be(ErrorType.Validation);
}
```
