---
layout: post
title: Design Patterns I Use in Backend Java
date: 2026-02-25 10:59:00-0400
description: Practical backend Java patterns focused on reusability and maintainability.
tags:
categories:
giscus_comments: false
related_posts: false
toc:
  beginning: true
---
<span style="margin-left: 10px;"></span>

## Introduction

<span style="margin-left: 10px;"></span>
For me, design patterns in backend Java are not about making the code look “advanced”. They are about solving 
practical problems that appear over and over again in real backend systems:
- How do I avoid repeating the same conditional logic everywhere?
- How do I add new behavior without rewriting existing code?
- How do I keep business logic away from persistence details?
- How do I make code easier to test?
- How do I extract reusable components without creating a generic mess?

<span style="margin-left: 10px;"></span>
That is where design patterns are still useful. Not as theory or as decoration, but as tools I can actually use.

<span style="margin-left: 10px;"></span>
In backend Java, especially in applications built with Spring Boot, patterns like Strategy, Factory, and 
Repository appear naturally. You notice that the code is becoming harder to change, and then you introduce 
a structure that makes the next change easier. That is the practical value of design patterns.

<span style="margin-left: 10px;"></span>
Imagine we are building a backend system for an online beer shop.
Customers can browse beers, place orders, pay using different providers, and receive notifications when 
their order is confirmed. Something like this:
- Customers browse beers
- Customers place beer orders
- Payments can be made through different providers
- Notifications can be sent through different channels
- Beer data is stored in a database
- Shared logic is reused across services

<span style="margin-left: 10px;"></span>
At the beginning, the system might look simple. Maybe we only support one payment provider.

```java
public class BeerOrderService {

    public void placeOrder(BeerOrder order) {
        chargeWithStripe(order);
        sendEmailConfirmation(order);
        saveOrder(order);
    }

    private void chargeWithStripe(BeerOrder order) {
        System.out.println("Charging beer order with Stripe");
    }

    private void sendEmailConfirmation(BeerOrder order) {
        System.out.println("Sending beer order confirmation email");
    }

    private void saveOrder(BeerOrder order) {
        System.out.println("Saving beer order");
    }
}
```

<span style="margin-left: 10px;"></span>
This is not necessarily bad code, for a small application, it might be the perfect solution. The problem 
starts when the requirements change (oh and they do!). A few weeks later, the business wants to support PayPal
or credits. Then email notifications are not enough, so we add SMS. Then different beer types need different 
validation rules, for example for alcohol content. Suddenly, the original service starts growing and we find
ourselves with the great wall of if-else statements. 

<span style="margin-left: 10px;"></span>
I mean, it works, but the service now "knows" too much. It knows about payment providers, 
notification channels, order persistence, and business flow. Every time we add a new payment provider or 
notification channel, we have to modify this class.
That means the class becomes harder to test, harder to maintain, and easier to break!

<span style="margin-left: 10px;"></span>
For example, adding a new payment provider like BEER_WALLET means opening BeerOrderService again and changing the if/else chain.

```java
if ("BEER_WALLET".equals(paymentProvider)) {
    chargeWithBeerWallet(order);
}
```

<span style="margin-left: 10px;"></span>
In a real backend system, this pattern repeats everywhere. The same provider checks might appear in the order service, 
refund service, invoice service, payment audit service, and customer support service.
That is how duplication spreads and once duplication spreads, the codebase becomes expensive to change.

<span style="margin-left: 10px;"></span>
This is where design patterns become useful, where the goal is not to “apply patterns” just because it is how others say it should be done.
The goal is to move responsibilities into the right places.
Instead of one service knowing every payment implementation, we can use the Strategy Pattern and model each payment provider as a separate strategy.
Instead of scattering notification creation logic across the application, we can use the Factory Pattern to resolve the correct notification handler.
Instead of mixing database access with business logic, we can use the Repository Pattern to isolate persistence concerns.
And the result is code that can grow independently.

<span style="margin-left: 10px;"></span>
In this article, I will focus on three design patterns I regularly use in backend Java development.

1. Strategy Pattern - Used when the application needs to choose between different behaviors.
2. Factory Pattern - Used when the application needs to create or resolve the right implementation.
3. Repository Pattern - Used when the application needs a clean boundary around database access.

<span style="margin-left: 10px;"></span>
One important thing before going further, design patterns should usually appear as a response to a real problem.
I do not start a backend service by creating ten interfaces because “maybe one day” there will be multiple implementations.
For example, if the beer shop only supports Stripe and there is no plan to support anything else, this might be enough:

```java
@Service
public class BeerPaymentService {

    public PaymentResult pay(BeerOrder order) {
        System.out.println("Charging beer order with Stripe");

        return new PaymentResult(
            "stripe-transaction-id",
            PaymentStatus.SUCCESS
        );
    }
}
```

<span style="margin-left: 10px;"></span>
This is simple and enough and the same idea applies to reusable components. 

<span style="margin-left: 10px;"></span>
Reusability is valuable, but only when we extract the right things.
For example, this kind of duplicated beer price calculation is a good candidate for reuse:

```java
BigDecimal total = beer.price()
    .multiply(BigDecimal.valueOf(quantity))
    .multiply(BigDecimal.ONE.add(taxRate));
```

<span style="margin-left: 10px;"></span>
If this logic appears in the order service, invoice service, refund service, and reporting service, then extracting it makes sense.

```java
public final class BeerPriceCalculator {

    private BeerPriceCalculator() {
    }

    public static BigDecimal calculateTotal(
        BigDecimal unitPrice,
        int quantity,
        BigDecimal taxRate
    ) {
        return unitPrice
            .multiply(BigDecimal.valueOf(quantity))
            .multiply(BigDecimal.ONE.add(taxRate));
    }
}
```

<span style="margin-left: 10px;"></span>
Now the logic has one home and that is useful reuse.

<span style="margin-left: 10px;"></span>
A good use of a pattern should make the code easier to read once you understand the domain. For example:

```java
PaymentStrategy paymentStrategy =
paymentStrategyResolver.resolve(order.paymentProvider());

PaymentResult result = paymentStrategy.pay(order);
```

<span style="margin-left: 10px;"></span>
The order service does not need to know how Stripe works, how PayPal works, or how credits are validated. It only needs 
to know that the selected payment strategy can process the beer order. That is a useful abstraction. The same applies here:

```java
BeerNotificationHandler handler =
notificationFactory.getHandler(order.notificationChannel());

handler.send(order);
```

<span style="margin-left: 10px;"></span>
The service does not care whether the notification is sent by email or SMS. It asks for the right handler and delegates the work.
The abstraction has a purpose it keeps the main business flow clean.

<span style="margin-left: 10px;"></span>
So the main idea of this article is simple:

- The Strategy Pattern helps us switch between different behaviors without filling services with conditional logic.
- The Factory Pattern helps us centralize the creation or resolution of the right implementation.
- The Repository Pattern helps us keep database access separate from business rules.
- And reusable components help us avoid repeating stable logic across different parts of the system.

<span style="margin-left: 10px;"></span>
None of these ideas are new.

## Example Domain

<span style="margin-left: 10px;"></span>
Before jumping into the patterns, I want to define the domain we will use throughout the article.
This matters because design patterns are easier to understand when they are connected to one consistent example.

<span style="margin-left: 10px;"></span>
Imagine a backend system for a craft beer shop.
This shop sells beers from different breweries. Customers can search the catalog, add beers to an order, pay for the 
order, and receive a confirmation.

<span style="margin-left: 10px;"></span>
The important thing is that this system has several areas where design patterns naturally appear. For example:
- Payment providers vary.
- Notification channels vary.
- Database access should be isolated.
- Shared calculations should not be duplicated.

<span style="margin-left: 10px;"></span>
That gives us a practical reason to use patterns like Strategy, Factory, and Repository.
We are not adding patterns because the code needs to look sophisticated.
We are adding them because the backend has real variation points.

### Main Domain Objects

```java

import java.math.BigDecimal;

public record Beer(
    Long id,
    String name,
    String brewery,
    BeerType type,
    BigDecimal alcoholPercentage,
    BigDecimal price
) {
}

public enum BeerType {
    IPA,
    STOUT,
    LAGER,
    PILSNER,
    SOUR,
    WHEAT,
    PORTER
}
```

```java
import java.math.BigDecimal;
import java.util.List;

public record BeerOrder(
    Long id,
    Long customerId,
    List<BeerOrderItem> items,
    BigDecimal totalAmount,
    BeerOrderStatus status,
    PaymentProvider paymentProvider,
    NotificationChannel notificationChannel
) {
}
```

```java
import java.math.BigDecimal;

public record BeerOrderItem(
    Long beerId,
    String beerName,
    int quantity,
    BigDecimal unitPrice
) {
}
```

```java
public enum BeerOrderStatus {
    CREATED,
    PAID,
    PAYMENT_FAILED,
    CONFIRMED,
    CANCELLED
}
```


```java
public enum PaymentProvider {
    STRIPE,
    PAYPAL,
    BREWERY_CREDIT,
    CASH_ON_DELIVERY
}
```

```java
public record PaymentResult(
    String transactionId,
    PaymentStatus status,
    String message
) {
    public static PaymentResult success(String transactionId) {
        return new PaymentResult(
            transactionId,
            PaymentStatus.SUCCESS,
            "Payment processed successfully"
        );
    }

    public static PaymentResult failed(String message) {
        return new PaymentResult(
            null,
            PaymentStatus.FAILED,
            message
        );
    }
}
```


```java
public enum PaymentStatus {
    SUCCESS,
    FAILED
}
```

```java
public enum NotificationChannel {
    EMAIL,
    SMS,
    PUSH,
}
```

```java
public record Customer(
    Long id,
    String name,
    String email,
    String phoneNumber
) {
}
```


```java
import java.util.List;

public record BeerOrderRequest(
    Long customerId,
    List<BeerOrderItemRequest> items,
    PaymentProvider paymentProvider,
    NotificationChannel notificationChannel
) {
}
```

```java
public record BeerOrderItemRequest(
    Long beerId,
    int quantity
) {
}
```

## 3. Strategy Pattern — Handling Different Beer Payment Providers

<span style="margin-left: 10px;"></span>
The Strategy Pattern is useful when we have multiple ways to do something and we need to pick one at runtime.
In our beer shop, customers can pay using Stripe, PayPal, brewery credits, or cash on delivery.
Without the Strategy Pattern, the order service would need to know how to handle each payment method.

<span style="margin-left: 10px;"></span>
Let me show the problematic version first. Without Strategy, we might end up with something like this:

```java
@Service
public class BeerOrderServiceBeforeStrategy {

    public void placeOrder(BeerOrder order) {
        if (PaymentProvider.STRIPE.equals(order.paymentProvider())) {
            processStripePayment(order);
        } else if (PaymentProvider.PAYPAL.equals(order.paymentProvider())) {
            processPayPalPayment(order);
        } else if (PaymentProvider.BREWERY_CREDIT.equals(order.paymentProvider())) {
            processBreweryCreditPayment(order);
        } else if (PaymentProvider.CASH_ON_DELIVERY.equals(order.paymentProvider())) {
            processCashOnDelivery(order);
        }
        saveOrder(order);
    }

    private void processStripePayment(BeerOrder order) {
        //stripe-specific logic
    }

    private void processPayPalPayment(BeerOrder order) {
        //PayPal-specific logic
    }

    private void processBreweryCreditPayment(BeerOrder order) {
        //brewery credit-specific logic
    }

    private void processCashOnDelivery(BeerOrder order) {
        // cash on delivery-specific logic
    }

    private void saveOrder(BeerOrder order) {
        //save order
    }
}
```

<span style="margin-left: 10px;"></span>
This works, but the problem is obvious. Every payment provider is mixed into one service. The class knows too much about payment details.
When we need to add a new provider or change Stripe logic, we have to open this service and modify it.
We also have to test all the if-else branches and every payment logic is mixed together.

<span style="margin-left: 10px;"></span>
The Strategy Pattern solves this by extracting each payment method into its own strategy.

```java
public interface PaymentStrategy {
    PaymentProvider provider();
    PaymentResult pay(BeerOrder order);
}
```

<span style="margin-left: 10px;"></span>
Now each payment provider has its own implementation:

```java
@Service
public class StripePaymentStrategy implements PaymentStrategy {

    @Override
    public PaymentProvider provider() {
        return PaymentProvider.STRIPE;
    }

    @Override
    public PaymentResult pay(BeerOrder order) {
        System.out.println("Processing payment with Stripe for order " + order.id());
        return PaymentResult.success("stripe-" + order.id());
    }
}
```

```java
@Service
public class PayPalPaymentStrategy implements PaymentStrategy {

    @Override
    public PaymentProvider provider() {
        return PaymentProvider.PAYPAL;
    }

    @Override
    public PaymentResult pay(BeerOrder order) {
        System.out.println("Processing payment with PayPal for order " + order.id());
        return PaymentResult.success("paypal-" + order.id());
    }
}
```

```java
@Service
public class BreweryCreditPaymentStrategy implements PaymentStrategy {

    private final BreweryCreditService creditService;

    public BreweryCreditPaymentStrategy(BreweryCreditService creditService) {
        this.creditService = creditService;
    }

    @Override
    public PaymentProvider provider() {
        return PaymentProvider.BREWERY_CREDIT;
    }

    @Override
    public PaymentResult pay(BeerOrder order) {
        boolean debited = creditService.debit(order.customerId(), order.totalAmount());
        if (debited) {
            return PaymentResult.success("credit-" + order.id());
        }
        return PaymentResult.failed("Insufficient brewery credits");
    }
}
```

<span style="margin-left: 10px;"></span>
Now we need a component that resolves the right strategy based on the payment provider:

```java
@Component
public class PaymentStrategyResolver {

    private final Map<PaymentProvider, PaymentStrategy> strategies;

    public PaymentStrategyResolver(List<PaymentStrategy> strategyList) {
        this.strategies = strategyList.stream()
            .collect(Collectors.toMap(
                PaymentStrategy::provider,
                Function.identity()
            ));
    }

    public PaymentStrategy resolve(PaymentProvider provider) {
        return Optional.ofNullable(strategies.get(provider))
            .orElseThrow(() -> new IllegalArgumentException(
                "Unsupported payment provider: " + provider
            ));
    }
}
```

<span style="margin-left: 10px;"></span>
The magic here is that Spring automatically injects all implementations of `PaymentStrategy` as a List.
We then build a map from provider to strategy. In my opinion, this is extremely clean.

<span style="margin-left: 10px;"></span>
Now the order service becomes much simpler:

```java
@Service
public class BeerOrderServiceAfterStrategy {

    private final PaymentStrategyResolver paymentStrategyResolver;

    public BeerOrderServiceAfterStrategy(PaymentStrategyResolver paymentStrategyResolver) {
        this.paymentStrategyResolver = paymentStrategyResolver;
    }

    public void placeOrder(BeerOrder order) {
        PaymentStrategy paymentStrategy = paymentStrategyResolver.resolve(order.paymentProvider());
        PaymentResult result = paymentStrategy.pay(order);

        if (PaymentStatus.SUCCESS.equals(result.status())) {
            order = order.withStatus(BeerOrderStatus.PAID);
        } else {
            order = order.withStatus(BeerOrderStatus.PAYMENT_FAILED);
        }

        saveOrder(order);
    }

    private void saveOrder(BeerOrder order) {
        //save order
    }
}
```

<span style="margin-left: 10px;"></span>
The service no longer knows how Stripe works, how PayPal works, or how brewery credits work.
It just asks for the right strategy and delegates. If we need to add a new payment provider like Apple Pay, we create a 
new strategy and register it with Spring.
The order service does not change at all. 

<span style="margin-left: 10px;"></span>
The benefits are clear:
- Each strategy is isolated and testable in isolation.
- Adding a new payment provider does not require changing existing code.
- The order service is simpler and focused on business flow, not payment details.
- Strategies can depend on different services (like `BreweryCreditService`) without bloating the order service.

## 4. Factory Pattern — Creating Beer Notification Handlers

<span style="margin-left: 10px;"></span>
The Factory Pattern is about centralizing object creation or resolution.
In the beer shop, after an order is placed, we need to notify the customer.
But notifications can be sent through different channels: email, SMS or push notifications to the brewery team.

<span style="margin-left: 10px;"></span>
Without a factory, notification creation logic might be scattered across the codebase:

```java
@Service
public class BeerOrderServiceBeforeFactory {

    public void placeOrder(BeerOrder order) {
        //process order

        if (NotificationChannel.EMAIL.equals(order.notificationChannel())) {
            new EmailNotificationHandler().send(order);
        } else if (NotificationChannel.SMS.equals(order.notificationChannel())) {
            new SmsNotificationHandler().send(order);
        } else if (NotificationChannel.PUSH.equals(order.notificationChannel())) {
            new PushNotificationHandler().send(order);
        }
    }
}
```

<span style="margin-left: 10px;"></span>
This is not ideal. Object creation is mixed with business logic. If we add a new notification channel in the refund service, 
payment service, and shipping service, we have to duplicate these checks everywhere.

<span style="margin-left: 10px;"></span>
The Factory Pattern centralizes this. First, we define a common interface:

```java
public interface BeerNotificationHandler {
    NotificationChannel channel();
    void send(BeerOrder order);
}
```

<span style="margin-left: 10px;"></span>
Then we implement each notification channel:

```java
@Component
public class EmailBeerNotificationHandler implements BeerNotificationHandler {

    private final EmailService emailService;

    public EmailBeerNotificationHandler(EmailService emailService) {
        this.emailService = emailService;
    }

    @Override
    public NotificationChannel channel() {
        return NotificationChannel.EMAIL;
    }

    @Override
    public void send(BeerOrder order) {
        System.out.println("Sending order confirmation to " + order.customerId() + " via email");
        emailService.send(order.customerId(), "Your beer order is confirmed!");
    }
}
```

```java
@Component
public class SmsBeerNotificationHandler implements BeerNotificationHandler {

    private final SmsService smsService;

    public SmsBeerNotificationHandler(SmsService smsService) {
        this.smsService = smsService;
    }

    @Override
    public NotificationChannel channel() {
        return NotificationChannel.SMS;
    }

    @Override
    public void send(BeerOrder order) {
        System.out.println("Sending SMS confirmation to " + order.customerId());
        smsService.send(order.customerId(), "Beer order confirmed!");
    }
}
```

<span style="margin-left: 10px;"></span>
Now the factory:

```java
@Component
public class BeerNotificationFactory {

    private final Map<NotificationChannel, BeerNotificationHandler> handlers;

    public BeerNotificationFactory(List<BeerNotificationHandler> handlerList) {
        this.handlers = handlerList.stream()
            .collect(Collectors.toMap(
                BeerNotificationHandler::channel,
                Function.identity()
            ));
    }

    public BeerNotificationHandler getHandler(NotificationChannel channel) {
        return Optional.ofNullable(handlers.get(channel))
            .orElseThrow(() -> new IllegalArgumentException(
                "Unsupported notification channel: " + channel
            ));
    }
}
```

<span style="margin-left: 10px;"></span>
And now any service can use it:

```java
@Service
@RequiredArgsConstructor
public class BeerOrderServiceWithFactory {

    private final BeerNotificationFactory notificationFactory;

    public void placeOrder(BeerOrder order) {
        BeerNotificationHandler handler = notificationFactory.getHandler(order.notificationChannel());
        handler.send(order);
    }
}
```

<span style="margin-left: 10px;"></span>
This is much cleaner! The service does not know how to create or construct handlers. It just asks the factory for the right one.
The factory handles all the complexity like dependency injection, configuration, and error handling.

### Factory vs Strategy

<span style="margin-left: 10px;"></span>
Factory and Strategy patterns may look similar. They both have an interface with multiple implementations, 
and they both use a resolver or factory to pick the right one. But they solve different problems:

<span style="margin-left: 10px;"></span>
- **Strategy** is about choosing behavior. It answers the question: "How should I do this?" Multiple strategies 
represent different ways to accomplish the same goal. A payment strategy is still about paying, just using different methods.
- **Factory** is about creating or resolving objects. It answers the question: "Which object should I use?" The factory 
handles the construction, configuration, and lifecycle of objects.

<span style="margin-left: 10px;"></span>
In our example, the payment strategies all implement the same pay behavior but in different ways.
The notification factory creates different handler objects, each designed to work with a different channel.
The strategy is about the algorithm. The factory is about object creation.

## 5. Repository Pattern

<span style="margin-left: 10px;"></span>
The Repository Pattern is about creating a boundary between business logic and persistence logic.
In the backend, we need to query beers, find customers, retrieve orders, and manage stock.
Without the Repository Pattern, business services often mix this database access with business decisions.

<span style="margin-left: 10px;"></span>
The Repository Pattern says to create an interface that represents the data access layer.
The service depends on this interface, not on the database directly.

```java
public interface BeerRepository extends JpaRepository<BeerEntity, Long> {

    List<BeerEntity> findByType(BeerType type);

    Optional<BeerEntity> findByNameAndBrewery(String name, String brewery);

    List<BeerEntity> findAvailableBeers();
}
```

<span style="margin-left: 10px;"></span>
The interface is clean and focused on what data we need. The service does not need to know about SQL, JPA, or database schemas.
It just calls repository methods.

```java
@Service
public class BeerCatalogService {

    private final BeerRepository beerRepository;

    public BeerCatalogService(BeerRepository beerRepository) {
        this.beerRepository = beerRepository;
    }

    public List<Beer> findAvailableIpas() {
        return beerRepository.findByType(BeerType.IPA)
            .stream()
            .filter(BeerEntity::isAvailable)
            .map(BeerMapper::toDomain)
            .toList();
    }

    public Optional<Beer> findBeerByNameAndBrewery(String name, String brewery) {
        return beerRepository.findByNameAndBrewery(name, brewery)
            .map(BeerMapper::toDomain);
    }
}
```

<span style="margin-left: 10px;"></span>
Similarly for orders:

```java
public interface BeerOrderRepository extends JpaRepository<BeerOrderEntity, Long> {

    List<BeerOrderEntity> findByCustomerIdAndStatus(Long customerId, BeerOrderStatus status);

    Optional<BeerOrderEntity> findByIdAndCustomerId(Long orderId, Long customerId);
}
```

```java
@Service
public class BeerOrderQueryService {

    private final BeerOrderRepository orderRepository;

    public BeerOrderQueryService(BeerOrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    public List<BeerOrder> getCustomerOrders(Long customerId) {
        return orderRepository.findByCustomerIdAndStatus(customerId, BeerOrderStatus.PAID)
            .stream()
            .map(BeerOrderMapper::toDomain)
            .toList();
    }

    public BeerOrder getOrder(Long orderId, Long customerId) {
        return orderRepository.findByIdAndCustomerId(orderId, customerId)
            .map(BeerOrderMapper::toDomain)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }
}
```

<span style="margin-left: 10px;"></span>
The key benefit is testing. We can mock the repository:

```java
@Test
void testFindAvailableIpas() {
    BeerRepository mockRepository = mock(BeerRepository.class);
    BeerEntity ipaEntity = new BeerEntity(1L, "IPA Beer", "Brewery", BeerType.IPA, true);
    when(mockRepository.findByType(BeerType.IPA)).thenReturn(List.of(ipaEntity));

    BeerCatalogService service = new BeerCatalogService(mockRepository);
    List<Beer> result = service.findAvailableIpas();

    assertEquals(1, result.size());
    verify(mockRepository).findByType(BeerType.IPA);
}
```

<span style="margin-left: 10px;"></span>
The service is tested without touching a database. That is powerful.

### What Should Not Go Into a Repository

<span style="margin-left: 10px;"></span>
The Repository Pattern has a clear scope. It should only handle data access. It should not include:

<span style="margin-left: 10px;"></span>
- Payment logic: A repository should not process payments or call payment gateways. That belongs in a service or strategy.
- Notification logic: A repository should not send emails or SMS.
- Business decision rules: A repository should not decide if a discount applies or if an order is valid. That is business logic, not data access.
- API formatting: A repository should not transform data into JSON or API responses. That is the controller's job.

<span style="margin-left: 10px;"></span>
A repository is focused: it queries, saves, and updates data. Period. If you find yourself thinking "but I could also 
validate here" or "I could also call an external service here", stop. That belongs elsewhere.

## 6. Combining the Patterns in One Backend Flow

<span style="margin-left: 10px;"></span>
Now let's see how these patterns work together in a real backend scenario.
A customer places an order for a pack of beers. The backend needs to:
1. Load beer data from the repository
2. Validate stock availability
3. Process payment using the right strategy
4. Send a notification using the right handler
5. Save the order and return a result

<span style="margin-left: 10px;"></span>
Here is the service that orchestrates this flow:

```java
@Service
public class BeerOrderService {

    private final BeerRepository beerRepository;
    private final BeerOrderRepository orderRepository;
    private final PaymentStrategyResolver paymentStrategyResolver;
    private final BeerNotificationFactory notificationFactory;

    public BeerOrderService(
        BeerRepository beerRepository,
        BeerOrderRepository orderRepository,
        PaymentStrategyResolver paymentStrategyResolver,
        BeerNotificationFactory notificationFactory
    ) {
        this.beerRepository = beerRepository;
        this.orderRepository = orderRepository;
        this.paymentStrategyResolver = paymentStrategyResolver;
        this.notificationFactory = notificationFactory;
    }

    public BeerOrderResult placeOrder(BeerOrderRequest request) {
        //step 1 - load beers from repository and validate stock
        List<BeerEntity> beers = new ArrayList<>();
        for (BeerOrderItemRequest item : request.items()) {
            BeerEntity beer = beerRepository.findById(item.beerId())
                .orElseThrow(() -> new BeerNotFoundException(item.beerId()));

            if (beer.stock() < item.quantity()) {
                throw new InsufficientStockException(item.beerId());
            }

            beers.add(beer);
        }

        //step 2 - reate order entity
        BeerOrderEntity order = createOrderEntity(request, beers);

        //step 3 - process payment using strategy
        PaymentStrategy paymentStrategy = paymentStrategyResolver.resolve(request.paymentProvider());
        PaymentResult paymentResult = paymentStrategy.pay(BeerOrderMapper.toOrder(order));

        if (PaymentStatus.FAILED.equals(paymentResult.status())) {
            order.setStatus(BeerOrderStatus.PAYMENT_FAILED);
            orderRepository.save(order);
            throw new PaymentFailedException(paymentResult.message());
        }

        //step 4 - update order status and save
        order.setStatus(BeerOrderStatus.PAID);
        order.setTransactionId(paymentResult.transactionId());
        BeerOrderEntity savedOrder = orderRepository.save(order);

        //step 5 - send notification using factory
        BeerNotificationHandler handler = notificationFactory.getHandler(request.notificationChannel());
        handler.send(BeerOrderMapper.toOrder(savedOrder));

        return BeerOrderResult.success(paymentResult.transactionId(), savedOrder.getId());
    }

    private BeerOrderEntity createOrderEntity(BeerOrderRequest request, List<BeerEntity> beers) {
        BigDecimal total = beers.stream()
            .map(BeerEntity::getPrice)
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        return new BeerOrderEntity(
            request.customerId(),
            beers,
            total,
            BeerOrderStatus.CREATED,
            request.paymentProvider(),
            request.notificationChannel()
        );
    }
}
```

```java
public record BeerOrderResult(
    String transactionId,
    Long orderId,
    String message
) {
    public static BeerOrderResult success(String transactionId, Long orderId) {
        return new BeerOrderResult(transactionId, orderId, "Order placed successfully");
    }
}
```

<span style="margin-left: 10px;"></span>
Notice how clean this is. The service:
- Uses repositories to fetch data without knowing about SQL.
- Uses the payment strategy resolver to pick the right payment method without knowing how they work.
- Uses the notification factory to pick the right handler without knowing how to construct it.
- Focuses purely on business flow: validate, pay, notify, save.

<span style="margin-left: 10px;"></span>
Each pattern handles its concern. The repository handles data access.
The strategy handles payment behavior. The factory handles notification resolution.
The service orchestrates them. This is maintainable, testable, and easy to extend.

<span style="margin-left: 10px;"></span>
If we need to add Apple Pay, we create a new strategy and register it. The order service does not change.
If we need to add a new notification channel, we add a handler. The order service does not change.
If we need a new query on beers, we add a repository method. The service calls it without caring about the SQL.

## 7. Reusable Shared Libraries and Components

<span style="margin-left: 10px;"></span>
So far, we have talked about patterns within a single service, but we often have multiple services.
A beer platform might have:
- A catalog service
- An order service
- A payment service
- A notification service
- A reporting service

<span style="margin-left: 10px;"></span>
As these services grow, you start noticing repeated code:
- Beer validation (checking alcohol content, price, availability)
- Brewery formatting (normalizing brewery names)
- Money calculations (tax, discounts, totals)
- Order error handling (specific exceptions for order failures)
- Notification abstractions (sending to different channels)

<span style="margin-left: 10px;"></span>
When you see this repetition across multiple services, that is when you extract shared components.
In a multi-module Maven or Gradle project, you might structure it like this:

<span style="margin-left: 10px;"></span>
```
beer-platform/
├── beer-catalog-service/
│   ├── src/main/java/com/beer/catalog/
│   └── pom.xml
├── beer-order-service/
│   ├── src/main/java/com/beer/order/
│   └── pom.xml
├── beer-payment-service/
│   ├── src/main/java/com/beer/payment/
│   └── pom.xml
├── beer-notification-service/
│   ├── src/main/java/com/beer/notification/
│   └── pom.xml
└── beer-common/
    ├── src/main/java/com/beer/common/
    │   ├── validation/
    │   ├── exceptions/
    │   ├── money/
    │   ├── mappers/
    │   └── events/
    └── pom.xml
```

<span style="margin-left: 10px;"></span>
The `beer-common` module contains shared code that all services depend on. For example:

<span style="margin-left: 10px;"></span>
```java
public final class BeerValidator {

    private BeerValidator() {
    }

    public static void validateBeerPrice(BigDecimal price) {
        if (price.compareTo(BigDecimal.ZERO) <= 0) {
            throw new InvalidBeerException("Beer price must be greater than zero");
        }
    }

    public static void validateAlcoholContent(BigDecimal alcoholPercentage) {
        if (alcoholPercentage.compareTo(BigDecimal.ZERO) < 0 
            || alcoholPercentage.compareTo(new BigDecimal("100")) > 0) {
            throw new InvalidBeerException("Alcohol percentage must be between 0 and 100");
        }
    }
}
```

<span style="margin-left: 10px;"></span>
```java
public final class BeerPriceCalculator {

    private BeerPriceCalculator() {
    }

    public static BigDecimal calculateTotal(
        BigDecimal unitPrice,
        int quantity,
        BigDecimal taxRate
    ) {
        return unitPrice
            .multiply(BigDecimal.valueOf(quantity))
            .multiply(BigDecimal.ONE.add(taxRate));
    }

    public static BigDecimal applyDiscount(BigDecimal total, BigDecimal discountRate) {
        return total.multiply(BigDecimal.ONE.subtract(discountRate));
    }
}
```

<span style="margin-left: 10px;"></span>
```java
public class BeerNotFoundException extends RuntimeException {
    public BeerNotFoundException(Long beerId) {
        super("Beer with ID " + beerId + " not found");
    }
}

public class InsufficientStockException extends RuntimeException {
    public InsufficientStockException(Long beerId) {
        super("Insufficient stock for beer ID " + beerId);
    }
}
```

<span style="margin-left: 10px;"></span>
```java
public final class BeerOrderMapper {

    private BeerOrderMapper() {
    }

    public static BeerOrderRequest fromJson(String json) {
        // Parse JSON to BeerOrderRequest
        return null; // Simplified for this example
    }

    public static BeerOrder toDomain(BeerOrderEntity entity) {
        // Map entity to domain
        return null; // Simplified for this example
    }
}
```

<span style="margin-left: 10px;"></span>
Now all services can depend on `beer-common` and use these shared utilities:

<span style="margin-left: 10px;"></span>
```java
@Service
public class BeerOrderValidationService {

    public void validateOrderRequest(BeerOrderRequest request) {
        for (BeerOrderItemRequest item : request.items()) {
            BeerValidator.validateBeerPrice(item.getPrice());
        }
    }
}
```

```java
@Service
public class BeerCatalogService {

    public BigDecimal calculateOrderTotal(List<Beer> beers, BigDecimal taxRate) {
        return beers.stream()
            .map(beer -> BeerPriceCalculator.calculateTotal(
                beer.price(),
                1,
                taxRate
            ))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}
```

<span style="margin-left: 10px;"></span>
The key insight is that these shared components should be:
- Small and focused: Each component solves one problem.
- Stable: They should not change often. Once beer validation is settled, it should stay that way.
- Reused multiple times: If a component is used in only one service, it probably should not be shared yet.

### Reusable Does Not Mean Generic Too Early

<span style="margin-left: 10px;"></span>
A common mistake is creating "reusable" components before they are actually needed.
For example, do not create a generic "BeerCalculator" that handles everything related to beers and might be used "someday".

<span style="margin-left: 10px;"></span>
Instead, extract components when you see real duplication. If the beer price calculation appears in the order service, 
the invoice service, and the refund service, then extract `BeerPriceCalculator`. But if it only appears in the order 
service, leave it there for now. You are not saving time by extracting something that is not repeated yet.

<span style="margin-left: 10px;"></span>
The same applies to exceptions. Do not create a huge exception hierarchy "just in case". Create exceptions when you 
need them. `BeerNotFoundException` exists because multiple services need to handle the case when a beer is not found.
`InsufficientStockException` exists because the order service needs to tell the customer why an order failed.
These exceptions are useful abstractions because they are reused.

<span style="margin-left: 10px;"></span>
Shared components should emerge from real problems, not from theoretical planning.

## Conclusion

<span style="margin-left: 10px;"></span>
Design patterns in backend Java are not about following rules or making code look advanced.
They are about managing complexity as the system grows.

<span style="margin-left: 10px;"></span>
- Use the Strategy Pattern when you have multiple ways to do something (payment providers, notification channels). 
It lets you add new behaviors without modifying existing code.
- Use the Factory Pattern when object creation becomes scattered. It centralizes construction and dependency injection.
- Use the Repository Pattern to keep database access out of business logic. It makes services easier to test and data access easier to change.
- Extract reusable components when you see the same code repeated across multiple services. But only extract when duplication is real, not theoretical.

<span style="margin-left: 10px;"></span>
The goal is always the same, make the code easier to understand, easier to change, and easier to test.
If a pattern does not achieve that, skip it. Patterns are tools, not laws!
