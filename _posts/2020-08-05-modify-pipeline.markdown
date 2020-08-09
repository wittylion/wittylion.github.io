---
layout: post
title: Modify Pipeline
author: Sergey Solomentsev
comments: true
---

In [Pipelines.Net](https://www.nuget.org/packages/Pipelines.Net/) version 1.1.8 has been introduced a new functionality that allows modifying an existing pipeline. This functionality might be useful when pipeline needs to be extended, updated or debugged by a new library or custom code.

![Man in the field doing magic](/assets/posts/modify-pipeline/open-magic.jpg)

Photo by [Aziz Acharki](https://unsplash.com/@acharki95?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/change?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

To quickly start with pipeline modification you have to use `Modify` extension method of any pipeline object. 

```cs
PredefinedPipeline.Empty.Modify();
```

`Modify` extension accepts a modification - an object that describes how the processors of the pipeline should be modified. For example processors can be added:

```cs
var composer = new ChainingModification();
composer.AddFirst()
        .AddLast()
        .After()
        .AfterEach()
        .Before()
        .Insert();

PredefinedPipeline.Empty.Modify(composer);
```

Processors can be replaced:

```cs
var composer = new ChainingModification();
composer.Instead<Original, New>();
composer.Instead(ProcessorMatcher.Custom(...), collection);

pipeline.Modify(composer);
```

Processors can be removed:

```cs
pipeline.Modify(
    composer => composer.Remove<Outdated>(),
    composer => composer.Remove(instance));
```

**Note**: method `Modify` does not mutate an original instance, so you have to assign the result of this method to a new instance:

```cs
var extendedPipeline = pipeline.Modify();
```

To match processors can be used: type refereces, object reference or custom matcher. In most cases `ChainingModification` should cover all needs, but if any specific modification is needed, there is an interface `IModificationConfiguration` that accepts collection of processors and returns a new collection. Examples of implementation can be found in the source code of the library in [Implementations.Pipelines folder](https://github.com/wittylion/Pipelines.Net/tree/develop/Pipelines/Implementations/Pipelines), like: [AddLastProcessorModification](https://github.com/wittylion/Pipelines.Net/blob/develop/Pipelines/Implementations/Pipelines/AddLastProcessorModification.cs) that is used to add new processors to the end of the processors collection or [BeforeProcessorModification](https://github.com/wittylion/Pipelines.Net/blob/develop/Pipelines/Implementations/Pipelines/BeforeProcessorModification.cs) that finds a processor using a matcher instance and inserts new collection of processors before each found by matcher processor.

There is also an interface of `IProcessorMatcher` which is just a helpful contract to use for processors matching. A static class `ProcessorMatcher` provides a set of methods to generate instances of that interface. One interesting static instance can be found in this static class, it is `ProcessorMatcher.True` that returns true for every instance. It was added in case there is a need to modify all collection. It is used in `ChainingModification.AfterEach` method. Another question is: where can we use `AfterEach` method? Let's imagine that there is a code that must be debugged in development with extra processors and be as fast as possible on production. For this might be implemented a code like this:

```cs
public IPipeline GetPipelineForEnvironment(IPipeline pipeline, IEnvironment environment)
{
    if (!environment.IsDevelopment())
    {
        return pipeline;
    }

    var logging = ActionProcessor.FromAction(context => Console.WriteLine(context));
    return pipeline.Modify(composer => composer.AfterEach(logging));
}
```

Can you imagine any other interesting scenario with `AfterEach` method? Let me know in the comments.