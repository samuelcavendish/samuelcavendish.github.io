---
layout: post
title:  "Scientist & DependencyInjection"
---

Recently I've been using Scientist to test critical path changes within a codebase. The library is amazing, but one thing I found a bit cumbersome was getting the dependencies in place for the Try method. I often wanted to use interfaces for the experiment that were already being used and registered by the main application. To plug this hole, I created a library, Scientist.DependencyInjection

## What is it?

Scientist.DependencyInjection is a library that allows you to register an ExperimentContext that contains dependencies for use with your experiment. Here's how it looks to register an experiment

```
var container = Host.CreateDefaultBuilder()
    .ConfigureServices(services =>
    {
        services
            .AddTransient<IScientist, Scientist>()
            .AddTransient<IResultPublisher, Publisher>();
            // Normal dependency
            .AddSingleton(originalDependency);
            .AddExperimentContexts(scientist =>
            {
                scientist.AddExperimentContext<ExperimentContext>(services =>          
                {
                    // Experiment dependency
                    return services.AddSingleton(experimentDependency);
                });
            });
    }).Build();
```

When adding multiple contexts, calls to AddExperimentContext can be chained. In the snippit, we're adding the ExperimentContext (which inherits IExperimentContext) with the singleton experiment dependency. Here's what the ExperimentContext looks like

```
internal class ExperimentContext : IExperimentContext
{
    public ExperimentContext(Dependency experimentDependency)
    {
        ExperimentDependency = experimentDependency;
    }

    public Dependency ExperimentDependency { get; }
}
```

The experiment context simply exposes the dependency for use in experiments

## How do I use the context

My preferred way of using the context is using Scientist via Dependency Injection. From the example above, you can see I'm registering IScientist & IResultPublisher in the container. I've been using it within an experiment class, e.g.

```
internal class Experiment
{
    private readonly IScientist _scientist;
    private readonly Dependency _experimentDependency;
    private readonly ExperimentContext _experimentContext;

    public Experiment(IScientist scientist, Dependency experimentDependency, ExperimentContext experimentContext)
    {
        _scientist = scientist;
        _experimentDependency = experimentDependency;
        _experimentContext = experimentContext;
    }

    public Guid GetValue()
    {
        return _scientist.Experiment<Guid>("Experiment Name", experiment =>
        {
            experiment.Use(() => _experimentDependency.GetValue()); // old way
            experiment.Try(() => _experimentContext.ExperimentDependency.GetValue()); // new way
        });
    }
}
```

I can then register the Experiment class in the containter and use it in the critical path being tested. The example experiment being used can be found in the test project to see it working.

## Summary

Hopefully some other users of Scientist can get some use out of this pattern to allow multiple dependencies with the same interface. Check out the package on [GitHub](https://github.com/samuelcavendish/Scientist.DependencyInjection) and [Nuget.org](https://www.nuget.org/packages/Scientist.DependencyInjection/). Happy sciencing!
