# `@output` directive

The `@output` directive marks a property or field to be included in the query results. Every query must have at least one `@output` directive to return data.

## Basic usage

The simplest form of `@output` marks a property to be returned with its default name:

```graphql
{
    User {
        username @output
    }
}
```

This query returns a result set where each row has a `username` field.

## Implicit output name

When you use `@output` without any arguments, the output field name is determined implicitly from the property name or any alias you've given it:

```graphql
{
    User {
        username @output
        display_name @output
    }
}
```

Result:
```json
{
    "username": "alice",
    "display_name": "Alice Smith"
}
```

## Explicit output names

You can specify a custom output name using the `name` parameter:

```graphql
{
    User {
        username @output(name: "user_id")
        display_name @output(name: "name")
    }
}
```

Result:
```json
{
    "user_id": "alice",
    "name": "Alice Smith"
}
```

## Aliases

GraphQL-style aliases can be used to rename fields before applying `@output`:

```graphql
{
    User {
        user_id: username @output
        name: display_name @output
    }
}
```

Result:
```json
{
    "user_id": "alice",
    "name": "Alice Smith"
}
```

## Assigning aliases to a scope

You can assign an alias to an entire scope (edge traversal), which acts as a prefix for all outputs within that scope:

```graphql
{
    User {
        username @output
        
        authored_: posts {
            title @output
        }
    }
}
```

Result:
```json
{
    "username": "alice",
    "authored_title": "My First Post"
}
```

### Nesting aliased scopes

Aliased scopes can be nested, and their prefixes accumulate:

```graphql
{
    User {
        username @output
        
        authored_: posts {
            post_: author {
                username @output
            }
        }
    }
}
```

Result:
```json
{
    "username": "alice",
    "authored_post_username": "alice"
}
```

### Implicit vs explicit output names

There are three ways to name an output:

1. **Implicit from property**: `value @output` - uses the property name `value`
2. **Implicit from alias**: `alias: value @output` - uses the alias name `alias` (plus any scope prefixes)
3. **Explicit**: `value @output(name: "my_output")` - uses exactly `my_output`, ignoring aliases and prefixes

```graphql
{
    User {
        # Implicit from property
        username @output
        
        # Implicit from alias
        user_name: username @output
        
        # Explicit name (ignores any prefixes or aliases)
        username @output(name: "login")
    }
}
```

Result:
```json
{
    "username": "alice",
    "user_name": "alice",
    "login": "alice"
}
```

## Outputting from folded edges

When using `@output` inside a `@fold` directive, the result is a list of values:

```graphql
{
    User {
        username @output
        
        posts @fold {
            title @output
        }
    }
}
```

Result:
```json
{
    "username": "alice",
    "title": ["My First Post", "My Second Post", "Another Post"]
}
```

See the [`@fold` directive](fold.md) documentation for more details on aggregating data.
