---
title: Details on ASP.NET Core Dependency Injection scope validation feature
---

Recently I've implemented scope validation feature in ASP.NET Core Dependency Injection implementation and wanted to give some details on it's motivation and what?.

Erly motivation for this feature was [this bug](https://github.com/aspnet/Mvc/issues/631) which was caused by scoped `IUrlHelper` instance being injected into implemenation of `IAsyncActionFilter` which was registered as singleton and led to `NullReferenceException` thrown from `UrlHelper` constructor. In this case bug was obvius because of UrlHelper implementaion details that cause exception but if service construction succed the instanse would be forever cached in singleton scope which may lead to hard to diagnose issues.

Decision was made to add diagnostics that throw in case scoped service is constructed in singleton scope:

### Singleton service directly or transitively requires scoped service

In following code sample singleton service `IFoo` transitively depends on scoped service `IBaz`. `Foo` implementation stores `IBar` instance and `Bar` stores `IBaz` instance, resulting in scoped `IBaz` forever be cached inside singleton.

``` csharp
private interface IFoo { }

private class Foo : IFoo
{
    IBar _bar;
    public Foo(IBar bar)
    {
        _bar = bar;
    }
}

private interface IBar { }

private class Bar : IBar
{
    IBaz _baz;
    public Bar(IBaz baz)
    {
        _baz = baz;
    }
}

private interface IBaz { }

private class Baz : IBaz { }

public static void Main()
{
    var serviceCollection = new ServiceCollection();
    serviceCollection.AddSingleton<IFoo, Foo>();
    serviceCollection.AddTransient<IBar, Bar>();
    serviceCollection.AddScoped<IBaz, Baz>();
    var serviceProvider = serviceCollection.BuildServiceProvider();
    serviceProvider.GetService<IFoo>();
}
```

### Scoped services is resolved from root provider by calling GetService

But you don't even need singleton to cache scoped service forever, you just need directly or indirectly resolve in from root context. Calling `GetService<IFoo>` creates transient instance of `Bar` and causes `Baz` instance to be created and cached in root container.


``` csharp

private interface IBar { }

private class Bar : IBar
{
    public Bar(IBaz baz)
    {
    }
}

private interface IBaz { }

private class Baz : IBaz { }

public static void Main()
{
    var serviceCollection = new ServiceCollection();
    serviceCollection.AddTransient<IBar, Bar>();
    serviceCollection.AddScoped<IBaz, Baz>();
    var serviceProvider = serviceCollection.BuildServiceProvider();
    serviceProvider.GetService<IBar>();
}
```

To enable diagnostics you should call `BuildServiceProvider` with `validateScopes` parameter set to `true`. This will make service provider throw if it detects invalid scoped service usage.