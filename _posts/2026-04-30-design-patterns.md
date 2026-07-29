---
layout: post
title: Design Patterns I Use in Backend Java
date: 2026-04-30 10:59:00-0400
description: A practical overview of core design patterns in backend Java with simple examples.
tags:
categories:
giscus_comments: false
related_posts: false
toc:
  beginning: true
---

## Introduction

Design patterns are reusable solutions to common software design problems. They are not rules, and they are not a shortcut to writing "better" code by default. Their value is more practical than that: they give you a way to express collaboration, object creation, and communication clearly.

The best patterns usually solve one of these problems:

- One class should exist only once.
- Object creation should be centralized.
- Many objects need to react to one event.
- Behavior should be added without changing the base class.
- Communication between objects is becoming tangled.
- A complex subsystem needs one simple entry point.

In this article, I will use very simple examples:

- A logger for Singleton
- Email and SMS notifications for Factory Method
- News subscribers for Observer
- Coffee with milk for Decorator
- A chat room for Mediator
- A home cinema for Facade

I am intentionally keeping the examples small. That makes the pattern easier to understand and easier to apply correctly in real code.

## What To Look For Before Using A Pattern

Before adding a design pattern, I ask a few questions:

- Is the problem real, or am I designing for a future that may never come?
- Will this make the code easier to read and test?
- Does the pattern have a single clear responsibility?
- Will the abstraction still feel natural if the system grows?

If the answer is no, I usually keep the code simpler.

## 1. Singleton Pattern

The Singleton Pattern ensures that a class has only one instance and provides a global access point to it.

Use it when:

- Only one shared instance should exist.
- You need centralized access to configuration, logging, caching, or application state.

Use it carefully. Excessive use of Singleton can create hidden dependencies and make testing more difficult.

### Example

This logger is shared by the whole application:

```java
public class Logger {
    private static final Logger INSTANCE = new Logger();

    private Logger() {
    }

    public static Logger getInstance() {
        return INSTANCE;
    }

    public void log(String message) {
        System.out.println(message);
    }
}
```

```java
Logger logger = Logger.getInstance();
logger.log("Application started");
```

### Why It Works

The main benefit is control:

- One instance.
- One global access point.
- One place to route shared behavior.

That is useful when the object really is a shared service or shared state holder.

### What To Watch Out For

- Singleton can hide dependencies, because any class can reach for it directly.
- It can make unit tests harder if the singleton holds mutable state.
- It becomes risky if it grows into a global dumping ground.

If you are using Spring, remember that singleton scope is already the default for many beans. You often do not 
need to implement the pattern manually.

## 2. Factory Method Pattern

The Factory Method Pattern defines a method for creating objects while allowing subclasses to decide which object 
type to create.

Use it when:

- Object creation should be separated from the code that uses the object.
- Subclasses need to control which object is created.
- New product types may be added later.

This is useful when the caller should not need to know the concrete class.

### Example

Here we model notifications that can be sent by email or SMS:

```java
interface Notification {
    void send(String message);
}
```

```java
class EmailNotification implements Notification {
    public void send(String message) {
        System.out.println("Email: " + message);
    }
}
```

```java
class SmsNotification implements Notification {
    public void send(String message) {
        System.out.println("SMS: " + message);
    }
}
```

```java
abstract class NotificationCreator {
    public abstract Notification createNotification();

    public void notifyUser(String message) {
        Notification notification = createNotification();
        notification.send(message);
    }
}
```

```java
class EmailNotificationCreator extends NotificationCreator {
    public Notification createNotification() {
        return new EmailNotification();
    }
}
```

```java
NotificationCreator creator = new EmailNotificationCreator();
creator.notifyUser("Your order has shipped.");
```

### Why It Works

The creator knows the workflow, but not the concrete implementation details.

That gives you:

- Creation logic in one place.
- Easier substitution of new notification types.
- Less branching in the caller.

### What To Watch Out For

- Do not turn the factory into a large `switch` statement if the main purpose is still just object creation.
- If object creation is trivial and never changes, a direct constructor may be enough.
- Keep the abstraction honest: use it because object creation varies, not because interfaces feel sophisticated.

## 3. Observer Pattern

The Observer Pattern creates a one-to-many relationship between objects. When the subject changes, all registered 
observers are notified automatically.

Use it when:

- Multiple objects must react to the same event.
- You are building event-driven systems.
- The subject should not depend directly on its listeners.

This pattern is a natural fit when updates need to fan out to several consumers.

### Example

A news publisher notifies subscribers when a new article becomes available:

```java
interface Observer {
    void update(String message);
}
```

```java
class EmailSubscriber implements Observer {
    public void update(String message) {
        System.out.println("Email received: " + message);
    }
}
```

```java
import java.util.ArrayList;
import java.util.List;

class NewsPublisher {
    private final List<Observer> observers = new ArrayList<>();

    public void subscribe(Observer observer) {
        observers.add(observer);
    }

    public void publish(String news) {
        for (Observer observer : observers) {
            observer.update(news);
        }
    }
}
```

```java
NewsPublisher publisher = new NewsPublisher();
publisher.subscribe(new EmailSubscriber());

publisher.publish("A new article is available.");
```

### Why It Works

The publisher does not need to know who is listening.
It just sends the update and lets observers react.

That gives you:

- Loose coupling between the source and the listeners.
- Easy addition of new subscribers.
- A clean way to model change propagation.

### What To Watch Out For

- Too many observers can make behavior harder to trace.
- Notification order may matter, so be deliberate.
- If observers are long-lived, make sure they can unsubscribe to avoid leaks.

## 4. Decorator Pattern

The Decorator Pattern adds new behavior to an object without changing its original class.

Use it when:

- Features need to be added dynamically.
- Inheritance would create too many subclasses.
- Several optional behaviors may be combined.

This pattern is ideal when the base object is simple, but extra features should be layered on top.

### Example

A basic coffee can be decorated with milk:

```java
interface Coffee {
    String getDescription();
    double getPrice();
}
```

```java
class BasicCoffee implements Coffee {
    public String getDescription() {
        return "Coffee";
    }

    public double getPrice() {
        return 2.00;
    }
}
```

```java
class MilkDecorator implements Coffee {
    private final Coffee coffee;

    public MilkDecorator(Coffee coffee) {
        this.coffee = coffee;
    }

    public String getDescription() {
        return coffee.getDescription() + ", milk";
    }

    public double getPrice() {
        return coffee.getPrice() + 0.50;
    }
}
```

```java
Coffee coffee = new MilkDecorator(new BasicCoffee());

System.out.println(coffee.getDescription()); // Coffee, milk
System.out.println(coffee.getPrice());       // 2.50
```

### Why It Works

The base object stays untouched.
Extra features are composed around it.

That gives you:

- Flexible combination of behaviors.
- No explosion of subclasses.
- Reusable wrappers that stay focused.

### What To Watch Out For

- Decorators should preserve the same interface as the wrapped object.
- Deep chains of decorators can become hard to read if overused.
- If the "extra behavior" is really a separate business process, another pattern may fit better.

## 5. Mediator Pattern

The Mediator Pattern introduces a central object that manages communication between related objects.

Instead of objects communicating directly with each other, they communicate through the mediator.

Use it when:

- Many objects depend directly on one another.
- Communication logic has become difficult to manage.
- You want to reduce coupling between components.

This is a good pattern when interaction rules matter more than direct object references.

### Example

A chat room coordinates users:

```java
interface ChatMediator {
    void sendMessage(String message, User sender);
}
```

```java
import java.util.ArrayList;
import java.util.List;

class ChatRoom implements ChatMediator {
    private final List<User> users = new ArrayList<>();

    public void addUser(User user) {
        users.add(user);
    }

    public void sendMessage(String message, User sender) {
        for (User user : users) {
            if (user != sender) {
                user.receive(message);
            }
        }
    }
}
```

```java
class User {
    private final String name;
    private final ChatMediator mediator;

    public User(String name, ChatMediator mediator) {
        this.name = name;
        this.mediator = mediator;
    }

    public void send(String message) {
        mediator.sendMessage(message, this);
    }

    public void receive(String message) {
        System.out.println(name + " received: " + message);
    }
}
```

```java
ChatRoom room = new ChatRoom();

User alice = new User("Alice", room);
User bob = new User("Bob", room);

room.addUser(alice);
room.addUser(bob);

alice.send("Hello everyone");
```

### Why It Works

The mediator handles the communication policy.
The users stay focused on sending and receiving messages.

That gives you:

- Less direct coupling.
- One place to manage interaction rules.
- Easier changes to communication behavior.

### What To Watch Out For

- A mediator can become a "god object" if it starts doing too much.
- Keep it focused on coordination, not on all business logic.
- If the communication is simple, direct calls may be better.

## 6. Facade Pattern

The Facade Pattern provides a simple interface to a complex group of classes or subsystems.

Use it when:

- A system has many complicated components.
- Clients only need a small number of common operations.
- You want to hide implementation details.

This is one of the most useful patterns when a workflow touches several subsystems.

### Example

A home cinema can be started with one facade instead of several separate calls:

```java
class Projector {
    public void turnOn() {
        System.out.println("Projector on");
    }
}
```

```java
class SoundSystem {
    public void turnOn() {
        System.out.println("Sound system on");
    }
}
```

```java
class StreamingService {
    public void playMovie() {
        System.out.println("Movie started");
    }
}
```

```java
class HomeCinemaFacade {
    private final Projector projector = new Projector();
    private final SoundSystem soundSystem = new SoundSystem();
    private final StreamingService streamingService = new StreamingService();

    public void watchMovie() {
        projector.turnOn();
        soundSystem.turnOn();
        streamingService.playMovie();
    }
}
```

```java
HomeCinemaFacade cinema = new HomeCinemaFacade();
cinema.watchMovie();
```

### Why It Works

The facade gives the caller one entry point.
It hides the internal sequence and the subsystem details.

That gives you:

- A cleaner API.
- Less knowledge required from the caller.
- A natural place for common orchestration.

### What To Watch Out For

- A facade should simplify, not become another layer of business rules.
- If it gets too large, split it into smaller facades.
- If callers still need all the internals, the facade is probably not doing enough.

## How These Patterns Compare

- **Singleton**: one shared instance.
- **Factory Method**: create the right object through subclass-controlled creation.
- **Observer**: notify multiple listeners when one thing changes.
- **Decorator**: add optional behavior without changing the base object.
- **Mediator**: centralize communication between related objects.
- **Facade**: provide one simple entry point to a complex subsystem.

## Common Mistakes

- Using Singleton for convenience instead of necessity.
- Creating factories before there is more than one real variant.
- Letting observers become hidden business logic with side effects everywhere.
- Stacking decorators so deeply that the object graph becomes unreadable.
- Turning the mediator into a dumping ground.
- Writing a facade that is just a thin pass-through with no real simplification.

## Final Guidance

- Use **Singleton** when one shared instance is the right model.
- Use **Factory Method** when object creation depends on type-specific logic.
- Use **Observer** when several parts must react to the same event.
- Use **Decorator** when behavior should be composable.
- Use **Mediator** when direct communication is getting out of hand.
- Use **Facade** when a complex workflow needs one clear entry point.

That is what makes the difference between pattern knowledge and pattern expertise: not using every pattern 
everywhere, but using the right one for the problem in front of you.
