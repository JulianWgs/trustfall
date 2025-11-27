# Edge Parameters

Edge parameters allow you to pass arguments when traversing an edge, enabling filtered or parameterized traversals defined by the schema.

## Basic usage

Edges can accept parameters as defined in the schema:

```graphql
{
    Directory {
        name @output
        
        # Traverse to files with .txt extension
        files(extension: "txt") {
            name @output
        }
    }
}
```

This traverses only to files matching the specified extension.

## Required vs optional parameters

The schema defines which parameters are required and which are optional:

```graphql
{
    User {
        username @output
        
        # Parameter is required
        posts(limit: 10) {
            title @output
        }
        
        # Parameter is optional (has default value)
        follows(active_only: true) {
            username @output
        }
    }
}
```

## Using variables in parameters

Edge parameters commonly use query variables:

```graphql
{
    Repository {
        name @output
        
        issues(state: "$issue_state") {
            title @output
        }
    }
}
```

With variables: `{"issue_state": "open"}`

## Multiple parameters

Edges can accept multiple parameters:

```graphql
{
    Post {
        title @output
        
        comments(min_likes: "$min", max_likes: "$max") {
            text @output
            likes @output
        }
    }
}
```

With variables: `{"min": 5, "max": 100}`

## Parameter types

Parameters can have different types as defined in the schema:

### String parameters

```graphql
{
    Directory {
        files(extension: "pdf") {
            name @output
        }
    }
}
```

### Integer parameters

```graphql
{
    User {
        posts(limit: 20) {
            title @output
        }
    }
}
```

### Boolean parameters

```graphql
{
    Repository {
        issues(include_closed: false) {
            title @output
        }
    }
}
```

### List parameters

```graphql
{
    Post {
        comments(categories: ["tech", "science"]) {
            text @output
        }
    }
}
```

## Nullable parameters

Parameters can be nullable, allowing `null` as a valid value:

```graphql
{
    User {
        # null means "no filter on creation date"
        posts(created_after: null) {
            title @output
        }
    }
}
```

## Default parameter values

If the schema defines a default value, you can omit the parameter:

```graphql
{
    User {
        # Uses default limit defined in schema
        posts {
            title @output
        }
        
        # Overrides default
        recent_posts: posts(limit: 5) {
            title @output
        }
    }
}
```

## Parameters vs filters

Edge parameters and `@filter` directives serve different purposes:

### Edge parameters
- Defined by the schema for specific edges
- May affect which edges exist or are traversed
- Applied at the edge traversal level

### Filters
- Generic filtering mechanism
- Applied to vertex properties
- Work on vertices reached through edges

```graphql
{
    Directory {
        # Edge parameter: limits traversal to .txt files
        files(extension: "txt") {
            # Filter: further filters to files larger than min_size
            size @filter(op: ">", value: ["$min_size"])
                 @output
            name @output
        }
    }
}
```

## Parameters with optional edges

Parameters work with `@optional` edges:

```graphql
{
    User {
        username @output
        
        posts(category: "tech") @optional {
            title @output
        }
    }
}
```

Users without tech posts are included with `null` for title.

## Parameters with fold

Parameters filter which items are included in the fold:

```graphql
{
    User {
        username @output
        
        posts(min_likes: 10) @fold {
            popular_post: title @output
        }
    }
}
```

Only posts with at least 10 likes are included in the folded list.

## Parameters with recurse

Parameterized edges can be used with recursion:

```graphql
{
    Directory {
        name @output
        
        subdirectory(max_depth: 3) @recurse(depth: 5) {
            name @output
        }
    }
}
```

The parameter applies at each level of recursion.

## Semantic differences with optional and recurse

When using edge parameters with `@optional` or `@recurse`, the parameters become part of the edge's existence condition:

### With @optional

```graphql
# Edge parameter approach
{
    Directory {
        files(extension: "txt") @optional {
            name @output
        }
    }
}
```

This is **different** from:

```graphql
# Filter approach
{
    Directory {
        files @optional {
            extension @filter(op: "=", value: ["$ext"])
            name @output
        }
    }
}
```

In the first case (parameter), if no .txt files exist, the result is `null`.
In the second case (filter), if files exist but none are .txt, the result is still `null`, but the edge is considered to exist.

### With @recurse

```graphql
# Edge parameter approach
{
    User {
        follows(active: true) @recurse(depth: 3) {
            username @output
        }
    }
}
```

Only follows active users at each level of recursion. An inactive user stops that path of recursion.

```graphql
# Filter approach
{
    User {
        follows @recurse(depth: 3) {
            active @filter(op: "=", value: ["$active"])
            username @output
        }
    }
}
```

Follows all users but only outputs those who are active. Recursion continues through inactive users.

## Common patterns

### Pagination

```graphql
{
    User {
        username @output
        
        posts(offset: "$offset", limit: "$limit") {
            title @output
        }
    }
}
```

With variables: `{"offset": 0, "limit": 20}`

### Date range filtering

```graphql
{
    Repository {
        name @output
        
        commits(since: "$start_date", until: "$end_date") {
            message @output
            timestamp @output
        }
    }
}
```

### Category filtering

```graphql
{
    Blog {
        title @output
        
        posts(category: "$category") @fold {
            post_title: title @output
        }
    }
}
```

### Combined with count

```graphql
{
    User {
        username @output
        
        posts(status: "published") 
              @fold 
              @transform(op: "count")
              @output(name: "published_count")
    }
}
```

## Important notes

### Schema definition required

Edge parameters must be defined in the schema. You cannot add arbitrary parameters to edges:

```graphql
# Invalid if 'foo' parameter is not defined in schema
{
    User {
        posts(foo: "bar") {  # Error!
            title @output
        }
    }
}
```

### Type safety

Parameter values must match the type defined in the schema:

```graphql
# Invalid: passing string when integer expected
{
    User {
        posts(limit: "10") {  # Error! Should be 10 not "10"
            title @output
        }
    }
}
```

### Required parameters must be provided

If a parameter is required (non-nullable without a default), it must be provided:

```graphql
# Invalid if 'category' is required
{
    Post {
        comments {  # Error! Missing required parameter
            text @output
        }
    }
}
```
