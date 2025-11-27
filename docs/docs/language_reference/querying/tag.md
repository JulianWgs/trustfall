# `@tag` directive

The `@tag` directive allows you to save a value from one part of the query and use it later in a `@filter` directive. This enables comparisons between different fields within the same query.

## Basic usage

Tags are created by applying `@tag` to a property, and then referenced in `@filter` directives using the `%` prefix:

```graphql
{
    User {
        username @tag(name: "user")
        display_name @filter(op: "=", value: ["%user"])
        username @output
    }
}
```

This query finds users whose `display_name` exactly matches their `username`.

## Implicit tag names

If you don't specify a name, the tag uses the property name or alias:

```graphql
{
    User {
        username @tag
        display_name @filter(op: "=", value: ["%username"])
        username @output
    }
}
```

This is equivalent to the previous example with explicit tag name.

## Using tags with aliases

When using an alias, the tag name is derived from the alias, not the property:

```graphql
{
    User {
        user: username @tag
        display_name @filter(op: "=", value: ["%user"])
        username @output
    }
}
```

## Tags across edges

Tags can be used across edge traversals to compare values at different levels:

```graphql
{
    User {
        username @tag(name: "author_name")
        
        posts {
            title @output
            author {
                username @filter(op: "=", value: ["%author_name"])
            }
        }
    }
}
```

This query gets posts where the post's author matches the original user (a self-referential check).

## Tags in optional scopes

Tags defined within an `@optional` scope have special behavior. When the optional edge doesn't exist, filters using that tag are automatically considered to pass:

```graphql
{
    User {
        username @output
        
        posts @optional {
            title @tag(name: "post_title")
        }
        
        display_name @filter(op: "!=", value: ["%post_title"])
    }
}
```

If the user has no posts, the filter on `display_name` is treated as true, and the user is included in results.

See the [`@optional` directive](optional.md) documentation for more details.

## Multiple tags

You can tag multiple values and use them together:

```graphql
{
    User {
        username @tag(name: "uname")
        display_name @tag(name: "dname")
        
        posts {
            title @filter(op: "=", value: ["%uname"])
                  @filter(op: "!=", value: ["%dname"])
                  @output
        }
    }
}
```

This finds posts where the title equals the author's username but differs from their display name.

## Scope and ordering

Tags must be defined before they are used in a `@filter`. The general rule is:
- A tag must appear earlier in the query than any filter that uses it
- Tags defined in parent scopes can be used in child scopes
- Tags defined in sibling scopes cannot reference each other

```graphql
# Valid: tag defined before use
{
    User {
        username @tag
        display_name @filter(op: "=", value: ["%username"])
    }
}

# Invalid: tag used before definition
{
    User {
        username @filter(op: "=", value: ["%display"])
        display_name @tag(name: "display")
    }
}
```
