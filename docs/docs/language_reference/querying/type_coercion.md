# Type coercion

Type coercion allows you to narrow down a vertex to a more specific type. This is similar to type casting or "instanceof" checks in programming languages and is particularly useful when working with interfaces and type hierarchies.

## Basic usage

Type coercion is performed using GraphQL's inline fragment syntax with `... on TypeName`:

```graphql
{
    Number(max: 10) {
        value @output
        
        ... on Prime {
            # This scope only executes for Prime numbers
            is_prime: value @output
        }
    }
}
```

Results include `is_prime` only for prime numbers:
```json
{"value": 2, "is_prime": 2}
{"value": 3, "is_prime": 3}
{"value": 4}  // not prime, no is_prime field
{"value": 5, "is_prime": 5}
```

## Filtering by type

Type coercion acts as an implicit filter. Only vertices that are instances of the specified type continue in the coerced scope:

```graphql
{
    Number(max: 20) {
        ... on Composite {
            # Only composite (non-prime) numbers
            value @output
            
            primeFactor @fold {
                factor: value @output
            }
        }
    }
}
```

This query only processes composite numbers and outputs their prime factors.

## Multiple type coercions

You can have multiple type coercions in the same query to handle different types separately:

```graphql
{
    SearchResults(query: "example") {
        ... on User {
            type: __typename @output
            username @output
        }
        
        ... on Post {
            type: __typename @output
            title @output
        }
        
        ... on Comment {
            type: __typename @output
            text @output
        }
    }
}
```

Each search result is processed according to its actual type.

## Type coercion on edges

Type coercion can be applied after traversing an edge:

```graphql
{
    Repository {
        name @output
        
        issue {
            ... on Bug {
                severity @output
                stack_trace @output
            }
            
            ... on FeatureRequest {
                priority @output
                estimated_effort @output
            }
        }
    }
}
```

This processes bugs and feature requests differently.

## Type coercion with interfaces

When a vertex implements an interface, you can coerce from the interface to specific implementing types:

```graphql
{
    Webpage {  # Webpage is an interface
        url @output
        
        ... on User {
            # User-specific fields
            username @output
        }
        
        ... on Post {
            # Post-specific fields
            title @output
        }
    }
}
```

## Nested type coercions

Type coercions can be nested:

```graphql
{
    Number(max: 100) {
        ... on Composite {
            value @output
            
            primeFactor {
                ... on LargePrime {
                    # Only large prime factors
                    size_category @output
                }
            }
        }
    }
}
```

## Type coercion with filters

Combining type coercion with filters provides fine-grained control:

```graphql
{
    Number(max: 50) {
        ... on Prime {
            value @filter(op: ">", value: ["$min"])
                  @output
        }
    }
}
```

This finds primes greater than `$min`.

## Type coercion with optional

Type coercion can be combined with `@optional`:

```graphql
{
    User {
        username @output
        
        posts @optional {
            ... on FeaturedPost {
                featured_title: title @output
            }
        }
    }
}
```

If a user has no featured posts (either no posts at all, or no posts of the FeaturedPost type), `featured_title` is `null`.

## Type coercion with fold

Type coercion within a fold only aggregates vertices of the specified type:

```graphql
{
    Repository {
        name @output
        
        issues @fold {
            ... on Bug {
                bug_title: title @output
            }
        }
    }
}
```

This creates a list of only bug titles, excluding other issue types.

## Type coercion with recurse

Type coercion can be used within recursive traversals:

```graphql
{
    Number(max: 5) {
        value @output
        
        successor @recurse(depth: 10) {
            ... on Prime {
                prime: value @output
            }
        }
    }
}
```

This follows the successor chain but only outputs prime numbers encountered along the way.

## Accessing type information

The special field `__typename` returns the actual type of a vertex:

```graphql
{
    Number(max: 10) {
        type: __typename @output
        value @output
    }
}
```

Result:
```json
{"type": "Zero", "value": 0}
{"type": "Prime", "value": 2}
{"type": "Prime", "value": 3}
{"type": "Composite", "value": 4}
```

## Type coercion rules

### Valid coercions

Type coercion is only valid when:
1. Coercing from an interface to a type that implements it
2. Coercing from a type to one of its subtypes (if the schema has inheritance)

### Invalid coercions

Type coercion is invalid when:
1. Coercing to an unrelated type
2. Coercing from a concrete type to a parent type (use the parent type directly instead)
3. Coercing to a non-existent type

```graphql
# Invalid: User and Post are unrelated types
{
    User {
        ... on Post {  # Error!
            title @output
        }
    }
}
```

## Common patterns

### Process different types differently

```graphql
{
    SearchResults(query: "trustfall") {
        ... on Repository {
            repo_name: name @output
            stars @output
        }
        
        ... on User {
            username @output
            bio @output
        }
    }
}
```

### Filter to a specific subtype

```graphql
{
    Shape {
        ... on Circle {
            # Only process circles
            radius @output
            area @output
        }
    }
}
```

### Conditional edges based on type

```graphql
{
    Vehicle {
        model @output
        
        ... on ElectricVehicle {
            battery {
                capacity @output
            }
        }
        
        ... on GasVehicle {
            engine {
                displacement @output
            }
        }
    }
}
```
