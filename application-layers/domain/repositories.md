Repositories within the system should implement the generic interface defined in `Platform.Blocks.Persistence`. This contains the standard interface implementation for any `IRepository` provided by the Platform libraries.

## `IRepository` interface
```
public interface IRepository<TObject, TIdType>
{
    IQueryable<TObject> AsQueryable();

    IEnumerable<TObject> All();

    Task<List<TObject>> AllAsync(CancellationToken ct = default);

    List<TObject> Query(Expression<Func<TObject, bool>> predicate);

    Task<List<TObject>> QueryAsync(Expression<Func<TObject, bool>> predicate, CancellationToken ct = default);

    TObject? QueryOne(Expression<Func<TObject, bool>> predicate);

    Task<TObject?> QueryOneAsync(Expression<Func<TObject, bool>> predicate, CancellationToken ct = default);

    TObject? Read(TIdType id);

    Task<TObject?> ReadAsync(TIdType id, CancellationToken ct = default);

    TObject Create(TObject data);

    Task<TObject> CreateAsync(TObject data, CancellationToken ct = default);

    List<TObject> CreateMany(List<TObject> items);

    Task<List<TObject>> CreateManyAsync(List<TObject> items, CancellationToken ct = default);

    TObject Update(TObject data);

    Task<TObject> UpdateAsync(TObject data, CancellationToken ct = default);

    void Delete(TObject data);

    Task DeleteAsync(TObject data, CancellationToken ct = default);

    void DeleteMany(Expression<Func<TObject, bool>> predicate);

    Task DeleteManyAsync(Expression<Func<TObject, bool>> predicate, CancellationToken ct = default);
}
```

## `MongoRepository`
The `MongoRepository` is part of the Platform libraries and handles all of the implementation details associated with reading and writing data from a MongoDB. The base class has a definition of

```
public abstract class MongoRepository<TObject, TIdType> : IRepository<TObject, TIdType>
    where TObject : MongoDocument
```

## Bounded Context Repositories
The Bounded Context makes use of two different interfaces to provide repository access, a read-only interface and a write interface.

### `IOrderReadOnlyRepository`
```
public interface IOrderReadOnlyRepository
{
    Task<Order?> GetOrderByIdAsync(string id, CancellationToken token);
}
```

### `IOrderWriteRepository`
```
public interface IOrderWriteRepository
{
    Task<Order> CreateAsync(Order order, CancellationToken token);

    Task<Order> SaveOrderAsync(Order order, CancellationToken token);
}
```

The `IOrderReadOnlyRepository` is used in the API layer. The `IOrderWriteRepository` is used withing the Root Aggregate. The `OrderRepository` implements both of the interfaces, as well as inherits from the `MongoRepository`.

```
public sealed class OrderRepository :
    MongoRepository<Entities.Order, string>,
    IOrderReadOnlyRepository,
    IOrderWriteRepository
{
    public OrderRepository(MongoSettings settings) : base(settings) { }

    public async Task<Order?> GetOrderByIdAsync(string id, CancellationToken token)
    {
        var result = await ReadAsync(id, token);

        return result?.ToAggregate();
    }

    public async Task<Order> CreateAsync(Order order, CancellationToken token)
    {
        var result = await CreateAsync(order.ToEntity(), token);

        return result.ToAggregate();
    }

    public async Task<Order> SaveOrderAsync(Order order, CancellationToken token)
    {
        var result = await UpdateAsync(order.ToEntity(), token);

        return result.ToAggregate();
    }
}
```