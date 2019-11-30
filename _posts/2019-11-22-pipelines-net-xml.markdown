---
layout: page
title: Pipelines.Net XML
author: Sergey Solomentsev
comments: true
---

Composing pipelines can be treated as an art: how many processor would you like to have, how would you call them, how do you put them together. Here will be explained how to compose pipelines with a [Pipelines.Net.Xml](https://www.nuget.org/packages/Pipelines.Net.Xml/) nuget library.

![Smashing Blocks Like Atoms](/assets/posts/pipelines-net-xml/smashing-blocks-like-atoms.jpg)
Photo by [Jelleke Vanooteghem](https://unsplash.com/@ilumire?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/baby-blocks?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

# Example

Suppose you have several pipelines in your project: `CheckCityDistance` and `HelloWorld`. You put them in your project in structure like this:

```
Application.csproj
|_Pipelines
  |_CheckCityDistance
    |_ValidateData.cs
    |_PerformCalculation.cs
  
  |_HelloWorld
    |_Hello.cs
    |_World.cs
```

Each pipeline is presented by folder and processors inside. Then you need to compose the processors in an order. You can do this by creating another class derived from IPipeline with predefined processors inside:

```cs
public class HelloWorldPipeline : IPipeline
{
    public IProcessor[] Processors { get; }
    public HelloWorldPipeline()
    {
        Processors = new IProcessor[] {
            new Hello(),
            new World()
        };
    }

    public IEnumerable<IProcessor> GetProcessors()
    {
        return Processors;
    }
}
```

You can use PredefinedPipeline class:

```cs
// using Pipelines.Implementations.Pipelines;

PredefinedPipeline.FromProcessors<Hello, World>()
```

Or you can use a separate XML file, where you can describe all your processors in an order with a neat structure like this:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<pipelines>
    <helloWorld>
        <processor type="Application.Pipelines.HelloWorld.World, Application" />
        <processor type="Application.Pipelines.HelloWorld.Hello, Application" />
    </helloWorld>

    <checkCityDistance>
        <processor type="Application.Pipelines.CheckCityDistance.ValidateData, Application" />
        <processor type="Application.Pipelines.CheckCityDistance.PerformCalculation, Application" />
    </checkCityDistance>    
</pipelines>
```

Then to use it, you may use `PipelinesXmlApi` class like this:

```cs
public class Program
{
    public static void Main()
    {
        // Load XML Document from bin/Pipelines folder.
        var document = XDocument.Load("./Pipelines/Pipelines.xml");

        // Pass a document and a path to a pipeline for parsing.
        var pipeline = PipelinesXmlApi.GetPipelineFromXmlByXPath(
            document, 
            "/pipelines/helloWorld", 
            defaultValue: PredefinedPipeline.Empty
        );

        var context = ContextConstructor.Create();
        pipeline.Run(context);
        Console.WriteLine(context.GetPropertyValueOrDefault<string>("message", "No message."));
    }
}
```

Library also allows to customize the xml tags and attributes for processors. For example, if instead of word processor you want to use just letter 'p' and instead of word type just letter 't' - you can use parser directly instead of API like this:

```cs
public class Program
{
    public static void Main()
    {
        var document = XDocument.Load("./Pipelines/Pipelines.xml");
        var pipeline = PipelinesXmlApi.Parser.GetPipeline(new GetPipelineContext()
        {
            XElement = document.Root.XPathSelectElement("/pipelines/helloWorld"),
            ProcessorTagName = "p",
            TypeAttributeName = "t"
        });
        var context = ContextConstructor.Create();
        pipeline.Run(context);
        Console.WriteLine(context.GetPropertyValueOrDefault<string>("message", "No message."));
    }
}
```