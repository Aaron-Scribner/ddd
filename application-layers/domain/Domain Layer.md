The Domain layer consists of Aggregates and Value Objects.
Cases in which the Aggregate is persisted to a data store that is something other than a json document store, the definition of the Entity model is also required.

## Value Objects

Value objects are pieces of data that may be part of an aggregate, but they are not unique to the system. An example of a value object would be a street address associated with an applicant on an application. The street address may be owned by multiple applicants and may change over time, but the address itself never changes.

## Aggregates

The Aggregate is an object that is unique to the system and will contain an identifier used to access, update, or delete it. In the case of our example, the `LoanApplication` is unique to a user within the system and will have a unique identifier associated with each record inserted into the data store.

### Aggregate Structure

The Aggregate is broken into three separate partial classes, each with its own specific purpose. The main part of the Aggregate is the file name containing just the name of the Aggregate. In addition to that file, there is an Operations file and a With file.

The operations file contains all of the functions are that are accessible on the aggregate. 

**Important: ONLY the Root Aggregate should publicly expose the functions on the Aggregate. All other composed aggregates within the Root Aggregate must have `internal` accessibility only**

### Creating a Root Aggregate

By inheriting the `RootAggreate<TObject, TIdType, TDomainServiceInterface>` we have the ability to call the `DomainService` to interact with the `Persistence` layer within the Aggregate. This exposes the ability to persist the aggregate to a data store and publish events. This functionality is not accessible to the `Aggreate` object.

```
/// <summary>
/// Base class to provide common functionality for a Root Aggregate.
/// </summary>
/// <typeparam name="TObject">The underlying aggregate type, AKA 'this'.</typeparam>
/// <typeparam name="TIdType">The data type for the identifier of the aggregate.</typeparam>
/// <typeparam name="TDomainServiceInterface">The interface that defines the IDomainService.</typeparam>
public abstract class RootAggregate<TObject, TIdType, TDomainServiceInterface> : Aggregate<TIdType>
    where TObject : RootAggregate<TObject, TIdType, TDomainServiceInterface>
    where TDomainServiceInterface : IDomainService<TObject, TIdType, TDomainServiceInterface>
```

####Aggregate Properties

The properties of the Aggregate are read-only and only mutable through the Operations

Filename: LoanApplication.cs

```
public partial class LoanApplication : RootAggregate<LoanApplication, string, ILoanApplicationDomainService>
{
    private readonly List<Applicant> _applicants = new();

    public IReadOnlyList<Applicant> Applicants => _applicants.AsReadOnly();

    public LoanApplicationStatus Status { get; private set; }
}
```

#### With

The `With` internal class is the fluent builder for the Aggregate and contains the validation logic to ensure that the Aggregate is valid. It also has the ability to set the default values on the Aggregate.

The `With` constructor parameters contain the list of the values are that required by the Aggregate. All of these values must be supplied or the Aggregate is invalid and must throw a `DomainException` when building.

In addition to required properties, the Aggregate may also have optional parameters. These are defined outside of the constructor.

In our example of an `Applicant` Aggregate, it must have a first and last name, along with a userId. The optional parameters are `MiddleName` and `Uid`.

The `With` class inherits from the `FluentBuilder` which provides functionality for accessing the `Instance`, adding a `DomainException`, and building the Aggregate while validating the build process.

```
/// <summary>
/// Base class for the 'With' pattern fluent syntax builder support for building Aggregates.
/// </summary>
/// <typeparam name="T">The object type returned from the builder.</typeparam>
public abstract class FluentBuilder<T>
    where T : class, new()
{
    private readonly List<string> domainExceptions = new();

    protected FluentBuilder()
    {
        Instance = new T();
    }

    protected T Instance { get; }

    protected void AddDomainException(string exceptionMessage)
    {
        domainExceptions.Add(exceptionMessage);
    }

    public T Build()
    {
        Validate();
        return Instance;
    }

    private void Validate()
    {
        if (domainExceptions.IsNullOrEmpty())
        {
            return;
        }

        StringBuilder exceptionMessage = new();
        domainExceptions.ForEachItem(x => { exceptionMessage.Append($"{x}. "); });

        throw new DomainException(exceptionMessage.ToString().TrimEnd());
    }
}
```

Defining the fluent builder for the `Applicant` Aggregate would look similar to this.

```
public partial class Applicant
{
    public class With : FluentBuilder<Applicant>
    {
        public With(string firstName, string lastName, string userId)
        {
            if (firstName.IsNullOrEmpty())
            {
                AddDomainException("First name cannot be empty;");
            }

            if (lastName.IsNullOrEmpty())
            {
                AddDomainException("Last name cannot be empty");
            }

            if (userId.IsNullOrEmpty())
            {
                AddDomainException("User ID cannot be empty");
            }

            Instance.FirstName = firstName;
            Instance.LastName = lastName;
            Instance.UserId = userId;
        }

        public With MiddleName(string middleName)
        {
            Instance.MiddleName = middleName;
            return this;
        }

        public With Uid(string uid)
        {
            Instance.Uid = uid;
            return this;
        }
    }
}
```

#### Aggregate Operations

The Operations are the location for all business logic related to the bounded context and the aggregates.

If we want to add an `Applicant` to the `LoanApplication` and ensure that this only succeeds if the `Status` is `New`, then we would have something that looks like.

```
    public async Task<Applicant> AddApplicantAsync(Applicant applicant)
    {
        if (Status != LoanApplicationStatus.New)
        {
            throw new DomainException("Application can only be updated before submission");
        }

        try
        {
            _applicants.Add(applicant);
        }
        catch (Exception exception)
        {
            throw new DomainException("Unable to add applicant");
        }
        
        return await DomainService.AddApplicant(this, applicant);
    }
```

When creating new objects that are part of the collection composed by a Root Aggregate and we want to return the newly created item in the collection, then we need to extend the default functionality of the `IDomainService`. This piece is necessary given that the unique ID for the create object comes from the Entity definition, as we are using Mongo with this example. We have done that by adding a method to the `ILoanApplicationDomainService` interface with the following definition.

```
Task<Applicant> AddApplicant(LoanApplication application, Applicant applicant);
```

In the `LoanApplicationDomainService`, we do this by converting the Aggregate to an Entity, saving the Entity, then converting the Entity to an Aggregate and returning it.

```
    public async Task<Aggregates.Applicant> AddApplicant(Aggregates.LoanApplication application, Aggregates.Applicant applicant)
    {
        var entity = applicant.ToEntity();
        await UpdateAsync(application);
        return entity.ToAggregate();
    }
```
