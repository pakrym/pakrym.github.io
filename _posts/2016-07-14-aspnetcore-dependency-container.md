---
title: ASP.NET Core compatible dependency injection container in 200 lines of code.
---

# Setup

To start we need two projects - one for container implentation and one for tests. Implementaion would target `netstandard1.1` as it has all the things we need, and reference single dependency `Microsoft.Extensions.DependencyInjection.Abstractions` which is set of abstraction ASP.NET Core us you to implement.

```
{
  "version": "1.0.0-*",
  "dependencies": {
    "Microsoft.Extensions.DependencyInjection.Abstractions": "1.0.0-*",
  },
  "frameworks": {
    "netstandard1.1": {
      "dependencies": {
        "System.Reflection": "4.1.0",
        "System.Collections": "4.0.11-*"
      }
    }
  }
}

```
We'll need a source file that will contain stub implementation:

```
using System;

namespace SimpleDI
{
    public class ServiceProvider : IServiceProvider
    {
        public object GetService(Type serviceType)
        {
            throw new NotImplementedException();
        }
    }
}

```


For testing lets create typical `dotnet cli` test project and reference `Microsoft.Extensions.DependencyInjection.Specification.Tests` which contains a set of unit test our implementation would be required to pass.

```
{
  "version": "1.0.0-*",
  "dependencies": {
    "dotnet-test-xunit": "2.2.0-*",
    "SimpleDI": "1.0.0-*",
    "Microsoft.Extensions.DependencyInjection.Specification.Tests": "1.0.0-*",
    "xunit": "2.2.0-*"
  },
  "frameworks": {
    "netcoreapp1.0": {
      "dependencies": {
        "Microsoft.NETCore.App": {
          "version": "1.0.0-*",
          "type": "platform"
        }
      }
    },
    "net451": {}
  },
  "testRunner": "xunit"
}
```
And in last step lets add a stub test class, inheriting `DependencyInjectionSpecificationTests` and overriding container creation delegate.

```
using System;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.DependencyInjection.Specification;

namespace SimpleDI.Tests
{
    public class SimpleDISpecificationTests : DependencyInjectionSpecificationTests
    {
        protected override IServiceProvider CreateServiceProvider(IServiceCollection collection) =>
            new ServiceProvider();
    }
}

```
Run `dotnet test`.

**Passed: 0/45**

# Trivial case

You'll notice that container creation delegate receives `IServiceCollection` interface instance, which by itself is `IList<ServiceDescriptor>`. ServiceDescriptor contains information about a registered service:

 1. ImplementationFactory - if instance of service is created using factory this property would contain factory delegate.
 2. ImplementationInstance - if service is represented by a know instance this property will contain it.
 3. ImplementationType - if instance of service is created by constructing type this property would contain type reference.
 4. ServiceType - type reference of a service being registered.
 5. Lifetime - we'll discuss later.

So to implement trivial case we would need to store the descriptor list and use it to return an instance from `GetService` method:

```
using System;
using System.Collections.Generic;
using System.Linq;
using Microsoft.Extensions.DependencyInjection;

namespace SimpleDI
{
    public class ServiceProvider : IServiceProvider
    {
        private readonly IList<ServiceDescriptor> _services;

        public ServiceProvider(IList<ServiceDescriptor> services)
        {
            _services = services;
        }

        public object GetService(Type serviceType)
        {
            var descriptor = _services.FirstOrDefault(service => service.ServiceType == serviceType);
            if (descriptor == null)
            {
                return null;
            }

            if (descriptor.ImplementationInstance != null)
            {
                return descriptor.ImplementationInstance;
            }
            else if (descriptor.ImplementationFactory != null)
            {
                return descriptor.ImplementationFactory(this);
            }
            else if (descriptor.ImplementationType != null)
            {
                return Activator.CreateInstance(descriptor.ImplementationType);
            }
            // we should never get here
            throw new NotImplementedException();
        }
    }
}
```
Changing test code to pass `IServiceCollection` to `ServiceProvider` constructor:

```
using System;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.DependencyInjection.Specification;

namespace SimpleDI.Tests
{
    public class SimpleDISpecificationTests : DependencyInjectionSpecificationTests
    {
        protected override IServiceProvider CreateServiceProvider(IServiceCollection collection) =>
            new ServiceProvider(collection);
    }
}
```
After running `dotnet test` you'll see that even this implementation passes lots of tests.

**Passed: 18/45**

## Constructor

Lots of the test fail with `No parameterless constructor defined for this object.` on `Activator.CreateInstance` line because service implementation type has constructor with parametes. Specification requires us to use constructor with most parameters if all of them can be satisfied. Lets write the code:

```
        private object CreateInstance(Type implementationType)
        {
            var consctructors = implementationType.GetTypeInfo().DeclaredConstructors.OrderByDescending(c => c.GetParameters().Length);
            List<object> arguments = new List<object>();

            foreach (var constructorInfo in consctructors)
            {
                var parameters = constructorInfo.GetParameters();
                arguments.Clear();

                foreach (var parameterInfo in parameters)
                {
                    var value = GetService(parameterInfo.ParameterType);
                    // Could not resolve parameter
                    if (value == null)
                    {
                        break;
                    }
                    arguments.Add(value);
                }

                if (parameters.Length != arguments.Count)
                {
                    continue;
                }
                // We got values for all paramters
                return Activator.CreateInstance(implementationType, arguments.ToArray());
            }
            throw new InvalidOperationException("Cannot find constructor");
        }
```

And replace `Activator.CreateInstance` call with `CreateInstance` call in `GetService`.

**Passed: 24/45**

## Lifetime

There are three kinds of service ligetimes in ASP.NET Core:

 1. Transient - new instance is created per request
 2. Singleton - single instance is created and cached for whole container hierarchy
 3. Scoped - instance is cached in `IServiceProvider` it was created in.

To describe a notion of scopes we need to look at `IServiceScopeFactory` and `IServiceScope` interfaces:

```
public interface IServiceScopeFactory
{
    IServiceScope CreateScope();
}

public interface IServiceScope : IDisposable
{
    IServiceProvider ServiceProvider { get; }
}
```
So service scope is a service provider that guarantess that all services created using it would be disposed with scope being disposed.

After implementing lifetime support service provider implementation grew quite a bit:

```
public class ServiceProvider : IServiceProvider, IServiceScopeFactory, IDisposable
{
    private readonly Dictionary<Type, object> _scoped = new Dictionary<Type, object>();
    private readonly List<object> _transient = new List<object>();

    private readonly IList<ServiceDescriptor> _services;
    private readonly ServiceProvider _root;

    private bool disposed;

    public ServiceProvider(IList<ServiceDescriptor> services)
    {
        _services = services;
        _root = this;
    }

    public ServiceProvider(ServiceProvider parent)
    {
        _services = parent._services;
        _root = parent._root;
    }

    public object GetService(Type serviceType)
    {
        if (serviceType == typeof(IServiceScopeFactory))
        {
            return this;
        }

        var descriptor = _services.FirstOrDefault(service => service.ServiceType == serviceType);
        if (descriptor == null)
        {
            return null;
        }
        switch (descriptor.Lifetime)
        {
            case ServiceLifetime.Singleton:
                return Singleton(serviceType, () => Create(descriptor));
            case ServiceLifetime.Scoped:
                return Scoped(serviceType, () => Create(descriptor));
            case ServiceLifetime.Transient:
                return Transient(Create(descriptor));
            default:
                throw new ArgumentOutOfRangeException();
        }
    }

    private object Transient(object o)
    {
        _transient.Add(o);
        return o;
    }

    private object Singleton(Type type, Func<object> factory)
    {
        return Scoped(type, factory, _root);
    }

    private object Scoped(Type type, Func<object> factory)
    {
        return Scoped(type, factory, this);
    }

    private static object Scoped(Type type, Func<object> factory, ServiceProvider provider)
    {
        object value;
        if (!provider._scoped.TryGetValue(type, out value))
        {
            value = factory();
            provider._scoped.Add(type, value);
        }
        return value;
    }

    private object Create(ServiceDescriptor descriptor)
    {
        if (descriptor.ImplementationInstance != null)
        {
            return descriptor.ImplementationInstance;
        }
        else if (descriptor.ImplementationFactory != null)
        {
            return descriptor.ImplementationFactory(this);
        }
        else if (descriptor.ImplementationType != null)
        {
            return CreateInstance(descriptor.ImplementationType);
        }
        // we should never get here
        throw new NotImplementedException();
    }

    public void Dispose()
    {
        if (!disposed)
        {
            disposed = true;
            foreach (var o in _transient.Concat(_scoped.Values))
            {
                (o as IDisposable)?.Dispose();
            }
        }
    }

    public IServiceScope CreateScope()
    {
        return new ServiceScope(new ServiceProvider(this));
    }

    private object CreateInstance(Type implementationType)
    {
        var consctructors = implementationType.GetTypeInfo().DeclaredConstructors.OrderByDescending(c => c.GetParameters().Length);
        List<object> arguments = new List<object>();

        foreach (var constructorInfo in consctructors)
        {
            var parameters = constructorInfo.GetParameters();
            arguments.Clear();

            foreach (var parameterInfo in parameters)
            {
                var value = GetService(parameterInfo.ParameterType);
                // Could not resolve parameter
                if (value == null)
                {
                    break;
                }
                arguments.Add(value);
            }

            if (parameters.Length != arguments.Count)
            {
                continue;
            }
            // We got values for all paramters
            return Activator.CreateInstance(implementationType, arguments.ToArray());
        }
        throw new InvalidOperationException("Cannot find constructor");
    }
}
```

Three field were added `_scoped` to cache scoped services, `_transient` to keep track of objects that were created by container and need to be disposed with it and `_root` to keep track of root container when creating child ones.

Singleton lifetime is implemented as scoped always using `_root` as a scope. Also `IServiceScopeFactory` interface was implemented to returne new instances of `ServiceScope` class containig new `ServiceProvider` object that inherits list of services from current but not cached services.

```
internal class ServiceScope : IServiceScope
{
    private ServiceProvider _serviceProvider;

    public ServiceScope(ServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public IServiceProvider ServiceProvider => _serviceProvider;

    public void Dispose()
    {
        _serviceProvider.Dispose();
    }
}

And `Dispose` method to dispose of all objects owned by service provider.s

```
**Passed: 36/45**

# Open generics

ASP.NET Core requires a feature called "open generics" to be supported by service providers, this means that if services is registered as `IService<T>` with implementation type `Service<T>` request for `IService<Foo>` would return `Service<Foo>` instance.

```

    public class ServiceProvider : IServiceProvider, IServiceScopeFactory, IDisposable
    {
        private readonly Dictionary<Type, object> _scoped = new Dictionary<Type, object>();
        private readonly List<object> _transient = new List<object>();

        private readonly IList<ServiceDescriptor> _services;
        private readonly ServiceProvider _root;

        private bool disposed;

        public ServiceProvider(IList<ServiceDescriptor> services)
        {
            _services = services;
            _root = this;
        }

        public ServiceProvider(ServiceProvider parent)
        {
            _services = parent._services;
            _root = parent._root;
        }

        public object GetService(Type serviceType)
        {
            if (serviceType == typeof(IServiceScopeFactory))
            {
                return this;
            }

            var descriptor = _services.FirstOrDefault(service => service.ServiceType == serviceType);
            if (descriptor == null)
            {
                if (serviceType.IsConstructedGenericType)
                {
                    var genericType = serviceType.GetGenericTypeDefinition();
                    descriptor = _services.FirstOrDefault(service => service.ServiceType == genericType);
                    if (descriptor != null)
                    {
                        return Resolve(descriptor, serviceType, serviceType.GenericTypeArguments);
                    }
                }
                return null;
            }
            return Resolve(descriptor, serviceType, null);
        }

        private object Resolve(ServiceDescriptor descriptor, Type serviceType, Type[] typeArguments)
        {
            switch (descriptor.Lifetime)
            {
                case ServiceLifetime.Singleton:
                    return Singleton(serviceType, () => Create(descriptor, typeArguments));
                case ServiceLifetime.Scoped:
                    return Scoped(serviceType, () => Create(descriptor, typeArguments));
                case ServiceLifetime.Transient:
                    return Transient(Create(descriptor, typeArguments));
                default:
                    throw new ArgumentOutOfRangeException();
            }
        }

        private object Transient(object o)
        {
            _transient.Add(o);
            return o;
        }

        private object Singleton(Type type, Func<object> factory)
        {
            return Scoped(type, factory, _root);
        }

        private object Scoped(Type type, Func<object> factory)
        {
            return Scoped(type, factory, this);
        }

        private static object Scoped(Type type, Func<object> factory, ServiceProvider provider)
        {
            object value;
            if (!provider._scoped.TryGetValue(type, out value))
            {
                value = factory();
                provider._scoped.Add(type, value);
            }
            return value;
        }

        private object Create(ServiceDescriptor descriptor, Type[] typeArguments)
        {
            if (descriptor.ImplementationInstance != null)
            {
                return descriptor.ImplementationInstance;
            }
            else if (descriptor.ImplementationFactory != null)
            {
                return descriptor.ImplementationFactory(this);
            }
            else if (descriptor.ImplementationType != null)
            {
                return CreateInstance(descriptor.ImplementationType, typeArguments);
            }
            // we should never get here
            throw new NotImplementedException();
        }

        public void Dispose()
        {
            if (!disposed)
            {
                disposed = true;
                foreach (var o in _transient.Concat(_scoped.Values))
                {
                    (o as IDisposable)?.Dispose();
                }
            }
        }

        public IServiceScope CreateScope()
        {
            return new ServiceScope(new ServiceProvider(this));
        }

        private object CreateInstance(Type implementationType, Type[] typeArguments)
        {
            if (typeArguments != null)
            {
                implementationType = implementationType.MakeGenericType(typeArguments);
            }
            var consctructors = implementationType.GetTypeInfo().DeclaredConstructors.OrderByDescending(c => c.GetParameters().Length);
            List<object> arguments = new List<object>();

            foreach (var constructorInfo in consctructors)
            {
                var parameters = constructorInfo.GetParameters();
                arguments.Clear();

                foreach (var parameterInfo in parameters)
                {
                    var value = GetService(parameterInfo.ParameterType);
                    // Could not resolve parameter
                    if (value == null)
                    {
                        break;
                    }
                    arguments.Add(value);
                }

                if (parameters.Length != arguments.Count)
                {
                    continue;
                }
                // We got values for all paramters
                return Activator.CreateInstance(implementationType, arguments.ToArray());
            }
            throw new InvalidOperationException("Cannot find constructor");
        }
    }
```

We added a code to deconstruct generic type and check if there is service registered with closed generic type `ISerivce<>` if it's found we construct an instance using requested type arguments.

# Open IEnumerable
When `IEnumberable<T>` is requested provider should return enumeration containing resolved instances for all registered services with type T.

To implement this, lets change `GetService` method to this:

```
public object GetService(Type serviceType)
{
    if (serviceType == typeof(IServiceScopeFactory))
    {
        return this;
    }

    var descriptor = _services.FirstOrDefault(service => service.ServiceType == serviceType);
    if (descriptor == null)
    {
        if (serviceType.IsConstructedGenericType)
        {
            var genericType = serviceType.GetGenericTypeDefinition();
            if (genericType == typeof(IEnumerable<>))
            {
                var genericAgument = serviceType.GenericTypeArguments[0];
                var descriptors = _services.Where(s => s.ServiceType == genericAgument).ToList();
                var array = Array.CreateInstance(genericAgument, descriptors.Count());
                for (int i = 0; i < array.Length; i++)
                {
                    array.SetValue(Resolve(descriptors[i], descriptors[i].ServiceType, null), i);
                }
                return array;
            }

            descriptor = _services.FirstOrDefault(service => service.ServiceType == genericType);
            if (descriptor != null)
            {
                return Resolve(descriptor, serviceType, serviceType.GenericTypeArguments);
            }
        }
        return null;
    }
    return Resolve(descriptor, serviceType, null);
}
```

**Passed: 42/45**

# Final fixes
Two of last three tests are failing because service provider should be able to return itself when `IServiceProvider` type is requested so changing first `if` in `GetService` method fixes them:

```
if (serviceType == typeof(IServiceProvider) ||
    serviceType == typeof(IServiceScopeFactory))
{
    return this;
}
```

And last failure is caused because we are selecting first `ServiceDescriptor` in line `var descriptor = _services.FirstOrDefault(service => service.ServiceType == serviceType);` and specification requires that services registered later override ones registered earlier. Thats an easy fix `FirstOrDefault` -> `LastOrDefault`.

**Passed: 45/45**

# Web application

It's time to use our newly written service provider in actual web application. For this lets create Web Application project in Visual Studio and change `ConfigureServices` to:

```
public IServiceProvider ConfigureServices(IServiceCollection services)
{
    // Add framework services.
    services.AddApplicationInsightsTelemetry(Configuration);

    services.AddMvc();
    return new SimpleDI.ServiceProvider(services);
}
```

Start the application!
If everything was done right you will see ASP.NET Core MVC application running and fully functional in your browser.