# `@recurse` directive

The `@recurse` directive allows you to traverse an edge multiple times, following the same relationship repeatedly. This is useful for hierarchical or graph-structured data like file systems, organizational charts, or social networks.

## Basic usage

The `@recurse` directive is applied to an edge and requires a `depth` parameter:

```graphql
{
    User {
        username @output
        
        follows @recurse(depth: 3) {
            followed_username: username @output
        }
    }
}
```

This query finds:
- Users directly followed by the starting user (depth 1)
- Users followed by those users (depth 2)
- Users followed by those users (depth 3)

## Depth parameter

The `depth` parameter specifies how many times to follow the edge:

- `depth: 1` - Direct connections only (same as no recursion)
- `depth: 2` - Direct connections and their connections
- `depth: N` - Up to N levels of connections

```graphql
{
    Number(max: 3) {
        value @output
        successor @recurse(depth: 3) {
            next: value @output
        }
    }
}
```

Starting from number 3:
- Depth 0: 3
- Depth 1: 4
- Depth 2: 5
- Depth 3: 6

Each starting vertex produces multiple result rows, one for each reachable vertex at each depth.

## Recursion with filters

You can apply filters within a recursive scope:

```graphql
{
    User {
        username @output
        
        follows @recurse(depth: 5) {
            username @filter(op: "has_prefix", value: ["$prefix"])
                     @output
        }
    }
}
```

The filter applies at every level of recursion. Only users matching the filter are followed further.

## Recursion on the same type

Recursion often follows edges that loop back to the same type:

```graphql
{
    Directory {
        name @output
        
        subdirectory @recurse(depth: 10) {
            subdir_name: name @output
        }
    }
}
```

This traverses a directory tree up to 10 levels deep.

## Combining recurse with other directives

### Recurse with tag

Tags can be used within recursive scopes:

```graphql
{
    User {
        username @tag(name: "original")
               @output
        
        follows @recurse(depth: 3) {
            username @filter(op: "!=", value: ["%original"])
                     @output
        }
    }
}
```

This finds all users within 3 degrees of connection who have a different username than the starting user.

### Recurse with optional

You can make a recursive edge optional:

```graphql
{
    Number(max: 5) {
        value @output
        
        predecessor @optional @recurse(depth: 3) {
            prev: value @output
        }
    }
}
```

For numbers without predecessors (like 0), the recursion stops and returns `null`.

### Recurse with output at multiple levels

Each level of recursion can produce outputs:

```graphql
{
    User {
        username @output
        
        follows @recurse(depth: 2) {
            # This outputs at each level of recursion
            friend: username @output
            
            posts {
                title @output
            }
        }
    }
}
```

## Recursion and type coercion

You can use type coercion within recursive scopes:

```graphql
{
    Number(max: 10) {
        value @output
        
        successor @recurse(depth: 5) {
            ... on Prime {
                prime_value: value @output
            }
        }
    }
}
```

This follows the successor chain but only outputs values that are prime numbers.

## Important considerations

### Performance

Recursion can generate many results. A starting set of N vertices with average branching factor B and depth D can produce up to N × B^D result rows. Use filters and reasonable depth values to limit results.

### No cycles with fold

The `@recurse` and `@fold` directives cannot be used on the same edge:

```graphql
# Invalid: cannot use both @recurse and @fold
{
    User {
        follows @recurse(depth: 3) @fold {
            username @output
        }
    }
}
```

### Schema restrictions

Not all edges support recursion. The schema must explicitly mark an edge as recursive-capable. Attempting to use `@recurse` on a non-recursive edge will result in an error.
