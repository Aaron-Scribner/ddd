# Definition

The Aggregate is an object that is unique to the system and will contain an identifier used to access, update, or delete it. 

## Example

The `Order` is a unique set of data in the `PointOfSale` Bounded Context. The `Order` contains a read-only list of `LineItem` and an `OrderStatus`

## Defining an Aggregate

An `Aggregate` consists of three separate partial class definitions. For the `Order` aggregate, we will have the following three files:

- Object Shape: `Order.cs`
- Functions: `Order.Operations.cs`
- Fluent Builder: `Order.With.cs`


## Object Shape
The `Order` Root Aggregate inherits from the `RootAggregate` base class, which provides access to an `IRepository`, an `ILogger`, and provides an `Id` property. In addition to the properties from the base class, we define the additional shape for the `Order`.

```
public sealed partial class Order : RootAggregate<string, IOrderWriteRepository>
{
    private readonly List<LineItem> _lineItems = new();

    [Required]
    public string CorrelationId { get; private set; } = string.Empty;

    public IReadOnlyList<LineItem> LineItems => _lineItems.AsReadOnly();

    public OrderStatus Status { get; private set; }
}
```
## Fluent Builder
Aggregates make use of a fluent builder pattern to hydrate the Aggregate with data, perform validation, and return the object. This is done with an internal `With` class. The pattern used for the `With` class makes sure of the constructor for required properties and optional properties are added with a fluent syntax.

For the `Order`, the `CorrelationId` and `OrderStatus` are required properties as the minimal set of data for an `Order` Aggregate. An `Order` may be created with nothing more than an `CorrelationId`, and `Status`. The `Id` and `LineItems are optional pieces of data.

The `With` class definition then consists of the following.

```
public sealed partial class Order
{
    public class With : FluentBuilder<Order>
    {
        public With(string correlationId, OrderStatus status)
        {
            Instance.CorrelationId = correlationId;
            Instance.Status = status;
        }

        public With Id(string? id = null)
        {
            Instance.Id = id ?? string.Empty;

            return this;
        }

        public With LineItems(IEnumerable<LineItem> items)
        {
            Instance._lineItems.AddRange(items);

            return this;
        }
    }
}
```

Building a Root Aggregate should only be done using the fluent syntax and calling `Build()`

A snippet from the [Order Extensions](./extensions.md) shows how the Root Aggregate is built

```
    public static Aggregates.Order ToAggregate(this Models.Order order)
    {
        return new Aggregates.Order.With(order.CorrelationId, order.Status)
            .Id(order.Id)
            .LineItems(order.LineItems.Select(l => l.ToAggregate()))
            .Build();
    }
```

## Operations
The public functions for a Root Aggregate exist in the `{AggregateName}.Operations.cs` file. The Operations implement all of the business logic as it pertains to the single Aggregate. For transactions that involve more than one Aggregate, we make use of a Domain Service.

The Root Aggregate contains both a `Repository` and `MessagePublisher`. This provide persistence for the Root Aggregate and the publishing of Domain Events related to the Operations performed on the Root Aggregate.

### `Order.CreateAsync()`
```
    public async Task<Order> CreateAsync(CancellationToken ct = default)
    {
        if (Status != OrderStatus.New)
        {
            throw new DomainException($"Order must be in {nameof(OrderStatus.New)} status when created.");
        }

        Status = OrderStatus.Created;

        var order = await Repository.CreateAsync(this, ct);

        return order;
    }
```

### `Order.AddLineItemAsync()`
```
    public async Task<Order> AddLineItemAsync(LineItem lineItem, CancellationToken ct = default)
    {
        if (Status != OrderStatus.Created)
        {
            throw new DomainException($"Cannot add a lineItem to an order that is not in {nameof(OrderStatus.Created)} status.");
        }

        if (_lineItems.Any(l => l.Product.Name.ToLower() == lineItem.Product.Name.ToLower()))
        {
            throw new DomainException($"Product {lineItem.Product.Name} has already been added to the order.");
        }

        _lineItems.Add(lineItem);

        var order = await Repository.SaveOrderAsync(this, ct);

        return order;
    }
```