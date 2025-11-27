# `@transform` directive

The `@transform` directive applies transformations to values. It's most commonly used with `@fold` to perform aggregations, but can also be used on individual values.

## Count operation

Currently, the only supported operation is `count`, which counts the number of items in a list produced by `@fold`.

### Basic counting

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

### Counting with filters

You can filter before counting to count only items matching certain criteria:

```graphql
{
    User {
        username @output
        posts @fold 
              @transform(op: "count")
              @output(name: "popular_post_count")
        {
            likes @filter(op: ">", value: ["$min_likes"])
        }
    }
}
```

This counts only posts with more than `$min_likes` likes.

## Filtering on counts

Transform is commonly used with `@filter` to filter based on aggregated counts:

```graphql
{
    User {
        username @output
        posts @fold 
              @transform(op: "count")
              @filter(op: ">=", value: ["$threshold"])
              @output(name: "post_count")
    }
}
```

This only returns users who have at least `$threshold` posts.

## Count without output

You can use transform with filter without outputting the count:

```graphql
{
    User {
        username @output
        posts @fold 
              @transform(op: "count")
              @filter(op: ">", value: ["$min_posts"])
        {
            # No outputs needed - we just need to verify the count
            __typename
        }
    }
}
```

This filters out users with `$min_posts` or fewer posts, without outputting the actual count.

## Multiple transforms in one query

You can use multiple transforms in different parts of a query:

```graphql
{
    User {
        username @output
        
        posts @fold 
              @transform(op: "count")
              @output(name: "post_count")
        
        follows @fold 
                @transform(op: "count")
                @output(name: "following_count")
    }
}
```

Result:
```json
{
    "username": "alice",
    "post_count": 5,
    "following_count": 12
}
```

## Transform with nested folds

When using nested folds, you can transform at each level:

```graphql
{
    User {
        username @output
        posts @fold 
              @transform(op: "count")
              @filter(op: ">", value: ["$min_posts"])
              @output(name: "post_count")
        {
            title @output
            
            comments @fold 
                     @transform(op: "count")
                     @output(name: "comment_count")
        }
    }
}
```

This returns users with more than `$min_posts` posts, and for each post, counts its comments.

## Using transform with filters inside folds

The transform operates on the filtered result:

```graphql
{
    User {
        username @output
        posts @fold 
              @transform(op: "count")
              @output(name: "recent_post_count")
        {
            created_at @filter(op: ">=", value: ["$since_date"])
        }
    }
}
```

This counts only posts created on or after `$since_date`.

## Comparison operators with counts

All standard comparison operators work with counts:

```graphql
# Equal to
posts @fold @transform(op: "count") @filter(op: "=", value: ["$exact_count"])

# Not equal to
posts @fold @transform(op: "count") @filter(op: "!=", value: ["$not_count"])

# Greater than
posts @fold @transform(op: "count") @filter(op: ">", value: ["$min_count"])

# Greater than or equal
posts @fold @transform(op: "count") @filter(op: ">=", value: ["$min_count"])

# Less than
posts @fold @transform(op: "count") @filter(op: "<", value: ["$max_count"])

# Less than or equal
posts @fold @transform(op: "count") @filter(op: "<=", value: ["$max_count"])

# One of (in a set)
posts @fold @transform(op: "count") @filter(op: "one_of", value: ["$valid_counts"])
```

## Common patterns

### Find items with no connections

```graphql
{
    User {
        username @output
        posts @fold @transform(op: "count") @filter(op: "=", value: ["$zero"]) {
            __typename
        }
    }
}
```

With `{"zero": 0}`, this finds users with no posts.

### Find items with at least one connection

```graphql
{
    User {
        username @output
        posts @fold @transform(op: "count") @filter(op: ">", value: ["$zero"]) {
            title @output
        }
    }
}
```

This filters out users with no posts.

### Count range filtering

```graphql
{
    User {
        username @output
        follows @fold 
                @transform(op: "count")
                @filter(op: ">=", value: ["$min"])
                @filter(op: "<=", value: ["$max"])
                @output(name: "following_count")
    }
}
```

This finds users following between `$min` and `$max` other users (inclusive).

## Important notes

### Transform requires fold

The `@transform(op: "count")` operation requires `@fold` to be present on the same edge:

```graphql
# Invalid: transform without fold
{
    User {
        posts @transform(op: "count") {
            title @output
        }
    }
}

# Valid: transform with fold
{
    User {
        posts @fold @transform(op: "count") @output(name: "count")
    }
}
```

### Future operations

Currently, only `count` is supported, but the `@transform` directive is designed to support additional operations in the future, such as `sum`, `avg`, `min`, and `max`.
