---
title: "Using HttpClient with System.Text.Json"
date: 2021-01-22T17:01:55-05:00
lastmod: 2021-01-22T17:15:00-05:00
summary: Using the HttpClient with System.Text.Json with System.Net.Http.Json.
tags:
- .net
keywords: .NET, .NET Core, HttpClient, Json, System.Text.Json, System.Net.Http.Json, Microsoft.AspNet.WebApi.Client
---

When working with JSON and `HttpClient` in the .NET Framework, it was pretty common to add a reference to [Microsoft.AspNet.WebApi.Client](https://www.nuget.org/packages/Microsoft.AspNet.WebApi.Client/), which added a number of extension methods to `HttpClient` and `HttpResponseMessage` that made it simpler to send and receive JSON documents.

That package still work with .NET Core, but it has a dependency on the [Newtonsoft.Json](https://www.nuget.org/packages/Newtonsoft.Json/). Since .NET Core 3+ comes with [System.Text.Json](https://docs.microsoft.com/en-us/dotnet/standard/serialization/system-text-json-how-to?pivots=dotnet-5-0), a more async friendly JSON parser, it might be worth making the switch, espcially in newer projects.

Conveniently, there's also a package called [System.Net.Http.Json](https://www.nuget.org/packages/System.Net.Http.Json/) that includes extension methods to make your serialization and deserializion easier.

# Microsoft.AspNet.WebApi.Client

This section will show a few calls made with `Microsoft.AspNet.WebApi.Client`.

The assumption is that you have an `HttpClient` already instanciated with the name `client`, a request model called `RequestModel`, and a response model called `ResponseModel`.

## A basic GET

First, we get a `HttpResponseMessage` from the client by making a request.
```cs
HttpResponseMessage response = await client.GetAsync("/");
```

Then, we using the generic verion of the `ReadAsAsync<T>` extension method to read and deserialize the JSON document into our object.
```cs
Task<ResponseModel> responseModel = await response.Content.ReadAsAsync<ResponseModel>();
```

## A POST with a request document

Sending a document is also pretty straightforward.

First, we initialize our request model.
```cs
RequestModel requestModel = new RequestModel();
```

Then, we make our request, including our model that will get serialized through the `PostAsJsonAsync` extension method.
```cs
HttpResponseMessage response = await client.PostAsJsonAsync("/", requestModel);
```

Finally, we read the response model, just like we did for the GET.
```cs
ResponseModel responseModel = await response.Content.ReadAsAsync<ResponseModel>();
```

# System.Net.Http.Json

Here, we'll be doing the same exact thing, but using the extension methods in `System.Net.Http.Json` instead.

## A basic GET

Again, we get a `HttpResponseMessage` from the client by making a request. This is the same as for `Microsoft.AspNet.WebApi.Client`.
```cs
HttpResponseMessage response = await client.GetAsync("/");
```

Then, we using the generic verion of the `ReadFromJsonAsync<T>` extension method to read and serialize the JSON document into our object.
```cs
ResponseModel responseModel = await response.Content.ReadFromJsonAsync<ResponseModel>();
```

Note that if you don't need to do any processing on the `HttpResponseMessage`, there's also a convenience method called `GetFromJsonAsync` so you can skip that step entirely.
```cs
ResponseModel responseModel = await client.GetFromJsonAsync<ResponseModel>("/");
```

## A POST with a request document

Sending a document is also pretty straightforward.

First, we initialize our request model.
```cs
RequestModel requestModel = new RequestModel();
```

Then, we make our request, including our model that will get serialized through the `PostAsJsonAsync` extension method, which conveniently has the same name as the extension method from `Microsoft.AspNet.WebApi.Client`.
```cs
HttpResponseMessage response = await client.PostAsJsonAsync("/", requestModel);
```

Finally, we read the response model, just like we did for the GET.
```cs
ResponseModel responseModel = await response.Content.ReadFromJsonAsync<ResponseModel>();
```

# Final notes

While serializing and deserializing documents with the `HttpClient` isn't particularly challenging, it does lead to a fair amount of repetition, so using these extension methods (or rolling your own!) should make your code look a lot leaner.