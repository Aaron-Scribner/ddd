The Entities layer defines the actual classes used to persist data to storage. The Entity definition is required except when using a pure json document storage such as CosmosDB or DynamoDB.

## Example
The `Order` repository persists the `Order` Root Aggregate into a MongoDB persistence store. The `Order` entity inherits from the `MongoDocument`. This base class implements all the necessary functionality to persist the data to MongoDB.

```
public class Order : MongoDocument
{
    public string CorrelationId { get; set; } = string.Empty;

    [BsonElement]
    public List<LineItem> LineItems { get; set; } = new();

    public OrderStatus Status { get; set; }
}
```