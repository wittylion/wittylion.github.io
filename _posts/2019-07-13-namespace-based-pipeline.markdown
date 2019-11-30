---
layout: page
title: Namespace Based Pipeline
author: Sergey Solomentsev
comments: true
---

Pipelines.Net tries to bring easier ways to create pipelines. With version 1.1.6 a new type of pipelines is added to the library `NamespaceBasedPipeline`.

![Streed Road Space](/assets/posts/namespace-based-pipeline/road-space.jpg)
Image by [Ryan McGuire](https://pixabay.com/users/RyanMcGuire-123690/?utm_source=link-attribution&utm_medium=referral&utm_campaign=image&utm_content=238458) from [Pixabay](https://pixabay.com/?utm_source=link-attribution&utm_medium=referral&utm_campaign=image&utm_content=238458)

# Overview

Namespace based pipeline is added for purpose of minimizing processors declaration. 

# Example

In this example used namespace `CheckCityDistance` which processors are intended to calculate distance between two cities. Implementation is not brought but the main processors are added and ready to be implemented.

```c#
using System;
using System.Threading.Tasks;
using Pipelines;
using Pipelines.Implementations.Pipelines;
using Pipelines.ExtensionMethods;
using Pipelines.Implementations.Processors;

class Program
{
    static void Main(string[] args)
    {
        // Create pipeline based on namespace and run it.
        new NamespaceBasedPipeline(nameof(CheckCityDistance))
            .RunSync(new PipelineContext());
    }
}

/// <summary>
/// New namespace that logically separates all processors related to city
/// distance calculation functionality.
/// </summary>
namespace CheckCityDistance
{
    /// <summary>
    /// A processor which purpose is to validate cities and
    /// add information messages about invalid data if needed.
    /// </summary>
    [ProcessorOrder(1)]
    public class Validate : SafeProcessor
    {
        public override Task SafeExecute(PipelineContext context)
        {
            Console.WriteLine("Validate city 1 and city 2");
            return Done;
        }
    }

    /// <summary>
    /// In case data is valid this processor will calculate the
    /// distance and put it to result field of the context.
    /// </summary>
    [ProcessorOrder(2)]
    public class CalculateTheDistance : SafeProcessor
    {
        public override Task SafeExecute(PipelineContext context)
        {
            Console.WriteLine("Using framework to calculate distance");
            return Done;
        }
    }
}
```