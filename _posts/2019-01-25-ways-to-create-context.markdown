---
layout: post
title: Ways To Create Context
author: Sergey Solomentsev
comments: true
---

Contexts are used every time pipeline is executed. Context is like a box of thingsh that is used while passing through the factory belt. Here will be shown how this box can be easily composed.

![Balerine on the railway](/assets/posts/ways-to-create-context/balerine.jpg)

# Overview

Context can be created simply by calling parameterless `new()` constructor and then filled in by using methods like `SetOrAddProperty(name, value)` or `AddOrSkipPropertyIfExists(name, value)` or `ApplyProperty(name, value, modificator)`. But what if all the properties already exist in dictionary, or in properties of some object? In this case were added ContextConstructor and PipelineContext constructor overload.

`ContextConstructor` - is a static class containing static methods like:

- `CreateFromDictionary(...)` - allows to pass dictionary where all keys will be treated as new property names and their values will be property values;
- `CreateFromProperties(...)` - allows to create pipeline context with the same properties that object has;
- `BuildContext(...)` and `BuildQueryContext(...)` - allow to add in a fluent manner new properties by calling method `Use(name, value)`.

# Example

In this example people are registering to an event with single pipeline called guest registration. This pipeline validates users and adds them to a registration list.

```c#
class Program
{
    public static void Main()
    {
        // Create pipeline from existing processors.
        var guestRegistration = 
            PredefinedPipeline.FromProcessors<
                ValidateGuestData, // validate person
                CreateGuestModel,  // create view model
                AddGuestToList     // add person to list
            >();

        // Create three random person and run them through
        // registration pipeline.
        Task.WaitAll(

            // Method 1 - using context fluent extensions.
            ContextConstructor.BuildContext()
                .Use("name", "David")
                .Use("lastname", "Jackson")
                .Use("age", 15)
                .Use("email", "test@sample.net")
                .RunWith(guestRegistration),

            // Method 2 - using anonymous object.
            new PipelineContext(new
                {
                    Name = "Karl",
                    LastName = "Jackson",
                    Age = 22,
                    Email = "test@sample.net"
                })
                .RunWithPipeline(guestRegistration),

            // Method 3 - using dictionary.
            ContextConstructor.Create<object>(
                    new Dictionary<string, object>
                {
                    ["name"] = "Dan",
                    ["lastname"] = "Smith",
                    ["age"] = 24,
                    ["email"] = "test@sample.net"
                })
                .RunWithPipeline(guestRegistration)
        );

        // GuestList.Guests:
        // - David Jackson (15)
        // - Karl Jackson (22)
        // - Dan Smith (24)
    }
}

/// <summary>
/// Sample guest model, containing only line of identification.
/// </summary>
public class GuestModel
{
    /// <summary>
    /// Simple line identifying guest.
    /// </summary>
    public string Identification { get; set; }
}

/// <summary>
/// Guest list container.
/// </summary>
public class GuestList
{
    /// <summary>
    /// A list of all registered guests.
    /// </summary>
    public static readonly List<GuestModel> Guests 
        = new List<GuestModel>();
}

/// <summary>
/// Validation class. Here should be implemented the complex user
/// verification before adding to the list.
/// </summary>
public class ValidateGuestData : SafeProcessor
{
    public override Task SafeExecute(PipelineContext args)
    {
        // Verify all properties. In case any property is invalid,
        // abort registration pipeline with error message.

        args.IfHasNoProperty("name",
            context => context.AbortPipelineWithErrorMessage(
                "We should know guest's name."));

        args.IfHasNoProperty("LastName",
            context => context.AbortPipelineWithErrorMessage(
                "Last name of guest required for identification."));

        var age = args.GetPropertyValueOrDefault("age", 0);
        if (age < 15)
        {
            args.AbortPipelineWithErrorMessage(
                "Guests must be at least 15 years old.");
        }

        return Done;
    }
}

/// <summary>
/// When user is verified, the guest model can be created.
/// </summary>
public class CreateGuestModel : SafeProcessor
{
    public override Task SafeExecute(PipelineContext args)
    {
        // Combine all properties into view model.

        var firstName = args.GetPropertyValueOrDefault("name", "[no value]");
        var lastName = args.GetPropertyValueOrDefault("lastname", "[no value]");
        var age = args.GetPropertyValueOrDefault("age", 0).ToString();

        args.SetOrAddProperty(
            "guestModel",
            new GuestModel
            {
                Identification = $"{firstName} {lastName} ({age})"
            });

        return Done;
    }
}

/// <summary>
/// When user verified and view model is created, add user to the list.
/// </summary>
public class AddGuestToList : SafeProcessor
{
    public override bool SafeCondition(PipelineContext args)
    {
        // Additionally check that guest model was created.
        return base.SafeCondition(args) && args.ContainsProperty("guestmodel");
    }

    public override Task SafeExecute(PipelineContext args)
    {
        var guest = args.GetPropertyValueOrNull<GuestModel>("GuestModel");
        GuestList.Guests.Add(guest);

        return Done;
    }
}
```