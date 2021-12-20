---
layout: post
comments: true
title:  "How to setup API integration tests in .NET 6"
date:   2021-12-17 17:09:35 +0100
categories: howto
tags: integration-tests dotnet csharp xunit
---

### What is the goal?

In this post I want to explain how to setup integration tests in .NET 6. I want to use TestServer and Entity Framework in memory database to check if my API is working correctly.

### Prequisites

As I prequisite you should have API project created. I'm using the WebAPI project which is using Clean Architecture approach.

### Add xUnit test project

Firstly, let's create the test project. I will be using xUnit as it has a really nice feature of injecting services into the Tests. This feature will be used to inject our startup class to the test class.

You can create xUnit Test project using the .NET CLI:

{% highlight shell %}

> mkdir MyProject.Tests.Integration
> cd MyProject.Tests.Integration
> dotnet new xunit

{% endhighlight %}

### Setup in memory web server with TestServer

#### Update Program.cs

Integration Tests which I will setup will be using in-memory web server. This mean, I don't need to deploy web app to server, to do integration testing.
I will use TestServer instance, invoke HttpClient from there, which will call endpoints to test them.

In the .NET 6, it was introduced new approach of starting application with much smaller and cleaner Program.cs. If you have created a new .NET 6 application,
your Program.cs file should look like this:

{% highlight csharp %}

var builder = WebApplication.CreateBuilder(args);

// configure services part

var app = builder.Build();

// Configure the HTTP request pipeline.

app.Run();

{% endhighlight %}

The first think to is to add the lines below to the end of the Program.cs file. There is not expicitly created Program class, so we will do this by the end of the file.

{% highlight csharp %}

// for integration test
public partial class Program { }

{% endhighlight %}

#### Create IntegrationTestsFactory

Next, I we need to create factory, which later be injected to our class as dependency.

{% highlight csharp %}

public class IntegrationTestsFactory<TStartup> : WebApplicationFactory<TStartup> where TStartup : class
{
    public IntegrationTestsConfiguration IntegrationTestsConfiguration { get; private set; }
    
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.UseEnvironment("Tests");
        builder.ConfigureServices(services =>
        {
            InitIntegrationTestsConfiguration(services);
            
            services.OverrideDbWithInMemoryDb<TStartup>(IntegrationTestsConfiguration);
            services.OverrideAuthentication();
        });
    }

    private void InitIntegrationTestsConfiguration(IServiceCollection services)
    {
        var configuration = services.BuildServiceProvider().GetService<IConfiguration>();
        IntegrationTestsConfiguration =  configuration
            .GetSection("IntegrationTests").Get<IntegrationTestsConfiguration>();
    }
}

{% endhighlight %}

You can see that, I'm switching to other environemnt. I'm using the "Test" environment. I've created new configuration file only for integration testing
which will read from appsettings.test.json file. Then you see that I'm injecting the Tests configuration - like users data or other data in `InitIntegrationTestsConfiguration` method.

Lastly I'm changing implementation of some services. I'm switching from standard database, to in-memory database, and from JWT Token implemenration to FakeJwtToken.

Having that, I'm ready to start writing integration test.

We can do much more in the factory here. We can e.g. seed the database with example data.

### Prepare Test class and first tests

### Prepare TestBase class

Next thing which I did, is preparing the TestBase class, which will prepare the HttpClient and configure e.g. the FakeJwtToken or prepare database. Let's see it:

{% highlight csharp %}

using System;
using System.Dynamic;
using System.Net;
using System.Net.Http;
using System.Threading.Tasks;
using Microsoft.Extensions.DependencyInjection;

namespace ExampleIntegrationTests;

public abstract class TestBase
{
    private readonly Microsoft.AspNetCore.TestHost.TestServer _server;
    
    protected IntegrationTestsConfiguration Configuration { get; }
    protected HttpClient Client { get; }
    
    public TestBase(IntegrationTestsFactory<Program> factory)
    {
        _server = factory.Server;
        
        Client = factory.CreateClient();
        Configuration = factory.IntegrationTestsConfiguration;
        AuthorizeClientWithFakeJwt(Client, factory.IntegrationTestsConfiguration);
    }

    private void AuthorizeClientWithFakeJwt(HttpClient client, IntegrationTestsConfiguration configuration)
    {
        dynamic bearer = new ExpandoObject();
        bearer.sub = configuration.UserIdentifier;
        bearer.oid = configuration.UserIdentifier;
        bearer.email = configuration.UserEmail;
        bearer.name = configuration.UserName;

        client.SetFakeBearerToken((object) bearer);
    }

    protected async Task ArrangeDatabaseAsync(Action<DataContext> action)
    {
        await using var context = _server.Services.GetService<DataContext>();

        action(context);

        context.SaveChanges();
    }
}

{% endhighlight %}

Having that we are ready to create first integration tests.

#### Prepare firsts integrarion test

{% highlight csharp %}

using System;
using System.Collections.Generic;
using System.Linq;
using System.Net;
using System.Net.Http;
using System.Threading.Tasks;
using FluentAssertions;
using Xunit;

namespace GetIntegrationTests;

public sealed class GetIntegrationTest : TestBase, IClassFixture<IntegrationTestsFactory<Program>>
{
    private Task<HttpResponseMessage> Act() => Client.GetAsync("api/Test");
    
    [Fact]
    public async Task GetAllTests_StatusCode200_WithOneResult()
    {
        // arrange
        await ArrangeDefaultDatabaseAsync();
        
        HttpResponseMessage response = await Act();
        TestClass queryResponse = await response.DeserializeFromQueryResponseAsync<IEnumerable<TestClass>>();

        response.StatusCode.Should().Be(HttpStatusCode.OK);
        queryResponse?.Result.Count().Should().Be(1);
    }
    
    #region Arrange

    public GetAllIntegrationTests(
        IntegrationTestsFactory<Program> factory) : base(factory)
    {
    }

    private async Task ArrangeDefaultDatabaseAsync()
    {
        await ArrangeDatabaseAsync(context =>
        {
            context.Tests.Add(new TestClass("Test name", "Test result");
        });
    }
    
    #endregion
}

{% endhighlight %}

As we see, in the Act I prepare which endpoint we will be testing. Later I arrange database using the method from TestBaseClass.
I decided to create this method in TestBase class, to not remember to save changes in every test class. With that approach I only need to
remember about setting up the database correctly before test.
Then I'm acting, and checking what is the mocked server response. It should be equivalent to the state, which I arranged before the test.

### References]
- [Supporting integration tests with WebApplicationFactory in .NET 6](https://andrewlock.net/exploring-dotnet-6-part-6-supporting-integration-tests-with-webapplicationfactory-in-dotnet-6){:target="_blank"}{:rel="noopener noreferrer"}
- [Integration tests in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/test/integration-tests?view=aspnetcore-6.0){:target="_blank"}{:rel="noopener noreferrer"}

## Comments

<div id="disqus_thread"></div>
<script>
    /**
    *  RECOMMENDED CONFIGURATION VARIABLES: EDIT AND UNCOMMENT THE SECTION BELOW TO INSERT DYNAMIC VALUES FROM YOUR PLATFORM OR CMS.
    *  LEARN WHY DEFINING THESE VARIABLES IS IMPORTANT: https://disqus.com/admin/universalcode/#configuration-variables    */
    /*
    var disqus_config = function () {
    this.page.url = PAGE_URL;  // Replace PAGE_URL with your page's canonical URL variable
    this.page.identifier = PAGE_IDENTIFIER; // Replace PAGE_IDENTIFIER with your page's unique identifier variable
    };
    */
    (function() { // DON'T EDIT BELOW THIS LINE
    var d = document, s = d.createElement('script');
    s.src = 'https://klucyszy-github-io.disqus.com/embed.js';
    s.setAttribute('data-timestamp', +new Date());
    (d.head || d.body).appendChild(s);
    })();
</script>
<noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>
