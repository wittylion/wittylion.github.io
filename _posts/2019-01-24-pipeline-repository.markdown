---
layout: page
title: Pipeline Repository
author: Sergey Solomentsev
comments: true
---

In this article will be described how to use pipeline repository and how it can help on finding code blocks represented by pipelines.

![A man is looking for a book](/assets/posts/pipeline-repository/looking-for-book.jpeg)

# Overview

The Repository pattern is used to decouple the business logic and the data access layers in application.

In Pipelines.Net was introduced an abstraction for pipeline repository called `IPipelineRepository`. This allows to bring different pipeline providers i.e. from XML where each pipeline is described with tags or JSON file that was received from server, from web resource by downloading string or from SQL by executing query. Being able to implement different repositories will allow to manipulate code more flexible.

# What does the pipeline repository consist of

It consist of tow methods that have to be implemented. Each method accepts a query parameter of type `QueryContext` (was decided to use query context since it is possible to dynamically specify parameters and provide a result with informational messages).

These methods look like this:

```c#
void GetSingle(QueryContext<IPipeline> query);
void GetMany(QueryContext<IPipeline[] > query);
```

Get single - returns only one pipeline, when get many - returns all or subset of pipelines depending on query. User can get resulting pipeline by using `query.GetResult()` method or `query.GetResultOr(fallbackValue)` which allows to avoid using nulls.

# Example

This example shows how to create a simple repository that retrieves a pipeline depending on the property provided by user.

```c#
class Program
{
    static void Main()
    {
        // Use context builder to create new query context.
        var query = ContextConstructor.BuildQueryContext<IPipeline>()
            .Use("pipelineName", "helloWorld")  // use pipeline name as a parameter
            .OriginalContext;                   // get a built context

        // Using an example implementation of repository (see below).
        new PipelineRepository().GetSingle(query);

        // Get a found pipeline and run it.
        query.GetResult().RunSync();

        // Output: Hello world!
    }

    /// <summary>
    /// An example implementation of pipeline repository.
    /// </summary>
    public class PipelineRepository : IPipelineRepository
    {
        /// <summary>
        /// Depending on pipelineName parameter in query
        /// returns one of the predefined pipelines.
        /// </summary>
        /// <typeparam name="TQuery">
        /// Type of the methods query that is derived from
        /// query context to provide a result in the same context.
        /// </typeparam>
        /// <param name="query">
        /// The query argument, containing properties,
        /// to be used when querying pipelines.
        /// </param>
        public void GetSingle<TQuery>(TQuery query) // query with additional info
            where TQuery : QueryContext<IPipeline>
        {
            if (!query.HasProperty("pipelineName")) // check the parameter
            {
                // If there is no property then pipeline cannot be retrieved.
                query.AbortPipelineWithErrorAndNoResult(
                    "Context must have property 'PipelineName' of type String.");
                return;
            }

            var pipelineName = // get property
                query.GetPropertyValueOrDefault("pipelineName", string.Empty);

            switch (pipelineName)
            {
                case "empty": // empty pipeline requested

                    // Return result in same context with result description.
                    query.SetResultWithInformation(
                        PredefinedPipeline.Empty,
                        "An empty pipeline was found for your request."
                    );
                    break;

                case "helloWorld": // another pipeline requested

                    // Return sample pipeline making console output.
                    query.SetResultWithInformation(
                        PredefinedPipeline.FromProcessors( // new pipeline
                            ActionProcessor.FromAction(    // new processor
                                () => Console.WriteLine("Hello world!"))),
                        "Hello world pipeline was found for a request."
                    );
                    break;
            }
        }

        public void GetMany<TQuery>(TQuery query) 
            where TQuery : QueryContext<IPipeline[]>
        {
            // your implementation here
        }
    }
}

```
