Value objects are pieces of data that may be part of an aggregate, but they are not unique to the bounded context. An example of a value object would be a street address associated with an applicant on an application. The street address may be owned by parties over time, but the address itself never changes.

## Example
In our Point of Sale Bounded Context, a product does not contain an identifier, thus it is treated as a Value Object.
```
public sealed partial class Product
{
    public decimal Cost { get; private set; }

    [Required]
    public string Name { get; private set; } = string.Empty;
}
```