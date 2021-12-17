---
layout: post
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

### Setup TestServer

Integration Tests which I will setup will be mocking HttpServer, and in those test I will create a HttpClient, which will call our API.

In the .NET 6, it was introduced new approach of starting application with much smaller and cleaner Program.cs. If you have created a new .NET 6 application,
your Program.cs file should look like this:

// 

While you have your test project created, let's crates 

### References



