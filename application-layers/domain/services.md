Domain services are used when a transaction affects more than one Root Aggregate.

In the case of submitting and processing a payment for an `Order`, both a `PaymentDetails` and an `Order` Root Aggregate perform operations in the single transaction.

## Submitting a Payment
In order for the Point of Sale Bounded Context to process a payment it must read the information of the `Order` and validate the transaction request.

The `PaymentDetails` contains the `OrderId` and `CorrelationId` for observability purposes. It does not contain the full `Order`. The concern of the `PaymentDetails` is to capture the payment information only. Given the separation of concerns, the `PaymentService` retrieves the `Order` and performs business rules before submitting the `PaymentDetails`

```
    public async Task<PaymentDetails?> SubmitPayment(PaymentDetails paymentDetails, CancellationToken ct)
    {
        var order = await _orderRepository.GetOrderByIdAsync(paymentDetails.OrderId, ct);

        if (order == null)
        {
            return null;
        }

        if (order.Status != OrderStatus.Created)
        {
            throw new DomainException($"Cannot submit payment for an order with a status other than {OrderStatus.Created}.");
        }

        if (!order.LineItems.Any())
        {
            throw new DomainException("Cannot submit payment for an order with no items.");
        }

        return await paymentDetails.SubmitPaymentAsync(ct);
    }
```

## Processing a Payment Result

The `PaymentDetails` Root Aggregate from the Point of Sale Bounded Context publishes a message to indicate that a payment is submitted. The PaymentProcessing PBC listens for messages related to payment submissions, processes them, then publishes the result of the payment processing to the distributed messaging system.

The `PaymentProcessMessageHandler` uses the `IHandleMessageOf<T>` interface define in `Platform.Blocks.Messaging`

The message handler also acts an an ACL between the different PBC/Bounded Contexts in the Coffee Shop ecosystem.

The handler interprets the `PaymentStatus` coming from outside of the Bounded Context and translates it to the proper `OrderStatus` (please disregard the code smell of the nested ternary, I already had a talk with myself about writing it).

```
public class PaymentProcessedMessageHandler : IHandleMessageOf<PaymentProcessedMessage>
{
    private readonly IPaymentDetailsReadOnlyRepository _repository;
    private readonly IPaymentService _paymentService;
    private readonly ILogger<PaymentProcessedMessageHandler> _logger;

    public PaymentProcessedMessageHandler(
	    IPaymentDetailsReadOnlyRepository repository,
	    IPaymentService paymentService,
	    ILogger<PaymentProcessedMessageHandler> logger)
    {
        _repository = repository;
        _paymentService = paymentService;
        _logger = logger;
    }

    public async Task HandleAsync(PaymentProcessedMessage message, CancellationToken ct = default)
    {
        var paymentDetails = await _repository.GetByIdAsync(message.PaymentDetailsId, ct);

        if (paymentDetails == null)
        {
	        var errorMessage = $"PaymentDetails not found for paymentDetailsId {message.PaymentDetailsId}.";
	        _logger.LogError(message.CorrelationId, null, errorMessage);
            throw new InvalidOperationException(errorMessage);
        }

        var orderStatus = message.Status == PaymentStatus.Declined
	        ? OrderStatus.Created
	        : message.Status == PaymentStatus.Approved
		        ? OrderStatus.Paid
		        : OrderStatus.Unknown;
        
        await _paymentService.ProcessPaymentResult(paymentDetails, orderStatus, ct);
    }
}
```

The handler performs the necessary logic then calls the `PaymentService` to process the payment processing request.

```
    public async Task ProcessPaymentResult(PaymentDetails paymentDetails, OrderStatus orderStatus, CancellationToken ct)
    {
        var order = await _orderRepository.GetOrderByIdAsync(paymentDetails.OrderId, ct)
            ?? throw new InvalidOperationException("Order not found for payment processing.");

        await order.UpdateOrderStatus(orderStatus, ct);
    }
```