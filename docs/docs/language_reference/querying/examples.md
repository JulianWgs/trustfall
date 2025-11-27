# Query Examples

This page provides comprehensive examples of Trustfall queries demonstrating all features of the query language.

## Basic queries

### Simple property selection

```graphql
{
    User {
        username @output
        email @output
    }
}
```

Returns all users with their username and email.

### Using entrypoint parameters

```graphql
{
    FindUser(username: "alice") {
        username @output
        display_name @output
    }
}
```

Finds a specific user by username.

## Filtering

### Basic equality filter

```graphql
{
    User {
        username @output
        age @filter(op: ">=", value: ["$min_age"]) @output
    }
}
```

With variables: `{"min_age": 18}`

### String filters

```graphql
{
    Post {
        title @filter(op: "has_prefix", value: ["$prefix"])
              @output
        content @filter(op: "has_substring", value: ["$keyword"])
                @output
    }
}
```

With variables: `{"prefix": "Introduction", "keyword": "Trustfall"}`

### Regex filter

```graphql
{
    User {
        email @filter(op: "regex", value: ["$email_pattern"])
              @output
    }
}
```

With variables: `{"email_pattern": ".*@example\\.com$"}`

### Multiple filters (AND logic)

```graphql
{
    Post {
        likes @filter(op: ">=", value: ["$min_likes"])
              @filter(op: "<=", value: ["$max_likes"])
              @output
        title @output
    }
}
```

### Null checking

```graphql
{
    User {
        username @output
        bio @filter(op: "is_not_null") @output
    }
}
```

Returns only users who have set a bio.

### List membership

```graphql
{
    Post {
        category @filter(op: "one_of", value: ["$categories"])
                 @output
        title @output
    }
}
```

With variables: `{"categories": ["tech", "science", "programming"]}`

## Traversing edges

### Basic edge traversal

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

Returns each user with each of their posts (multiple rows per user).

### Multiple edge traversals

```graphql
{
    User {
        username @output
        posts {
            title @output
            comments {
                text @output
            }
        }
    }
}
```

## Tags and cross-field filtering

### Comparing two fields

```graphql
{
    User {
        username @tag(name: "uname")
               @output
        display_name @filter(op: "=", value: ["%uname"])
                     @output
    }
}
```

Finds users whose display name equals their username.

### Using tags across edges

```graphql
{
    User {
        username @tag(name: "author")
               @output
        posts {
            title @output
            author {
                username @filter(op: "=", value: ["%author"])
            }
        }
    }
}
```

Verifies that posts belong to the correct author.

### Multiple tags

```graphql
{
    Post {
        title @tag(name: "post_title") @output
        author {
            username @tag(name: "author_name")
                     @output
            display_name @filter(op: "!=", value: ["%author_name"])
                         @filter(op: "!=", value: ["%post_title"])
                         @output
        }
    }
}
```

## Folding (aggregation)

### Basic fold

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

Returns one row per user with all post titles in a list.

### Counting with fold and transform

```graphql
{
    User {
        username @output
        posts @fold @transform(op: "count") @output(name: "post_count")
    }
}
```

### Filtering on counts

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

With variables: `{"min_posts": 5}`

Returns only users with at least 5 posts.

### Multiple folds

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

### Nested folds

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

Creates a 2D list structure.

### Fold with filtering

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

Folds only posts matching the filter.

## Optional edges

### Basic optional

```graphql
{
    User {
        username @output
        posts @optional {
            latest_post: title @output
        }
    }
}
```

Returns users even if they have no posts (with `null` for `latest_post`).

### Optional with fold

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

Users without posts get an empty list instead of `null`.

### Nested optional

```graphql
{
    User {
        username @output
        posts @optional {
            title @output
            featured_comment @optional {
                text @output
            }
        }
    }
}
```

### Optional with tags

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

If the user has no posts, the filter is automatically satisfied.

## Recursion

### Basic recursion

```graphql
{
    User {
        username @output
        follows @recurse(depth: 3) {
            friend: username @output
        }
    }
}
```

Follows the "follows" relationship up to 3 levels deep.

### Recursion with filters

```graphql
{
    Directory {
        name @output
        subdirectory @recurse(depth: 10) {
            name @filter(op: "has_prefix", value: ["$prefix"])
                 @output
        }
    }
}
```

### Recursion with tags

```graphql
{
    User {
        username @tag(name: "origin")
               @output
        follows @recurse(depth: 5) {
            username @filter(op: "!=", value: ["%origin"])
                     @output
        }
    }
}
```

Finds friends of friends, excluding the original user.

## Type coercion

### Basic type coercion

```graphql
{
    Number(max: 20) {
        value @output
        ... on Prime {
            is_prime: value @output
        }
    }
}
```

Outputs `is_prime` only for prime numbers.

### Multiple type coercions

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
    }
}
```

### Type coercion with fold

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

Folds only bugs, not other issue types.

### Type coercion with recursion

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

Follows successors but only outputs primes.

## Combining multiple features

### Complex filtering and aggregation

```graphql
{
    User {
        username @tag(name: "author")
               @output
        
        posts @fold 
              @transform(op: "count")
              @filter(op: ">=", value: ["$min_posts"])
              @output(name: "post_count")
        {
            title @filter(op: "has_substring", value: ["$keyword"])
            author {
                username @filter(op: "=", value: ["%author"])
            }
        }
    }
}
```

Finds users with at least `$min_posts` posts containing a keyword.

### Nested queries with multiple features

```graphql
{
    User {
        username @tag(name: "original_user")
               @output
        
        follows @recurse(depth: 2) {
            username @output
            
            posts @fold @transform(op: "count") @output(name: "post_count")
            
            follows @optional @fold {
                mutual: username @filter(op: "=", value: ["%original_user"])
                                 @output
            }
        }
    }
}
```

Complex social network analysis: finds friends of friends and identifies mutual connections.

### Advanced aggregation with type coercion

```graphql
{
    Repository {
        name @output
        
        issues @fold {
            ... on Bug {
                priority @filter(op: "one_of", value: ["$high_priority"])
                         @output
            }
        }
        
        bugs: issues @fold 
                     @transform(op: "count")
                     @output(name: "bug_count")
        {
            ... on Bug {
                severity @filter(op: ">=", value: ["$min_severity"])
            }
        }
    }
}
```

## Output naming

### Implicit names from properties

```graphql
{
    User {
        username @output
        email @output
    }
}
```

Output fields: `username`, `email`

### Explicit output names

```graphql
{
    User {
        username @output(name: "user_id")
        email @output(name: "user_email")
    }
}
```

Output fields: `user_id`, `user_email`

### Aliases

```graphql
{
    User {
        user_id: username @output
        user_email: email @output
    }
}
```

Output fields: `user_id`, `user_email`

### Scope prefixes

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

Output fields: `username`, `authored_title`

### Nested scope prefixes

```graphql
{
    User {
        username @output
        
        authored_: posts {
            post_: author {
                name @output
            }
        }
    }
}
```

Output fields: `username`, `authored_post_name`

## Real-world examples

### Social media query

```graphql
{
    User {
        username @tag(name: "current_user")
               @output
        followers @fold @transform(op: "count") @output(name: "follower_count")
        
        posts @fold 
              @transform(op: "count")
              @filter(op: ">=", value: ["$min_posts"])
              @output(name: "post_count")
        {
            created_at @filter(op: ">=", value: ["$since_date"])
            likes @filter(op: ">=", value: ["$min_likes"])
            
            author {
                username @filter(op: "=", value: ["%current_user"])
            }
        }
    }
}
```

Finds active users with recent popular posts.

### Repository analysis

```graphql
{
    Repository {
        name @output
        stars @filter(op: ">=", value: ["$min_stars"]) @output
        
        contributors @fold @transform(op: "count") @output(name: "contributor_count")
        
        issues @fold {
            ... on Bug {
                status @filter(op: "=", value: ["$status"])
                severity @output
            }
        }
        
        pull_requests @fold 
                      @transform(op: "count")
                      @filter(op: ">", value: ["$zero"])
                      @output(name: "pr_count")
        {
            merged @filter(op: "=", value: ["$merged_status"])
        }
    }
}
```

Analyzes repository health metrics.

### Hierarchical data traversal

```graphql
{
    Directory(path: "/home") {
        name @output
        
        subdirectory @recurse(depth: 5) {
            name @output
            size @output
            
            files @fold @transform(op: "count") @output(name: "file_count")
            
            subdirectory @fold @transform(op: "count") @output(name: "subdir_count")
        }
    }
}
```

Explores directory structure up to 5 levels deep.
