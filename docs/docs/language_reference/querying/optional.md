# `@optional` directive

The `@optional` directive is applied to edges to make them optional. It works similar to a SQL LEFT JOIN: if the edge doesn't exist, the query continues anyway, with `null` values for any outputs from that scope.

## Basic usage

Without `@optional`, if an edge doesn't exist, the entire result is filtered out:

```graphql
{
    User {
        username @output
        posts {
            title @output
        }
    }
}
```

This only returns users who have at least one post.

With `@optional`, users without posts are also included:

```graphql
{
    User {
        username @output
        posts @optional {
            title @output
        }
    }
}
```

Result when user has posts:
```json
{
    "username": "alice",
    "title": "My First Post"
}
```

Result when user has no posts:
```json
{
    "username": "bob",
    "title": null
}
```

## Multiple optional edges

You can have multiple `@optional` edges in the same query:

```graphql
{
    User {
        username @output
        
        posts @optional {
            post_title: title @output
        }
        
        follows @optional {
            followed_user: username @output
        }
    }
}
```

Each optional edge is evaluated independently. A user with no posts but some followers would return:
```json
{
    "username": "alice",
    "post_title": null,
    "followed_user": "bob"
}
```

## Nested optional edges

Optional edges can be nested:

```graphql
{
    User {
        username @output
        
        posts @optional {
            title @output
            
            author @optional {
                display_name @output
            }
        }
    }
}
```

If a user has no posts, both `title` and `display_name` are `null`. If a user has posts but the author data is missing, only `display_name` is `null`.

## Optional edges with filters

When you apply a `@filter` inside an `@optional` scope, the filter only applies if the edge exists:

```graphql
{
    User {
        username @output
        
        posts @optional {
            title @filter(op: "has_prefix", value: ["$prefix"])
                  @output
        }
    }
}
```

With variables `{"prefix": "Hello"}`:
- Users with posts starting with "Hello" return those posts
- Users with posts not starting with "Hello" return `null` for title
- Users with no posts also return `null` for title

## What if the edge exists but the other vertex doesn't match its `@filter`

The `@optional` directive will not pretend that it "didn't see" the edge. As soon as the edge is determined to exist for the given vertex, the remainder of the query proceeds as if no `@optional` was present.

In other words, if a user has posts, but none of them match the filter, the user will appear with `title: null`. The `@optional` only affects whether the edge *exists*, not whether vertices reached through that edge match their filters.

## Optional edges and tags

Tags defined within an `@optional` scope have special behavior in filters. If the optional edge doesn't exist, any filter using that tag is automatically satisfied:

```graphql
{
    User {
        username @output
        
        posts @optional {
            title @tag(name: "post_title")
        }
        
        display_name @filter(op: "!=", value: ["%post_title"])
                     @output
    }
}
```

For users without posts:
- The `@optional` scope produces no results
- The `@filter` on `display_name` is automatically satisfied (treated as true)
- The user is included in the results

For users with posts:
- The filter is evaluated normally
- Only users whose display name differs from their post title are included

See the [`@tag` directive](tag.md) documentation for more details.

## Using optional with fold

You can use `@optional` with `@fold` to aggregate optional data:

```graphql
{
    User {
        username @output
        
        posts @optional @fold {
            title @output
        }
    }
}
```

For users without posts, the `title` output is an empty list `[]` rather than `null`.

See the [`@fold` directive](fold.md) documentation for more details on aggregation.
