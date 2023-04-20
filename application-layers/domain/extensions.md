Properties mapping frameworks are avoid to prevent the bypassing of the validation logic that occurs when building a Root Aggregate. Extension methods ease the instantiation and conversion from Abstraction Models <-> Aggregates <-> Entities within the Controllers, Message Handlers, Aggregates, Domain Services, and Repositories

```
public static class OrderExtensions
{
    public static Models.Order ToModel(this Aggregates.Order order)
    {
        return new Models.Order
        {
            Id = order.Id ?? string.Empty,
            CorrelationId = order.CorrelationId,
            LineItems = order.LineItems.Select(x => x.ToModel()).ToList(),
            Status = order.Status,
        };
    }

    public static Aggregates.Order ToAggregate(this Models.Order order)
    {
        return new Aggregates.Order.With(order.CorrelationId, order.Status)
            .Id(order.Id)
            .LineItems(order.LineItems.Select(l => l.ToAggregate()))
            .Build();
    }

    public static Entities.Order ToEntity(this Aggregates.Order order)
    {
	    var entity = new Entities.Order
        {
            CorrelationId = order.CorrelationId,
            LineItems = order.LineItems.Select(x => x.ToEntity()).ToList(),
            Status = order.Status,
        };

        if (ObjectId.TryParse(order.Id, out var id))
        {
            entity.Id = id;
        }

        return entity;
    }

    public static Aggregates.Order ToAggregate(this Entities.Order order)
    {
        return new Aggregates.Order.With(order.CorrelationId, order.Status)
            .Id(order.Id.ToString())
            .LineItems(order.LineItems.Select(l => l.ToAggregate()))
            .Build();
    }
}

```