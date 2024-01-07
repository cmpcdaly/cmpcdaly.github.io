+++
title = 'Investigating Memory Leaks in EF Core'
date = 2020-12-12T00:00:00Z
draft = false
+++

## Background

For reasons beyond the scope of this post, I recently implemented an interface between a front-end web application and a MySQL database to facilitate dynamic filtering against different tables and columns (It's not as bonkers as sounds, model properties must be configured as filterable first!). The project requirements ruled out more conventional solutions like OData, and so I had to implement the bridge myself.

The implementation essentially takes two parameters; the model property name against which to filter, and the value to compare against. So a filter with the following payload:

```json
{
    "name": "UserId",
    "value": 840
}
```

Would filter results of the target entity to those belonging to the user 840. The front-end could combine any number of filters together, and mix and match between different comparison operators. Filters are specified using query strings (e.g. "api/things?**userId=%3D840**") and these are eventually serialised to SQL queries.

For demonstration, I've written an oversimplified version of the service responsible for generating the queries. The service takes a generic entity type, a property name and a value. It then generates an expression tree representing a predicate which compares the actual value of a property on an entity type against the value provided.

```csharp
public class PredicateBuilder
{
    public Expression<Func<TEntityType, bool>> Create<TEntityType>(string propertyName, object value)
    {
        // 1. Define the left parameter, which is a property accessor on the type
        var type = typeof(TEntityType);
        var parameter = Expression.Parameter(type);
        var property = type.GetProperty(propertyName) ?? throw new InvalidOperationException($"Property name {propertyName} does not exist on type {type.Name}");
        var accessor = Expression.Property(parameter, property);

        // 2. Define the right parameter, which is the value we're comparing against
        var constantExpression = Expression.Constant(value);
        var unboxedValue = Expression.Convert(constantExpression, accessor.Type);

        // 3. Define the expression
        var expression = Expression.Equal(accessor, unboxedValue);
        
        // 4. Return a comparison expression
        return Expression.Lambda<Func<TEntityType, bool>>(expression, parameter);
    }
}
```

The implementation worked great to begin with, but increased usage saw the applications memory begin to soar. After some investigation and memory profiling, I realised that the leak was being caused by the [CompiledQueryCache](https://github.com/dotnet/efcore/blob/release/2.1/src/EFCore/Query/Internal/CompiledQueryCache.cs) which is [used by EF Core to store SQL queries which have been compiled from Expression Trees](https://learn.microsoft.com/en-us/ef/core/querying/how-query-works#the-life-of-a-query) filling up beyond the capacity of the host.

So, let's figure out what's going on! For the purpose of this post, let's pretend that we have a **DbContext** with a **'Thing'** entity mapped to a **'things'** table, and each **'Thing'** has a **'UserId'** property which is mapped to a **'user_id'** column in that table.

```csharp
public class MyContext : DbContext
{
    public DbSet<Thing> Things { get; set; }
}

[Table("things")]
public class Thing
{
    [Column("user_id")]
    public int UserId { get; set; }
}
```

Now, let's use the **PredicateBuilder** class to generate a predicate expression, filter the **Things** collection by the expression, and investigate the output. The following:

```csharp
var predicate = predicateBuilder.Create<Thing>("UserId", 820);
var result = await myContext.Things.Where(predicate).ToListAsync();
```

Produces the following query:

```sql
SELECT `t`.`user_id`
FROM `things` AS `t`
WHERE `t`.`user_id` = 820;
```

Can you see the issue yet? Let's try filtering again, but this time providing the user id of 851:

```sql
SELECT `t`.`user_id`
FROM `things` AS `t`
WHERE `t`.`user_id` = 851;
```

The problem is that the queries aren't being [parameterised](https://dev.mysql.com/doc/connector-net/en/connector-net-tutorials-parameters.html)! For the same query with different values, EF Core generates two separate queries. This means that cache lookups for compiled queries will fail unless the values are exactly the same.

So what does this mean for my application? Well, instead of generating and caching one query per combination of filters, we're actually storing one entry per distinct value per filter! This becomes even more complex when users combine different filters together, and the cache grows exponentially. If we're being pedantic, you might not technically consider this a memory 'leak', but rather just really inefficient code.

To give a little more context, the real implementation of this allows queries to be applied against tens of columns for any given entity, and comparisons are not just limited to equals!

## The Problem

The expression generated by the service is essentially the same as one that might be generated at compile time with an in-line constant, like this:

```csharp
var result = await myContext.Things.Where(t => t.UserId == 820).ToListAsync();
```

But this code wouldn't be much use to us in a real-world scenario (unless your user ID is 820). Instead, we might write something that captures a function parameter and passes it to the expression predicate, like this:

```csharp
public async Task<IReadOnlyCollection<Thing>> GetForUserId(int userId)
{
    var result = await myContext.Things.Where(t => t.UserId == userId).ToListAsync();
    return result;
}
```

This second implementation actually gives us a parameterised SQL query, which looks like this:

```sql
SELECT `t`.`user_id`
FROM `things` AS `t`
WHERE `t`.`user_id` = @__p_0;
```

The problem here is that EF can't parameterise our query, and why would it? EF just sees a comparison against a constant, and spits it out into a SQL constant.

## The Solution

The solution is to somehow provide the **'UserId'** variable in a closure when building the expression tree, which will generate an expression that looks similar to the compile-time one that generates a parameterised query above.

I found the [answer on how to do this](https://stackoverflow.com/questions/14585184/issue-with-closure-variable-capture-in-c-sharp-expression/14586368#14586368) from one of the C# language design team members, who explained that we can use a lambda to hoist the value as part of an expression. In practice, it looks like this:

```csharp
Expression<Func<object>> hoistedValue = () => value;
```

So, if we replace the Constant expression with the body of this lambda, our implementation now looks like this:

```csharp
public class PredicateBuilder
{
    public Expression<Func<TEntityType, bool>> Create<TEntityType>(string propertyName, object value)
    {
        // 1. Define the left parameter, which is a property accessor on the type
        var type = typeof(TEntityType);
        var parameter = Expression.Parameter(type);
        var property = type.GetProperty(propertyName) ?? throw new InvalidOperationException($"Property name {propertyName} does not exist on type {type.Name}");
        var accessor = Expression.Property(parameter, property);

        // 2. Define the right parameter, which is the value we're comparing against
        Expression<Func<object>> hoistedValue = () => value;
        var unboxedValue = Expression.Convert(hoistedValue.Body, accessor.Type);

        // 3. Define the expression
        var expression = Expression.Equal(accessor, unboxedValue);
        
        // 4. Return a comparison expression
        return Expression.Lambda<Func<TEntityType, bool>>(expression, parameter);
    }
}
```

And now that the constant is parameterised, the memory leak is fixed! All queries against the same set of parameters but with different values will use the same cached query, improving the performance at the same time!

## Additional Improvements

Something else to note is that EF won't perform any optimizations to normalise queries which are logically the same but structurally different. For example, the following queries:

```csharp
var result1 = await myContext.Things.Where(t => t.UserId == 820 && t.SomethingElse == "Foo").ToListAsync();
var result2 = await myContext.Things.Where(t => t.SomethingElse == "Foo" && t.UserId == 820).ToListAsync();
```

Will produce two different SQL queries, even if we parameterised the int and the string. A really simple optimisation I made to the solution was to apply the filters in alphabetical order by property name. This ensured that the same combination of filters applied in any order would produce a single SQL query.