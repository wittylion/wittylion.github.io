---
layout: post
title: Auto Processor
author: Sergey Solomentsev
comments: true
---

How can processors and their logic be simpler and more descriptive? This time we have to use the power of C# language. Why should one, who uses this library, write massive constructions like `context.GetPropertyValueOrNull<int>("Age")` when null is already a default value and age can be a name of the parameter of the method. Taking this into account an idea of `AutoProcessor` was born.

![Easy meal](/assets/posts/auto-processor/auto-supra.jpg)

Image by [ilham mustakim](https://pixabay.com/users/ilhamtakim0612-5799470/?utm_source=link-attribution&utm_medium=referral&utm_campaign=image&utm_content=4565272) from [Pixabay](https://pixabay.com/?utm_source=link-attribution&utm_medium=referral&utm_campaign=image&utm_content=4565272)

# What new brings an AutoProcessor

With auto-processor you can separate the logic into smaller pieces that will be executed one by one. Even though processor itself should have a single responsibility, it might be that code still looks big, that's why you can create a processor with several methods **without declaring them**:

```cs
public class Proc : AutoProcessor
{
    [ExecuteMethod]
    public void Method1()
    {
        Console.WriteLine("Method 1");
    }

    [ExecuteMethod]
    public void Method2()
    {
        Console.WriteLine("Method 2");
    }
}
```

By default methods are sorted by name in ascending order, but you can specify an order yourself by providing a number parameter into the method, like this: `ExecuteMethod(Order = 100)`.

Parameters of the methods are also taken into account. Name of the parameter will be searched in the context properties and in case there is a property with such name, its value will be passed for execution of the method:

```cs
public class Proc : AutoProcessor
{
    [ExecuteMethod]
    public void Method(string message)
    {
        Console.WriteLine(message);
    }
}

public class Program
{
    public static void Main()
    {
        var context = new PipelineContext(new { Message = "Hello World!" });
        new Proc().RunSync(context);
    }
}
```

In case there is a need to do something with a context, it can be passed to parameters of the method with type derived from `PipelineContext`, for example:

```cs
public class Proc : AutoProcessor
{
    [ExecuteMethod]
    public void Method1(PipelineContext context)
    {
        context.AddError("Error");
    }
    
    [ExecuteMethod]
    public void Method2(QueryContext<string> context)
    {
        context.SetResultWithInformation("Done", "Result set.");
    }
}
```

And there is no restriction of the amount of parameters of the context that you want to accept in the method.

To bring more flexibility to the parameters and pipeline execution, a `ContextParameter` attribute was added. It allows to skip method if a parameter required or provide a default value if property was not found or even abort a pipeline with a specific error message. Here how it looks:

```cs
public class Proc : AutoProcessor
{
    [ExecuteMethod]
    public void Method(
        [ContextParameter(
            Name = "Message", 
            AbortIfNotExist = true, 
            ErrorMessage = "A message must be specified")]
        string helloWorld)
    {
        Console.WriteLine(helloWorld);
    }
}

public class Program
{
    public static void Main()
    {
        var context = new PipelineContext(new { Message = "Hello World!" });
        new Proc().RunSync(context);
    }
}
```

One more thing that is taken into account is the returning result of the method.

Actions that are executed when a specific type is returned:

- `object` - all the properties with their values will be taken from this object and added to the pipeline context as properties.
- `Task` - the task will be awaited.
- `Action` - will be executed.
- `Action<PipelineContext>` - will pass an execution context into an action and execute it.
- `Func<PipelineContext, T>` - will pass an execution context into a function and execute it, the result of the execution will be passed into result handler.
- `IEnumerable` - Will iterate through the objects and each object will be passed into result handler
- `Task<T>` - the task will be awaited and its result T will be passed again to be processed as a result.

To avoid returning of the actions and functions, in auto-processor were added several predifined method working with a context that can be returned from the method, like this:

```cs
public class Proc : AutoProcessor
{
    [ExecuteMethod]
    public IEnumerable Method(
        string phrase, string meaning)
    {
        yield return AddInformationMessage("Phrase is: " + phrase);
        yield return AddInformationMessage("Meaning is: " + meaning);
    }
}
```

# Full Example

Here is a full example of simple project divided into assemblies:

Assembly1:

```cs
using Pipelines.Implementations.Processors;
using System;
using System.Net.Http;

namespace CommonLib
{
    namespace MyNamespace
    {
        public class Initializer
        {
            /// <summary>
            /// Known problem.
            /// </summary>
            public static void JustBe() { }
        }

        public class DownloadStrings : AutoProcessor
        {
            [ExecuteMethod]
            public object DownloadRandomString(HttpClient client, string url)
            {
                try
                {
                    var jsonString = (client ?? new HttpClient()).GetStringAsync(url).Result;
                    return new { jsonString };
                }
                catch
                {
                    return null;
                }
            }

            [ExecuteMethod]
            public object ParseJsonResult(
                [ContextParameter(Required = true)]
                string jsonString)
            {
                dynamic randomWords = Newtonsoft.Json.JsonConvert.DeserializeObject(jsonString);
                return new { Words = randomWords.data };
            }
        }

        public class SelectRandomString : AutoProcessor
        {
            [ExecuteMethod]
            public object GenerateRandomNumber(
                [ContextParameter(Required = true)]
                dynamic words)
            {
                var length = words.Count;
                var random = new Random(DateTime.Now.Millisecond).Next(length);
                return new { number = random };
            }

            [ExecuteMethod]
            public object SelectRandomWord(
                [ContextParameter(Required = true)]
                dynamic words, int number)
            {
                return new
                {
                    phrase = words[number]["phrase"].ToString(),
                    meaning = words[number]["meaning"].ToString()
                };
            }
        }
    }
}
```

Assembly 2:

```cs
using System;
using System.Collections;
using Pipelines;
using Pipelines.ExtensionMethods;
using Pipelines.Implementations.Contexts;
using Pipelines.Implementations.Pipelines;
using Pipelines.Implementations.Processors;

namespace CommonLib
{
    namespace MyNamespace
    {
        [ProcessorOrder(1000)]
        public class WriteFormattedMessages : AutoProcessor
        {

            [ExecuteMethod]
            public IEnumerable GetMessage(
                [ContextParameter(AbortIfNotExist = true)]
                string phrase, string meaning)
            {
                yield return AddInformationMessage("Phrase is: " + phrase);
                yield return AddInformationMessage("Meaning is: " + meaning);
            }
        }
    }
}

namespace ConsoleApp2
{
    class Program
    {
        static void Main()
        {
            CommonLib.MyNamespace.Initializer.JustBe();
            PipelineContext context =
                ContextConstructor.BuildContext()
                .Use("url", "https://randomwordgenerator.com/json/phrases.json")
                .OriginalContext;

            new NamespaceBasedPipeline("CommonLib.MyNamespace")
                .RunSync(context);

            Console.WriteLine(context.GetSummaryMessage(format: o => o.MessageType.ToString() + ":" + Environment.NewLine + o.Message + Environment.NewLine));
            Console.ReadLine();
        }
    }
}
```