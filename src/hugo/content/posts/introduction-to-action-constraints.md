---
title: "Introduction to Action Constraints in ASP.NET Core"
date: 2019-12-02T18:45:08-05:00
tags:
- ASP.NET Core
- .NET

---
Ever had to integrate a 3rd party webhook into your application, only to realize that they won't let you configure a different address for each type of events?

A few years ago, I had to do just that for Github webhooks. Thankfully, they provide a HTTP header that describe the type of event being sent.

At the time, I used a middleware to modify the route, but there are cleaner solutions out there. In this case, I could've used a feature in ASP.NET Core called "Action Constraints", which allow you to impose constraint on otherwise identical routes.

Note: This is a follow-up on a question I had a few years ago on [StackOverflow](https://stackoverflow.com/questions/39302121/header-based-routing-in-asp-net-core), and I decided to dig a little deeper.

## Action Contraint Attribute

ASP.NET Core has an attribute called [`IActionConstraint`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.actionconstraints.iactionconstraint?f1url=https%3A%2F%2Fmsdn.microsoft.com%2Fquery%2Fdev16.query%3FappId%3DDev16IDEF1%26l%3DEN-US%26k%3Dk(Microsoft.AspNetCore.Mvc.ActionConstraints.IActionConstraint);k(DevLang-csharp)%26rd%3Dtrue&view=aspnetcore-3.0) which has only 2 members:
- `Order`, which will determine what order your constraint will be evaluated on (lowest number first)
- `Accept(ActionConstraintContext)`, which needs to return a boolean that will tell the framework whether to use this route

Let's see what an implementation can look like:

```cs
public class HeaderConstraintAttribute : Attribute, IActionConstraint
{
    public int Order => 0;
    private string Header { get; }
    private string Value { get; }

    public HeaderConstraintAttribute(string header, string value)
    {
        Header = header;
        Value = value;
    }

    public bool Accept(ActionConstraintContext context)
    {
        if (context.RouteContext.HttpContext.Request.Headers.TryGetValue(Header, out var value) && value.Any())
        {
            return value[0] == Value;
        }

        return false;
    }
}
```

Simple enough, right? Look for a matching `Header` in the `HttpContext`, and if present, match it with `Value`.

Now, let's update our controller.

```cs
[Route("api/[controller]")]
[ApiController]
public class ActionController : ControllerBase
{
    [HttpGet]
    [HeaderConstraint("MyHeader", "A")]
    public string GetStringA()
    {
        return "A";
    }

    [HttpGet]
    [HeaderConstraint("MyHeader", "B")]
    public string GetStringB()
    {
        return "B";
    }

}
```

There are 2 things to notice here:
- Both functions are HTTP Get, to the same route (`/api/action` in this case)
- We've added the `HeaderConstraintAttribute` on each route

Now, try using your HTTP API tool of choice, such as [Postman](https://www.getpostman.com/) or [Insomnia](https://insomnia.rest/), and do a GET request to `/api/action`.

You should be getting a 404 Not Found error, which makes sense, since both routes are contrained, and neither are matching the contraint.

Now, try passing a header called `MyHeader` with the value `A`.

Surprise! The API returns `A`. Change the header value to `B`, the API now returns `B`.

This is obviously a very simple scenario, but when dealing with 3rd party APIs that sends different requests to the same address, it's a great way to keep the routes seperated, use strong-typed classes and avoid using things like `dynamic` variables.

You can view the entire code written for this project on [GitHub](https://github.com/EricStG/ActionConstraintDemo).