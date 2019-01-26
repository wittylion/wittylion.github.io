---
layout: page
title: Transform Property Processor
author: Sergey Solomentsev
comments: true
---

This post describes what is Transform property processor. And how it can be used.

![Transformation of the strawberry](/assets/posts/transform-processor/transformation.jpg)

# Overview

Transform property processor is a predefined processor that can be found in `Pipelines.Implementations.Processors` namespace. It can be used in pipelines with `PipelineContext` class to transform one property into another.

If you've got a link to a site in your context and you want to transform it to a ready for use request object. In this case you should only pass name of the property where the url is expected, function that creates a new request object and name of the new property.

(_note: you might skip specifying the new property name and the url property will be overriden_)

```c#
CommonProcessors.TransformProperty<PipelineContext,
            string, WebRequest>("url", WebRequest.Create);
```

# Example

_Example can be run in console application with Pipelines.Net library installed through NuGet._

In this example pipeline consists of one processor, that transforms integers into strings.

```c#
class Program
{
    static void Main()
    {
        // Creating a processor unit from static constructor.
        var processor = CommonProcessors.TransformProperty<
            PipelineContext,      // type of the processed context
            IEnumerable<int>,     // type of the origin value
            IEnumerable<string>   // type of the new value
        >(
          "numbers",              // property to transform
            GetNumberStrings,     // transform function (see below)
          "strings"               // new property
        );

        // Creating a context that will be passed through the pipeline.
        var context = ContextConstructor.Create(
            // Anonymous object with array of numbers to be processed.
            new { Numbers = new[] {1, 2, 3} }
        );

        // Creating a pipeline from the processor defined earlier and run it.
        PredefinedPipeline.FromProcessors(processor).RunSync(context);

        // Check the context for the newly created by processor property.
        var strings = context.GetPropertyValueOrDefault(
            "strings",            // property where new value is expected
            Array.Empty<string>() // default value if something went wrong
        ); 

        // The value of the strings variable is { "1", "2", "3" }
    }

    /// <summary>
    /// The function that transforms integer values into strings.
    /// </summary>
    /// <param name="context">
    /// The context of the pipeline, that can be used to get additional properties.
    /// </param>
    /// <param name="numbers">
    /// The numbers array to be transformed.
    /// </param>
    /// <returns>
    /// Transformed array of numbers.
    /// </returns>
    static string[] GetNumberStrings(
        PipelineContext context, // the context, to get additional properties
        IEnumerable<int> numbers // the numbers array to be transformed
    )
    {
        return numbers.Select(x => x.ToString()).ToArray();
    }
}
```