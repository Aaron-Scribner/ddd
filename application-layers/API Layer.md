The API layer is the entry point for the bounded context for both endpoints and the handling of domain messages published by other parts of the ecosystem.


### `BoundedContextController`
The abstract `BoundedContextController` provides access to the `IReadOnlyRepository` for the inheriting controller. This allows the controller to read one, or many, root aggregates and to call the operations on the root aggregate.

Using the service locator is a known anti-pattern of dependency injection, but it is (currently) accepted in this pattern. The use of the service locator allows the use a parameter-less constructor while still injecting the object into the base class.

```
/// <summary>
/// Base controller for a bounded context that contains the service for read operations of the aggregate.
/// </summary>
/// <typeparam name="TRepository">The interface for the read-only repository.</typeparam>
public abstract class BoundedContextController<TRepository> : Controller
{
    /// <summary>
    /// Initializes a new instance of the <see cref="BoundedContextController{TRepository}"/> class.
    /// </summary>
    /// <exception cref="NullReferenceException">Occurs when <see cref="Repository"/> is missing from the DI container.</exception>
    protected BoundedContextController()
    {
        Repository = ServiceLocator.ServiceProvider.GetService<TRepository>();

        if (Repository == null)
        {
            throw new NullReferenceException($"Unable to load IAbstractContextServiceInterface");
        }
    }

    /// <summary>
    /// Gets the repository for reading root aggregates.
    /// </summary>
    protected TRepository Repository { get; }
}
```

## `OrdersController`

The orders controller inherits from the generic base and specifies in the interface for the read-only repository. To implement saving a new order or retrieving an order by its identifier, we create the corresponding `POST` and `GET` endpoints.

When return data from the endpoint, we want to return the abstraction model instead of the root aggregate. We may return the root aggregate, but we have no exposed the shape of the aggregate outside of the bounded context like we do with the [abstraction layer](./Abstractions.md).

### `OrdersController.GetById`
```
    [HttpGet("{id}")]
    public async Task<IActionResult> GetById(string id, CancellationToken ct = default)
    {
        var order = await Repository.GetOrderByIdAsync(id, ct);

        if (order == null)
        {
            return NotFound();
        }

        return Ok(order.ToModel());
    }
```

### `OrdersController.Create`

```
    [HttpPost]
    public async Task<IActionResult> Create([FromBody] Abstractions.Models.Order order, CancellationToken ct = default)
    {
        var savedOrder = await order.ToAggregate().CreateAsync(ct);

        var uri = new Uri($"{Request.GetEncodedUrl()}/{savedOrder.Id}");

        return Created(uri, savedOrder.ToModel());
    }
```

All operations for the bounded context must go through the root aggregate. If we would like to add a line item to an existing order, we first need to retrieve the `Order` root aggregate, then call the operation on the root aggregate.

In the following example, we retrieve the `Order` by its `Id`, then call `AddLineItemAsync`. The [domain layer](./domain/Domain%20Layer.md) contains more implementation details for aggregates, but the `IEnumerable` objects on an aggregate are read-only and immutable outside of the scope of the aggregate.

### `Order` aggregate (snippet)
```
    private readonly List<LineItem> _lineItems = new();

    public IReadOnlyList<LineItem> LineItems => _lineItems.AsReadOnly();
```

In this example, we define a `POST` endpoint since we are creating a new line item. It is debatable to have this as a `PUT` for the root aggregate as well. The suggestion is for the individual team to define how they view the modification of a root aggregate by adding new values to a composed collection inside said root aggregate.

```
    [HttpPost("{orderId}/LineItems")]
    public async Task<IActionResult> AddLineItem(string orderId, [FromBody] Abstractions.Models.LineItem item, CancellationToken ct = default)
    {
        var order = await Repository.GetOrderByIdAsync(orderId, ct);

        if (order == null)
        {
            return NotFound();
        }

        var updatedOrder = await order.AddLineItemAsync(item.ToAggregate(), ct);
        return Ok(updatedOrder.ToModel());
    }
```