---
layout: post
title:  "Injectable"
---

I've often found it interesting that the Microsoft Dependency Injection package doesn't offer a way to register all implementations of a type out the box. This tends to be standard with other DI providers to save on developer friction (having to add the implementation to your container every time you add a new implementation). 

This has led to me cracking out the same code to find types to register via reflection, now, I've found myself stepping into the world of open source for the first time and packaged the process into a new library called Injectable. Here's the story of its creation and how it can be used

## My current process

As I said, before Injectable I used to rely on reflection code similar to this 

```
public static IServiceCollection AddSingletonsOfType<T>(this IServiceCollection services, Assembly assembly)
{
    var type = typeof(T);
    var instances = assembly.GetTypes()
        .Where(x => x is { IsAbstract: false, IsClass: true, IsInterface: false } && type.IsAssignableFrom(x));

    foreach (var instance in instances)
    {
        if (type.IsAbstract || type.IsInterface)
            services.AddSingleton(type, instance);
        services.AddSingleton(instance);
    }

    return services;
}
```

Then in the service registration I'd add calls to assemblies implementations like

```
services.AddSingletonsOfType<IMyInterface>(this.GetType().Assembly);
```

Injectable flips this on its head and works by decorating injectable types with an `[Inject]` attribute. Injectable will then reflect the assembly to find any classes that inherit from a injectable type (classes or interfaces). The injectable type doesn't have to live in the same assembly as the implementation to register and Injectable will keep traversing base types/interfaces so even if the `[Inject]` attribute is on an "Ancestor" the type will still be detected to be injected.

Injectable produces a list of injectable types which can then be used like so:

```
foreach (var injectable in Injectables.GetInjectables())
{
    services.AddSingleton(injectable.Service, injectable.Implementation);
}
```

You may note I'm no longer specifying the `IMyInterface`. By default Injectable will get all the implementations in the assembly, if you want to just register certain types you could change the `Injectables.GetInjectables()` to `Injectables.GetInjectables().OfType<IMyInterface>()`. You can use this functionality to choose how services are registered (e.g. register some services as Singleton & others as Transient)

## What can I register

Injectable offers 4 ways to set the service type that's registered

### Decorated (default)

Registers Decorated type as the Service making 

```
[Inject]
public interface IInterface {}

public class Implementation : IInterface {} 
```

Equivalent to

```
services.AddSingleton<IInterface, Implementation>();
```

### Implementation

Registers Implementation type as the Service making 

```
[Inject(As = InjectType.Implementation)]
public interface IInterface {}

public class Implementation : IInterface {} 
```

Equivalent to

```
services.AddSingleton<Implementation>();
```

### Decorated Implementation

Registers both Decorated and Implementation type as the Service making 

```
[Inject(As = InjectType.DecoratedAndImplementation)]
public interface IInterface {}

public class Implementation : IInterface {} 
```

Equivalent to

```
services.AddSingleton<IInterface, Implementation>();
services.AddSingleton<Implementation>();
```

### First Generic

The first generic type allows you to mark a generic interface/class to inject as a different type (From the first generic argument). What's this used for you might ask? This allows Auto-Injection even for Services you can't add the Inject attribute to directly and also to register items without a common interface. The use case that led me to create it was I have multiple "repositories" that have no common interface, e.g. `ItemReadRepository` & `ItemWriteRepository`.

I wanted the ability to register all of my repositories automatically, so how could I detect them. My solution, make each repository have a common interface just for injection so:

```
[Inject(As = InjectType.FirstGeneric)]
public interface IRepository<T> { }
public class ItemReadRepository : IRepository<ItemReadRepository> { }
public class ItemWriteRepository : IRepository<ItemWriteRepository> { }
```

Is equivalent to

```
services.AddSingleton<ItemReadRepository, ItemReadRepository>();
services.AddSingleton<ItemWriteRepository, ItemWriteRepository>();
```

The setup is a bit more complicated, but now, all my repositories are automatically registered and can be injected/received from DI like normal

```
services.GetRequiredService<ItemReadRepository>();
```

### Multiple implementations

Injectable, by default, puts the power in your control by providing you the list to register instead of doing the registration for you. This means there can be duplicate service registrations within the list returned by Injectable. Microsoft DI by default has a "Last wins" implementation so calling 

```
services.AddSingleton<IInterface, Implementation1>();
services.AddSingleton<IInterface, Implementation2>();
```

Would result in `Implementation2` being retrieved when requesting IInterface. Injectable doesn't do any work to try and resolve this, that would be down to the user as Injectable has no way of determining the order. The multiple registration does however mean you can take advantage of desired multiple registrations e.g.

```
[Inject]
public interface IMessageHandler { }
public class MessageHandler1 : IMessageHandler { }
public class MessageHandler2 : IMessageHandler { }
```

This would lead to two registrations that can be retrieved & used by Microsoft DI e.g.

```
var messageHandlers = service.GetRequiredService<IEnumerable<IMessageHandler>>();
foreach (var messageHandler in messageHandlers) {
    messageHandler.Handle(message);
}
```

### Putting the power in your hands

When authoring a library like this, it's imposible to cater to everyones expectations. Lets take an example:

```
[Inject]
public MyClass { }
public MyDerivedClass { }
```
For some users, they'd expect MyClass to be injected (which it is by default). For others, they may only expect MyDerivedClass to be injected. I could of tried handling this in the library, added a flag on the inject attribute for IncludeDecorated, but the user actually has enough information to make these decisions themselves.

In this case the user could filter the results that come back from Injectable before registering the types, like so:

```
var injectables = Injectables.GetInjectables()
    .Where(x => x.Implementation.GetAttribute<Inject>() is null);
```

As easy as that we've filtered out any implementations that have the Inject attribute on directly

### Registering Injectables

To make it faster to add Injectables to your container, Injectable provides you with 3 extension methods on IServiceCollection

```
.AddSingletons
.AddTransients
.AddScopes
```

These 3 extensions allow you to add all the results from Injectables at once instead of having to loop through the results

## Summary 
So there you have it, thanks for reading about my new package Injectable, it's open source on [Github](https://github.com/samuelcavendish/Injectable) and listed on [Nuget](https://www.nuget.org/packages/Injectable/). I hope others can get some use out of it. 