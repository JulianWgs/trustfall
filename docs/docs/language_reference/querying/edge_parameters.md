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

## Edge Parameters vs @filter: Key Differences

Understanding when to use edge parameters versus `@filter` directives is crucial for writing efficient and correct queries.

### Conceptual Difference

**Edge parameters** are part of the edge definition and control which edges are traversed in the first place:
- Applied **before** edge traversal
- Defined in the schema for specific edges
- Can prevent edges from being followed at all
- The adapter's `resolve_neighbors` method receives the parameter values

**@filter directives** filter vertices after they've been reached:
- Applied **after** edge traversal
- Generic mechanism that works on any property
- Vertices are reached first, then filtered
- The adapter still traverses the edge and returns all neighbors

### Simple Case: No @optional or @recurse

When not using `@optional` or `@recurse`, edge parameters and filters are **semantically equivalent** (but may differ in performance):

```graphql
# Using edge parameter
{
    Directory {
        files(extension: "txt") {
            name @output
        }
    }
}

# Using filter - semantically equivalent
{
    Directory {
        files {
            extension @filter(op: "=", value: ["$ext"])
            name @output
        }
    }
}
```

Both produce the same results, but performance may differ based on adapter implementation.

### When They Differ: @optional and @recurse

With `@optional` or `@recurse`, edge parameters and filters have **different semantics**:

#### With @optional

```graphql
# Edge parameter: considers edge non-existent if no matches
{
    Directory {
        files(extension: "txt") @optional {
            name @output
        }
    }
}
# Result: null if no .txt files exist

# Filter: edge exists, but all neighbors filtered out
{
    Directory {
        files @optional {
            extension @filter(op: "=", value: ["$ext"])
            name @output
        }
    }
}
# Result: null even if .pdf files exist (edge existed, but filtered)
```

The difference: edge parameters make the edge itself conditional on the predicate, while filters process vertices after the edge is traversed.

#### With @recurse

```graphql
# Edge parameter: predicate applies at each recursion level
{
    User {
        follows(active: true) @recurse(depth: 3) {
            username @output
        }
    }
}
# Only recurses through active users; stops at inactive ones

# Filter: predicate only applies to final results
{
    User {
        follows @recurse(depth: 3) {
            active @filter(op: "=", value: ["$active"])
            username @output
        }
    }
}
# Recurses through all users, but only outputs active ones
```

### When to Use Each

**Use edge parameters when**:
- The schema defines parameters for the edge
- You want to limit which edges are traversed (especially important for expensive operations)
- The filtering criterion is fundamental to the edge semantics
- Using `@optional` and you want the edge to be considered non-existent if criteria aren't met
- Using `@recurse` and you want the predicate to apply at each recursion level
- The adapter can optimize by not fetching filtered-out neighbors at all

**Use @filter when**:
- You need to filter based on properties that aren't edge parameters
- You need complex filtering conditions (multiple operators, tagged values)
- You want to filter after reaching vertices through the edge
- The filtering criterion is a query-time decision, not fundamental to the edge

**Use both when**:
- You want to combine edge-level filtering with additional vertex-level filtering
- Example: `posts(category: "tech")` to limit to tech posts, then `likes @filter(op: ">", value: ["$min"])` to further filter by popularity

### Performance Implications

#### Adapter-Level Optimization

Edge parameters enable **adapter-level optimization**:

```graphql
# Good: Adapter can query database with WHERE clause
{
    User {
        posts(status: "published") {
            title @output
        }
    }
}

# Adapter implementation:
# SELECT * FROM posts WHERE user_id = ? AND status = 'published'
```

With filters, the adapter may need to fetch all posts, then Trustfall filters them:

```graphql
# Less optimal: Adapter fetches all posts
{
    User {
        posts {
            status @filter(op: "=", value: ["$status"])
            title @output
        }
    }
}

# Adapter implementation:
# SELECT * FROM posts WHERE user_id = ?  (fetches all)
# Then Trustfall filters by status
```

**However**, this depends entirely on your adapter implementation. A well-designed adapter could optimize either approach.

#### Predicate Pushdown

Trustfall itself does **not** automatically push down `@filter` predicates to adapters as edge parameters. This is an explicit design choice:

- **Edge parameters**: Explicitly passed to `resolve_neighbors()`, enabling the adapter to optimize
- **@filter directives**: Evaluated by Trustfall's query interpreter after vertices are returned

**Recommendation**: If you control both the schema and adapter, use edge parameters for filterable edges when performance matters. The adapter can then optimize at the data source level (e.g., database WHERE clauses, API query parameters).

#### Memory and Network Efficiency

Edge parameters can significantly reduce:
- **Network traffic**: Fewer vertices transferred from data sources
- **Memory usage**: Fewer vertices held in memory
- **Processing time**: Less data to process

Example impact:

```graphql
# Fetches 1,000,000 posts, filters to 100
{
    User {
        posts {  # Adapter returns 1M posts
            created_at @filter(op: ">=", value: ["$recent"])
            title @output
        }
    }
}

# Fetches only 100 posts
{
    User {
        posts(since: "$recent") {  # Adapter returns 100 posts
            title @output
        }
    }
}
```

The second query is much more efficient if the adapter can filter at the source (e.g., database query, API parameter).

### Best Practices

1. **Check the schema first**: Use edge parameters if they're defined for your use case
2. **Design schemas with performance in mind**: Add edge parameters for common filtering needs
3. **Optimize adapters**: Implement edge parameter filtering at the data source level
4. **Combine both**: Use edge parameters for coarse filtering, `@filter` for fine-grained conditions
5. **Measure**: Profile your queries to understand actual performance impact

### Example: Combining Edge Parameters and Filters

```graphql
{
    Repository {
        name @output
        
        # Edge parameter: limits to open issues (optimized at adapter)
        issues(state: "open") {
            # Filter: additional filtering on properties
            created_at @filter(op: ">=", value: ["$since"])
            priority @filter(op: "one_of", value: ["$priorities"])
            
            title @output
        }
    }
}
```

This query uses edge parameters to limit the adapter query, then applies additional filters for fine-grained control.

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
