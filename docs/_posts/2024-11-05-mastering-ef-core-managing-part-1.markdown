---
layout: post
comments: true
title:  "Mastering EF Core: Managing Relationships Without Data Context Dependence (Part 1)"
date:   2021-12-17 17:09:35 +0100
categories: [ef-core-series]
tags: [dotnet, ef-core]
---


# Mastering EF Core: Managing Relationships Without Data Context Dependence (Part 1)

**Welcome to the first installment of our "Mastering EF Core" series!** This series will delve into real-life scenarios and lessons learned by our team as we navigate the intricacies of writing clean, object-oriented code using Entity Framework (EF) Core. Our goal is to share practical insights that can help you harness the full potential of EF Core in your applications.

## Introduction

Entity Framework Core is a powerful ORM that facilitates data access in .NET applications. However, mastering its nuances—especially when aiming for a domain-driven design—can be challenging. In this article, we'll explore how to manage entity relationships without relying on the data context (`_dataContext`) within your domain logic. We'll focus on a practical example that underscores the importance of understanding your database's behavior and EF Core's delete behaviors.

## The Scenario: Devices and Permissions

Imagine we're building an application that manages devices and their permissions. While our example uses "Device" and "Permission," the concepts apply broadly to any domain involving complex relationships.

### Entity Definitions

Here's how our entities are defined:

```csharp
public sealed class Device
{
    public int Id { get; private set; }
    public string Name { get; private set; }
    public List<Permission> Permissions { get; private set; }

    public Device(int id, string name)
    {
        Id = id;
        Name = name;
        Permissions = new List<Permission>();
    }
    
    public void RemovePermission(int permissionId)
    {
        Permissions.RemoveAll(p => p.Id == permissionId);
    }
}

public sealed class Permission
{
    public int Id { get; private set; }
    public DateOnly? From { get; private set; }
    public DateOnly? To { get; private set; }

    public Permission(int id, DateOnly? from, DateOnly? to)
    {
        Id = id;
        From = from;
        To = to;
    }
}
```

In this setup:

- **Device** has a collection of **Permissions**.
- We aim to remove a permission using object-oriented principles without direct manipulation of the `_dataContext`.

## The Challenge: Constraints and Delete Behaviors

When attempting to remove a permission from a device using `device.RemovePermission(permissionId)`, we encounter a foreign key constraint error upon calling `_dataContext.SaveChangesAsync()`. This error arises due to the way we've configured the relationship in EF Core and how our database handles delete behaviors.

### Initial Relationship Configuration

```csharp
public class DeviceEntityTypeConfiguration : IEntityTypeConfiguration<Device>
{
    public void Configure(EntityTypeBuilder<Device> builder)
    {
        builder.HasKey(d => d.Id);
        builder.Property(d => d.Name).IsRequired();
        
        builder.HasMany(d => d.Permissions)
            .WithOne()
            .HasForeignKey(p => p.Id)
            .OnDelete(DeleteBehavior.Restrict);
    }
}
```

Using `DeleteBehavior.Restrict` means that EF Core will prevent the deletion of a `Permission` if it would violate a foreign key constraint. This setup requires us to manage deletions explicitly, often leading to less clean code that directly interacts with `_dataContext`.

### The Problem in Code

#### Before Refactoring

```csharp
public sealed class RemovePermissionsHandlerBefore
{
    private readonly DataContext _dataContext;

    public RemovePermissionsHandlerBefore(DataContext dataContext)
    {
        _dataContext = dataContext;
    }

    public async Task Handle(int deviceId, int permissionId, CancellationToken cancellationToken)
    {
        Device device = await _dataContext.Devices
            .Include(d => d.Permissions)
            .FirstOrDefaultAsync(d => d.Id == deviceId, cancellationToken);
        
        if (device is null) return;
        
        Permission permission = device.Permissions.FirstOrDefault(p => p.Id == permissionId);
        
        if (permission is null) return;
        
        _dataContext.Permissions.Remove(permission);
        await _dataContext.SaveChangesAsync(cancellationToken);
    }
}
```

This code directly manipulates the `_dataContext` to remove the permission, coupling our domain logic with the data access layer.

## The Solution: Understanding and Using `DeleteBehavior.ClientCascade`

To achieve a cleaner, more object-oriented approach, we need to adjust our understanding of delete behaviors and how our database handles them.

### Evaluating Database Support

Before making changes, it's crucial to evaluate how your database distinguishes between `NoAction` and `Restrict`. Not all databases support different behaviors for these settings. For instance, SQL Server treats `Restrict` and `NoAction` the same way, which can affect how EF Core applies these configurations.

Understanding your database's capabilities is essential because it determines whether your chosen delete behaviors will function as intended. If the database doesn't support the specific behavior you're relying on, you might encounter unexpected constraint violations or data inconsistencies.

### Understanding `DeleteBehavior.ClientCascade`

`DeleteBehavior.ClientCascade` is an EF Core setting that:

- **Enforces delete behavior on the client side**.
- **Does not create cascading deletes in the database**.
- **Allows us to remove entities from collections and have EF Core handle the deletions upon `SaveChangesAsync()`**.

By using `ClientCascade`, we can remove a permission from a device's collection, and EF Core will understand that it needs to delete the permission from the database without violating foreign key constraints.

### Updated Relationship Configuration

```csharp
public class DeviceEntityTypeConfiguration : IEntityTypeConfiguration<Device>
{
    public void Configure(EntityTypeBuilder<Device> builder)
    {
        builder.HasKey(d => d.Id);
        builder.Property(d => d.Name).IsRequired();
        
        builder.HasMany(d => d.Permissions)
            .WithOne()
            .HasForeignKey(p => p.Id)
            .OnDelete(DeleteBehavior.ClientCascade);
    }
}
```

### Refactored Code

#### After Applying `ClientCascade`

```csharp
public sealed class RemovePermissionsHandlerAfter
{
    private readonly DataContext _dataContext;

    public RemovePermissionsHandlerAfter(DataContext dataContext)
    {
        _dataContext = dataContext;
    }

    public async Task Handle(int deviceId, int permissionId, CancellationToken cancellationToken)
    {
        Device device = await _dataContext.Devices
            .Include(d => d.Permissions)
            .FirstOrDefaultAsync(d => d.Id == deviceId, cancellationToken);
        
        if (device is null) return;
        
        device.RemovePermission(permissionId);
        await _dataContext.SaveChangesAsync(cancellationToken);
    }
}
```

Now, we're removing the permission directly from the device's collection, adhering to object-oriented principles, and EF Core handles the deletion upon saving changes.

## Important Considerations

- **Database Behavior**: Always verify how your database interprets delete behaviors. Misalignment between EF Core configurations and database capabilities can lead to unexpected errors.
- **Foreign Key Recreation**: Be aware that EF Core might drop and recreate foreign keys during migrations, even if they appear identical. This can have implications for database deployments and should be managed carefully.

## Conclusion

By understanding and correctly applying `DeleteBehavior.ClientCascade`, we can write cleaner, more maintainable code that aligns with object-oriented principles. This approach allows us to manage entity relationships without tightly coupling our domain logic to the data access layer.

### Stay Tuned for More!

This is just the beginning of our "Mastering EF Core" series. In upcoming articles, we'll dive deeper into real-life scenarios and challenges we've faced, sharing insights and solutions that have expanded our team's knowledge about proper object-oriented programming with EF Core.

### Join the Conversation

Have you encountered similar challenges with EF Core or entity relationships? Share your experiences or solutions in the comments below! Developing a deep understanding of the code you write is essential to building efficient, reliable applications. If you've found unique solutions or hit roadblocks with EF Core’s delete behaviors, let us know!

**By embracing these principles and thoroughly understanding your tools, you can create robust applications that stand the test of time. We hope this article has provided valuable insights and look forward to sharing more in our "Mastering EF Core" series.**

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
