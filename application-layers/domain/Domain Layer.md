The heart of the Bounded Context is the Domain layer. Within the Domain are several different sets of objects.

- [Aggregates](./aggregates.md)
- [ValueObjects](./value-objects.md)
- [Entities](./entities.md)
- [Extensions](./extensions.md)
- [Repositories](./repositories.md)
- [Services](./services.md)

The Domain layer consists of Aggregates and Value Objects.
Cases in which the Aggregate is persisted to a data store that is something other than a json document store, the definition of the Entity model is also required.


### Aggregate Structure

The Aggregate is broken into three separate partial classes, each with its own specific purpose. The main part of the Aggregate is the file name containing just the name of the Aggregate. In addition to that file, there is an Operations file and a With file.

The operations file contains all of the functions are that are accessible on the aggregate. 

**Important: ONLY the Root Aggregate should publicly expose the functions on the Aggregate. All other composed aggregates within the Root Aggregate must have `internal` accessibility only**

