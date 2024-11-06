---
layout: post
comments: true
title:  "Mastering EF Core: Managing Relationships Without Data Context Dependence (Part 1)"
date:   2024-11-05 10:00:00 +0100
categories: [ef-core-series]
tags: [dotnet, ef-core]
---

**Welcome to the first chapter of my "Mastering EF Core" series!** 

This series will dive into real-life scenarios and lessons I’ve learned while writing clean, object-oriented code using Entity Framework (EF) Core. My goal is to share with you practical insights that can help you unlock the full potential of EF Core in your applications.

## Introduction

Entity Framework Core is a powerful ORM that facilitates data access in .NET applications. In this article, we'll explore how to manage entity relationships without relying on the Entity Framework `DbContext` within your domain logic. I will emphasize the importance of understanding `OnDelete` behaviors and how they can impact your application's behavior.


## The Scenario: Devices and Permissions

Imagine we're building an application that manages devices and their permissions. While our example uses "Device" and "Permission," the concepts apply broadly to any domain involving complex relationships.

### Tech Stack

This example is designed for applications using .NET 8, EF Core, and either Azure SQL/SQL Server as the database.

### Entity Definitions

Here's how our entities look:

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
- We aim to remove a permission using object-oriented principles without direct manipulation our `DbContext` implementation.

## The Challenge: Constraints and Delete Behaviors

When attempting to remove a permission from a device using `device.RemovePermission(permissionId)`, we encounter a foreign key constraint error upon calling `_dbContext.SaveChangesAsync()`. This error arises due to the way we've configured the relationship in EF Core and how our database handles delete behaviors.

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

Using `DeleteBehavior.Restrict` means that EF will prevent the deletion of a `Permission` if it would violate a foreign key constraint with `Device`. This setup requires us to manage deletions explicitly, often leading to less clean code that directly interacts with `DbContext`.

### The Problem in Code

#### Before Refactoring

```csharp
public sealed class RemovePermissionsHandlerBefore
{
    private readonly DataContext _dbContext;

    public RemovePermissionsHandlerBefore(DataContext dataContext)
    {
        _dbContext = dataContext;
    }

    public async Task Handle(int deviceId, int permissionId, CancellationToken cancellationToken)
    {
        Device device = await _dbContext.Devices
            .Include(d => d.Permissions)
            .FirstOrDefaultAsync(d => d.Id == deviceId, cancellationToken);
        
        if (device is null) return;
        
        Permission permission = device.Permissions.FirstOrDefault(p => p.Id == permissionId);
        
        if (permission is null) return;
        
        // We do not want to manually remove related Permission entities from the database
        _dbContext.Permissions.Remove(permission);
        
        await _dbContext.SaveChangesAsync(cancellationToken);
    }
}
```

This code directly manipulates the `DbContext` to remove the `Permission`, coupling our domain logic with the data access layer.

## The Solution: Understanding and using ClientCascade behavior

To achieve a cleaner, more object-oriented approach, we need to adjust our understanding of delete behaviors and how our database handles them.

### Evaluating Database Support

Before making changes, it’s important to check how your database handles `NoAction` and `Restrict`. Not all databases differentiate between these settings. Azure SQL/SQL Server treats `Restrict` and `NoAction` identically. Even if we configure it as `Restrict` in EF Core, on the database level - is creating non-cascading same foreign key as for `NoAction`.

Knowing your database’s capabilities is key, as it affects whether your delete behaviors will work as expected. If the database doesn’t support a specific behavior, you could face unexpected constraint errors or data inconsistencies on the application side.

### Understanding `DeleteBehavior.ClientCascade`

`DeleteBehavior.ClientCascade` is an EF Core setting that:

- **Enables EF to violate non-cascading FK constraints on application level**.
- **Creates non-cascading FK on the database level**.
- **Allows us to remove entities from collections and have EF Core handle the deletions upon `SaveChangesAsync()`**.

By using `ClientCascade`, we can remove a permission from a device's collection, and EF Core will understand that it needs to delete the permission from the database.

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
    private readonly DataContext _dbContext;

    public RemovePermissionsHandlerAfter(DataContext dataContext)
    {
        _dbContext = dataContext;
    }

    public async Task Handle(int deviceId, int permissionId, CancellationToken cancellationToken)
    {
        Device device = await _dbContext.Devices
            .Include(d => d.Permissions)
            .FirstOrDefaultAsync(d => d.Id == deviceId, cancellationToken);
        
        if (device is null) return;
        
        // Now, we can remove the permission directly on Device without manipulating the DbContext
        device.RemovePermission(permissionId);
        
        await _dbContext.SaveChangesAsync(cancellationToken);
    }
}
```

Now, we're removing the permission directly from the device's collection, adhering to object-oriented principles. EF Core changer tracker updates the removed `Permission` state as `Deleted` and removes it from the database upon calling `_dbContext.SaveChangesAsync()`.

## Important Considerations

- **Database Behavior**: Always verify how your database interprets delete behaviors. Misalignment between EF Core configurations and database capabilities can lead to unexpected errors.
- **Foreign Key Recreation**: Be aware that EF Core might drop and recreate foreign keys during migrations, even if they appear identical. This can have implications for database deployments and should be managed carefully.

## Conclusion

By understanding and correctly applying `DeleteBehavior.ClientCascade`, we can write cleaner, more maintainable code that aligns with object-oriented principles. This approach allows us to manage entity relationships without tightly coupling our domain logic to the data access layer.

### Stay Tuned for More!

This is just the beginning of my "Mastering EF Core" series. In future articles, I’ll dig into real-world challenges and scenarios I've tackled, sharing insights and solutions that have broadened my understanding of solid object-oriented programming with EF Core.

### Join the Conversation

Have you faced similar issues with EF Core or entity relationships? Share your thoughts or experiences in the comments below!

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
