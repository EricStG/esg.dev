---
title: "Using Cloud SQL IAM Authentication with PostgreSQL on .NET"
date: 2023-03-28T19:15:26-04:00
lastmod: 2023-03-28T19:15:26-04:00
summary: Password are so 1990s, so why not rely on service accounts to secure your PostgreSQL setup
tags:
- Entity Framework Core
- Google Cloud
- .NET
- Npgsql
- PostgreSQL
keywords: Google Cloud, gcp, Cloud SQL, .NET, dotnet, PostgreSQL, Npgsql, Entity Framework Core, efcore
---

Google Cloud Platform supports [automatic IAM authentication for databases](https://cloud.google.com/sql/docs/postgres/authentication#automatic), but .NET is not officially supported.

Thankfully, [Npgsql](https://www.npgsql.org/), a ADO.NET Data Provider for PostgreSQL, has enough features to make it easy!

# Create and configure a service account

First, [create a service account](https://cloud.google.com/iam/docs/service-accounts-create).

Then, assign it the `roles/cloudsql.instanceUser` on your Cloud SQL instance. This can be also be done at the project level, but it's generally better to keep your scopes tighter.

Lastly, add the [service account as an IAM user](https://cloud.google.com/sql/docs/postgres/add-manage-iam-users#creating-a-database-user) to your database instance

Note: This would also work with a user account, but we'll be using service accounts to keep things simple

Once all of this is done, your service account is ready to go

# Write the code

We'll be using a `NpgsqlDataSourceBuilder`, with the `UsePeriodicPasswordProvider` property in order to refresh our GCP access token.

You will need to install the following Nuget packages in your project:
- [Google.Apis.Auth](https://www.nuget.org/packages/Google.Apis.Auth/)
- [Npgsql](https://www.nuget.org/packages/Npgsql)

We first start by readying our `GoogleCredential`.

```csharp
// You could load this from a key file, or anywhere else, but this should cover most cases.
var credentials = await GoogleCredential.GetApplicationDefaultAsync();

// We want to limit the scope of the token, so we create scoped credentials 
var scopedCredentials = credentials.CreateScoped("https://www.googleapis.com/auth/sqlservice.login");
```

Next, we need to setup a `NpgsqlDataSource` using `NpgsqlDataSourceBuilder`.

Conveniently enough, Google auth will take care of refreshing the token on demand. If the token is not expired, `GetAccessTokenForRequestAsync` will return a cached version. This means that we can keep `successRefreshInterval` and `failureRefreshInterval` pretty short (the latter should never happen). In this case, we use 1 minute and 0 minutes respectively.

```csharp
var dataSourceBuilder = new NpgsqlDataSourceBuilder();
dataSourceBuilder.UsePeriodicPasswordProvider(
    (settings, cancellationToken) => new ValueTask<string>(scopedCredentials.UnderlyingCredential.GetAccessTokenForRequestAsync(cancellationToken: cancellationToken)),
    TimeSpan.FromMinutes(1),
    TimeSpan.FromSeconds(0));

dataSourceBuilder.ConnectionStringBuilder.Database = "<your database name>";
dataSourceBuilder.ConnectionStringBuilder.Username = "<Your iam user. ex: name@project.iam>";
dataSourceBuilder.ConnectionStringBuilder.Host = "<The IP address of your database>";

var dataSource = dataSourceBuilder.Build();
```

All that's left is to create the connection and query the database.

```csharp
using var connection = dataSource.CreateConnection();
await connection.OpenAsync();

using var cmd = connection.CreateCommand();
cmd.CommandText = "SELECT 1";

var result = cmd.ExecuteScalarAsync();
Console.WriteLine(result);
```

And that's it, you now have secure, password-less access to your PostgreSQL server.

# What about Entity Framework?

Good question! It turns out that [Npgsql.EntityFrameworkCore.PostgreSQL](https://www.nuget.org/packages/Npgsql.EntityFrameworkCore.PostgreSQL/) supports passing in a `DataSource` out of the box.

In an ASP.NET setting, it would look like this:

```csharp
builder.Services.AddDbContext<Context>(options =>
{
    options.UseNpgsql(dataSource);
});
```

Want to register your DataSource for dependency injection? Sure!

```csharp
builder.Services.AddSingleton(dataSource);

builder.Services.AddDbContext<Context>((serviceProvider, options) =>
{
    var dataSource = serviceProvider.GetRequiredService<NpgsqlDataSource>();
    options.UseNpgsql(dataSource);
});
```

And that's all there is to it.