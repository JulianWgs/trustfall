# `@fold` directive

The `@fold` directive aggregates multiple related items into lists. It's applied to edges to collect all vertices reached through that edge into a single result row, rather than creating separate rows for each vertex.

## Basic usage

Without `@fold`, traversing an edge creates one result row per vertex:

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

For a user with 3 posts, this produces 3 rows:
```json
{"username": "alice", "title": "Post 1"}
{"username": "alice", "title": "Post 2"}
{"username": "alice", "title": "Post 3"}
```

With `@fold`, all posts are aggregated into a single row:

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
    "title": ["Post 1", "Post 2", "Post 3"]
}
```

## Empty folds

If no vertices are reached through a folded edge, the result is an empty list:

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

For a user with no posts:
```json
{
    "username": "bob",
    "title": []
}
```

## Multiple outputs in a fold

You can output multiple properties within a fold:

```graphql
{
    User {
        username @output
        posts @fold {
            title @output
            created_at @output
        }
    }
}
```

Result:
```json
{
    "username": "alice",
    "title": ["Post 1", "Post 2"],
    "created_at": ["2024-01-01", "2024-01-15"]
}
```

The lists are parallel: `title[0]` and `created_at[0]` come from the same post.

## Filtering within folds

Filters within a fold limit which vertices are included in the aggregation:

```graphql
{
    User {
        username @output
        posts @fold {
            title @filter(op: "has_prefix", value: ["$prefix"])
                  @output
        }
    }
}
```

With variables `{"prefix": "Hello"}`, only posts with titles starting with "Hello" are included in the list.

## Fold with transform

The `@transform` directive can be used with `@fold` to compute aggregations:

```graphql
{
    User {
        username @output
        posts @fold @transform(op: "count") @output(name: "post_count")
    }
}
```

Result:
```json
{
    "username": "alice",
    "post_count": 3
}
```

Currently, `count` is the only supported transform operation. See the [`@transform` directive](transform.md) documentation for details.

## Filtering on transformed folds

You can filter based on the count of folded items:

```graphql
{
    User {
        username @output
        posts @fold 
              @transform(op: "count")
              @filter(op: ">=", value: ["$min_posts"])
              @output(name: "post_count")
    }
}
```

This only returns users with at least `$min_posts` posts.

## Nested folds

Folds can be nested to create multi-dimensional lists:

```graphql
{
    User {
        username @output
        posts @fold {
            title @output
            
            comments @fold {
                text @output
            }
        }
    }
}
```

Result:
```json
{
    "username": "alice",
    "title": ["Post 1", "Post 2"],
    "text": [
        ["Comment on Post 1", "Another comment on Post 1"],
        ["Comment on Post 2"]
    ]
}
```

The outer list corresponds to posts, and each inner list corresponds to comments on that post.

## Using tags from outside a fold

Tags defined outside a fold can be used within the fold:

```graphql
{
    User {
        username @tag(name: "author")
               @output
        
        posts @fold {
            title @output
            author {
                username @filter(op: "=", value: ["%author"])
            }
        }
    }
}
```

This verifies that each post's author matches the original user.

## Using tags from within a fold

Tags defined within a fold can only be used within that same fold, not outside it:

```graphql
{
    User {
        username @output
        
        posts @fold {
            title @tag(name: "post_title")
                  @output
            # Can use %post_title here
            subtitle @filter(op: "!=", value: ["%post_title"])
        }
        
        # Cannot use %post_title here - it's only available within the fold
    }
}
```

## Fold with optional

Combining `@fold` with `@optional` gives you an empty list when the edge doesn't exist:

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

For users without posts, `title` is `[]` (empty list) rather than `null`.

## Important considerations

### Cannot fold and recurse

The `@fold` and `@recurse` directives cannot be used on the same edge:

```graphql
# Invalid: cannot combine @fold and @recurse
{
    User {
        follows @fold @recurse(depth: 3) {
            username @output
        }
    }
}
```

### Performance implications

Folding aggregates all reachable vertices into memory. For edges with many connections, this can consume significant memory. Use filters within folds to limit the aggregated data when possible.

## Examples

### Count items without outputting them

```graphql
{
    User {
        username @output
        posts @fold @transform(op: "count") @filter(op: ">", value: ["$threshold"]) {
            # No @output directives needed when only counting
            __typename
        }
    }
}
```

This filters users who have more than `$threshold` posts, without outputting the posts themselves.

### Fold multiple edges

```graphql
{
    User {
        username @output
        
        posts @fold {
            post_title: title @output
        }
        
        follows @fold {
            followed: username @output
        }
    }
}
```

Result:
```json
{
    "username": "alice",
    "post_title": ["Post 1", "Post 2"],
    "followed": ["bob", "charlie"]
}
```
