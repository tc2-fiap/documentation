# Need to check if its worth if to implement:

Implement **MediatR** in this .NET 10 application for **in-process application events only**.

### Goal

I want to use MediatR as the equivalent of NestJS `EventEmitter2` for events that happen **inside single applications**.

Keep the implementation simple and appropriate for a student/local demonstration project.

---

## 1. Install MediatR

Add the appropriate MediatR NuGet package(s) for .NET 10.

Configure MediatR in `Program.cs` so that handlers from the application assemblies are automatically discovered.

---

## 2. Create an Application Event

Create an event called:

```csharp
UserCreatedEvent
```

It should contain at least:

```csharp
Guid UserId
```

Use MediatR's:

```csharp
INotification
```

Example:

```csharp
public record UserCreatedEvent(Guid UserId) : INotification;
```

---

## 3. Create an Event Handler

Create a handler:

```csharp
UserCreatedEventHandler
```

implementing:

```csharp
INotificationHandler<UserCreatedEvent>
```

For demonstration purposes, the handler should log:

```text
User {UserId} was created.
```

Use `ILogger<T>` rather than `Console.WriteLine`.

---

## 4. Publish the Event

Create a simple example where a user is created.

After successfully creating the user, publish:

```csharp
await mediator.Publish(
    new UserCreatedEvent(user.Id),
    cancellationToken
);
```

The important flow should be:

```text
HTTP Request
    ↓
User Service / Handler
    ↓
Create User
    ↓
Publish UserCreatedEvent
    ↓
UserCreatedEventHandler
    ↓
Log / side effect
```

The event must be **in-process**. Do not make an HTTP request or send anything to an external message broker.

---

## 5. Show Multiple Handlers

Demonstrate one of the main benefits of MediatR notifications by creating a second handler for the same event:

```csharp
UserCreatedSendWelcomeEmailHandler
```

It should only simulate sending an email by logging:

```text
Welcome email sent to user {UserId}.
```

Therefore:

```text
UserCreatedEvent
       ↓
 ┌─────┴──────────────┐
 ↓                    ↓
Log Handler      Welcome Email Handler
```

Both handlers should execute when:

```csharp
await mediator.Publish(new UserCreatedEvent(user.Id));
```

is called.

---

## 6. Explain the NestJS Equivalent

Add a short explanation showing the conceptual mapping:

NestJS:

```typescript
this.eventEmitter.emit('user.created', {
    userId: user.id
});
```

with:

```typescript
@OnEvent('user.created')
handleUserCreated(event) {
    // ...
}
```

MediatR:

```csharp
await mediator.Publish(
    new UserCreatedEvent(user.Id),
    cancellationToken
);
```

with:

```csharp
public class UserCreatedEventHandler
    : INotificationHandler<UserCreatedEvent>
{
    public Task Handle(
        UserCreatedEvent notification,
        CancellationToken cancellationToken)
    {
        // ...
    }
}
```

Explain that:

* `INotification` represents the event
* `Publish()` publishes the event
* `INotificationHandler<T>` represents an event handler/listener
* Multiple handlers can listen to the same notification
* Everything happens inside the same process

---

## 7. Project Structure

Use a clean structure similar to:

```text
Application/
├── Events/
│   └── UserCreatedEvent.cs
│
├── EventHandlers/
│   ├── UserCreatedEventHandler.cs
│   └── UserCreatedSendWelcomeEmailHandler.cs
│
└── Users/
    └── CreateUser/
        └── CreateUserHandler.cs
```

Adapt the structure to the existing project rather than unnecessarily restructuring the entire application.

---

## 8. Dependency Injection

Configure MediatR through the .NET dependency injection container.

Show exactly what needs to be added to `Program.cs`.

Prefer assembly scanning so new handlers are automatically discovered.

---

## 9. Error Handling

Explain what happens if one notification handler throws an exception.

Do not add complicated retry infrastructure.

Keep the default behavior unless the existing application already has an established error-handling strategy.

---

## 10. Important Constraints

* .NET 10
* C#
* MediatR
* In-process events only
* No custom event bus unless absolutely necessary
* Do not over-engineer the solution
* Reuse the application's existing architecture and conventions
* Make the smallest reasonable set of changes

Before modifying files, inspect the existing project structure and determine the appropriate project/assembly where the events and handlers should live.

After implementation, show:

1. Files created/modified
2. NuGet packages added
3. `Program.cs` changes
4. Event definition
5. Event handlers
6. Event publishing code
7. How to test it
8. A brief explanation of how this compares to NestJS `EventEmitter2`
