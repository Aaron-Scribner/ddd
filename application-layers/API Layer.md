The inheritance of the `BoundedContextControler` simplifies the reading of aggregates and the interactions with the bounded context.

The `ReadOnlyDomainSerivce` is instantiated in the constructor and available to the endpoint controller.

```
/// <summary>
/// Base controller for a bounded context that contains the service for read operations of the aggregate.
/// </summary>
/// <typeparam name="TAggregate">The type of the aggregate returned by the domain service.</typeparam>
/// <typeparam name="TIdType">The ID type for the aggregate.</typeparam>
/// <typeparam name="TReadOnlyDomainServiceInterface">The interface that defines the IDomainService.</typeparam>
public abstract class BoundedContextController<TAggregate, TIdType, TReadOnlyDomainServiceInterface> : Controller
    where TReadOnlyDomainServiceInterface : IReadOnlyDomainService<TAggregate, TIdType>
{
    protected BoundedContextController()
    {
        ReadOnlyDomainService = ServiceLocator.ServiceProvider.GetService<TReadOnlyDomainServiceInterface>();

        if (ReadOnlyDomainService == null)
        {
            throw new NullReferenceException("Unable to load IAbstractContextServiceInterface");
        }
    }

    protected TReadOnlyDomainServiceInterface ReadOnlyDomainService { get; private set; }
}
```

Implementing a controller that has a `LoanApplication` as the Root Aggregate would look similar to the following.

```
public class LoanApplicationsController : BoundedContextController<LoanApplication, string, ILoanApplicationReadOnlyDomainService>
```
Using the `ReadOnlyDomainService` simplifies the interaction with the DDD system from the controller. We can see the in the following example.

### GET /loanapplications/{loanApplicationId}

To retrieve the loan application by the given identifier, we need to read from the `ReadOnlyDomainService`

```
    [HttpGet]
    [Route("{applicationId}")]
    public async Task<IActionResult> GetById(string applicationId)
    {
        var result = await ReadOnlyDomainService.ReadAsync(applicationId);
        return Ok(result.ToModel());
    }
```

### POST /loanapplications

Creating a new loan application from the controller is straight-forward. The `Abstraction.Models.LoanApplication` received in the call to the endpoint is converted to an Aggregate, then it creates itself.

```
    [HttpPost]
    public async Task<IActionResult> Post([FromBody] Abstractions.Models.LoanApplication loanApplication)
    {
        var aggregate = loanApplication.ToAggregate();
        var result = await aggregate.CreateAsync();
        return Ok(result.ToModel());
    }
```