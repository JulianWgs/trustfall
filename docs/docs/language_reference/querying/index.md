# Trustfall query reference

Trustfall queries are written in a GraphQL-like syntax and support powerful querying capabilities including filtering, aggregation, recursion, and optional traversals.

## Query structure

Every Trustfall query starts from an entrypoint defined in the schema and traverses through vertices and edges to select and filter data.

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

## Directives

Trustfall queries may make use of several directives:

- [`@output`](output.md) - Mark fields to include in query results
- [`@filter`](filter.md) - Filter vertices based on property values
- [`@tag`](tag.md) - Save values for use in later filters
- [`@optional`](optional.md) - Make edges optional (like SQL LEFT JOIN)
- [`@recurse`](recurse.md) - Follow edges recursively to arbitrary depth
- [`@fold`](fold.md) - Aggregate multiple results into lists
- [`@transform`](transform.md) - Transform values (e.g., counting)

## Type coercions

Type coercions allow narrowing vertices to more specific types:

- [Type coercions](type_coercion.md) - Filter and process vertices by their specific type

## Edge parameters

Edges can accept parameters to control traversal:

- [Edge parameters](edge_parameters.md) - Pass arguments when traversing edges

## Comprehensive examples

For complete examples demonstrating all features:

- [Query examples](examples.md) - Comprehensive examples of all query features

## Quick reference

### Basic query

```graphql
{
    User {
        username @output
        email @output
    }
}
```

### With filtering

```graphql
{
    User {
        username @output
        age @filter(op: ">=", value: ["$min_age"])
    }
}
```

### With aggregation

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

### With optional edges

```graphql
{
    User {
        username @output
        posts @optional {
            latest_title: title @output
        }
    }
}
```

### With recursion

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

### With type coercion

```graphql
{
    Number(max: 10) {
        value @output
        ... on Prime {
            is_prime: value @output
        }
    }
}
```
