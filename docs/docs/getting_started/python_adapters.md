# Implementing Adapters in Python

Trustfall provides Python bindings that allow you to query data sources using Python adapters. This guide covers how to implement adapters, their features, limitations, and performance considerations.

## Python vs Rust Adapter Comparison

| Feature | Python Adapter | Rust Adapter | Notes |
|---------|----------------|--------------|-------|
| **Language** | Python 3.10+ | Rust | |
| **Installation** | `pip install trustfall` | Add to `Cargo.toml` | Python has pre-built wheels for common platforms |
| **Type Safety** | Runtime (dynamic) | Compile-time (static) | Rust catches type errors at compile time |
| **Performance** | Good (with Python-Rust overhead) | Excellent (native speed) | Python has boundary crossing overhead |
| **Required Methods** | 4 methods | 4 methods | Same core interface |
| **Method Signatures** | Dynamic typing | Generic types with lifetimes | Rust uses `&ResolveInfo` parameter |
| **Vertex Type** | Any Python type | `Clone + Debug + 'vertex` | Rust requires Clone and Debug traits |
| **Query Features** | ✅ All supported | ✅ All supported | Both support all directives |
| **@output** | ✅ Supported | ✅ Supported | |
| **@filter** | ✅ Supported | ✅ Supported | |
| **@tag** | ✅ Supported | ✅ Supported | |
| **@optional** | ✅ Supported | ✅ Supported | |
| **@recurse** | ✅ Supported | ✅ Supported | |
| **@fold** | ✅ Supported | ✅ Supported | |
| **@transform** | ✅ Supported | ✅ Supported | |
| **Type Coercion** | ✅ Supported | ✅ Supported | |
| **Edge Parameters** | ✅ Supported | ✅ Supported | |
| **Query Variables** | ✅ Supported | ✅ Supported | |
| **Lazy Evaluation** | ✅ Supported | ✅ Supported | Both support iterators/generators |
| **Schema Introspection** | ❌ Not available | ❌ Not available | Must know schema structure |
| **Optimization Hints** | ❌ Not exposed | ✅ `ResolveInfo` & `ResolveEdgeInfo` | Rust can access filter hints for optimization |
| **Predicate Pushdown** | ❌ Manual only | ✅ Via hint system | Rust adapters can check `statically_required_property()` |
| **Required Properties** | ❌ Not exposed | ✅ `required_properties()` | Rust can see which properties are needed |
| **Parallel Execution** | Manual (ThreadPoolExecutor) | Manual (Rayon, tokio, etc.) | Neither automatic, both possible |
| **Async Support** | ❌ Not supported | ✅ Supported (with async runtime) | Rust can use async/await in adapters |
| **Error Handling** | Python exceptions | `Result<T, E>` types | Different error paradigms |
| **Memory Management** | Automatic (GC) | Manual (ownership) | Rust prevents data races at compile time |
| **Iterator Protocol** | Python iterators | Rust iterators | Similar concepts, different APIs |
| **Context Ordering** | Must preserve order | Must preserve order | Both require maintaining context order |
| **Stub Generation** | ❌ Not available | ✅ `trustfall_stubgen` | Rust has automatic stub generation tool |
| **IDE Support** | Standard Python tools | rust-analyzer | Both have good IDE support |
| **Debugging** | Python debuggers (pdb, etc.) | Rust debuggers (lldb, gdb) | |
| **Testing** | pytest, unittest | cargo test, proptest | |
| **Ecosystem** | PyPI packages | crates.io crates | |
| **Learning Curve** | Lower (if know Python) | Higher (ownership, lifetimes) | |
| **Production Ready** | ✅ Yes | ✅ Yes | Both stable for production use |
| **Use Case** | Rapid prototyping, scripting | Performance-critical, type safety | Choose based on requirements |

### When to Choose Python

- **Rapid development**: Faster prototyping and iteration
- **Python ecosystem**: Need to use Python libraries (pandas, numpy, etc.)
- **Team expertise**: Team knows Python better than Rust
- **Scripting/automation**: One-off queries or data exploration
- **Acceptable overhead**: Python-Rust boundary overhead is acceptable
- **Dynamic data**: Working with highly dynamic or loosely-typed data

### When to Choose Rust

- **Performance critical**: Need maximum query performance
- **Type safety**: Want compile-time guarantees and error prevention
- **Large scale**: Building production systems with complex queries
- **Optimization**: Need access to hint system for predicate pushdown
- **Async operations**: Require async/await for I/O operations
- **Resource constrained**: Running on systems with limited resources

### Hybrid Approach

You can also use both:
- **Prototype in Python**: Develop and test adapter logic quickly
- **Optimize in Rust**: Rewrite performance-critical adapters in Rust
- **Mixed adapters**: Use Python for some data sources, Rust for others in the same application

## Quick Start

### Installing Trustfall

Trustfall is available as a Python package with pre-built wheels for common platforms:

```bash
pip install trustfall
```

Supported platforms:
- **Windows** (x86_64)
- **macOS** (x86_64 and ARM64)
- **Linux** (x86_64, manylinux compatible)
- **Python versions**: 3.10+

For other platforms, the Rust components will be compiled from source during installation.

### Basic Usage

```python
from trustfall import Adapter, Schema, execute_query
from typing import Any, Dict, Iterable, Iterator, Mapping, Tuple

# Define your vertex type
Vertex = Dict[str, Any]

# Create your adapter
class MyAdapter(Adapter[Vertex]):
    def resolve_starting_vertices(self, edge_name, parameters, /, *args, **kwargs):
        # Return starting vertices
        ...
    
    def resolve_property(self, contexts, type_name, property_name, /, *args, **kwargs):
        # Return property values
        ...
    
    def resolve_neighbors(self, contexts, type_name, edge_name, parameters, /, *args, **kwargs):
        # Return neighboring vertices
        ...
    
    def resolve_coercion(self, contexts, type_name, coerce_to_type, /, *args, **kwargs):
        # Check if vertices can be coerced to a subtype
        ...

# Create schema
schema = Schema("""
    schema {
        query: RootQuery
    }
    
    type RootQuery {
        User(id: Int!): User
    }
    
    type User {
        id: Int!
        name: String!
    }
""")

# Execute queries
adapter = MyAdapter()
query = """
{
    User(id: 1) {
        name @output
    }
}
"""
results = execute_query(adapter, schema, query, {})
for result in results:
    print(result)
```

## The Adapter Interface

The `Adapter` class is a generic abstract base class that requires implementing four methods:

### 1. resolve_starting_vertices

```python
def resolve_starting_vertices(
    self,
    edge_name: str,
    parameters: Mapping[str, FieldValue],
    /,
    *args: Any,
    **kwargs: Any,
) -> Iterable[Vertex]:
```

**Purpose**: Produce starting vertices for a query.

**Parameters**:
- `edge_name`: Name of the starting edge (e.g., `"User"`, `"FindPost"`)
- `parameters`: Dictionary of parameter values required by the edge

**Returns**: An iterable of vertices to start the query from

**Example**:
```python
def resolve_starting_vertices(self, edge_name, parameters, /, *args, **kwargs):
    if edge_name == "User":
        user_id = parameters["id"]
        user = self.database.get_user(user_id)
        if user:
            return [user]
        return []
    elif edge_name == "AllUsers":
        return self.database.get_all_users()
    else:
        raise NotImplementedError(f"Unknown edge: {edge_name}")
```

**Performance tip**: Return a list or other concrete iterable rather than a generator when possible, as this can improve performance through better iterator optimization.

### 2. resolve_property

```python
def resolve_property(
    self,
    contexts: Iterator[Context[Vertex]],
    type_name: str,
    property_name: str,
    /,
    *args: Any,
    **kwargs: Any,
) -> Iterable[Tuple[Context[Vertex], FieldValue]]:
```

**Purpose**: Resolve property values for vertices.

**Parameters**:
- `contexts`: Iterator of query contexts, each containing an `active_vertex`
- `type_name`: Name of the vertex type (e.g., `"User"`)
- `property_name`: Name of the property to resolve (e.g., `"name"`, `"email"`)

**Returns**: An iterable of `(context, property_value)` tuples

**Special cases**:
- If `property_name` is `"__typename"`, return the actual type name of the vertex
- If `context.active_vertex` is `None`, the property value must be `None`

**Example**:
```python
def resolve_property(self, contexts, type_name, property_name, /, *args, **kwargs):
    for context in contexts:
        vertex = context.active_vertex
        value = None
        
        if vertex is not None:
            if property_name == "__typename":
                value = vertex.get("__type", type_name)
            elif property_name == "name":
                value = vertex.get("name")
            elif property_name == "email":
                value = vertex.get("email")
            else:
                raise NotImplementedError(f"Unknown property: {property_name}")
        
        yield (context, value)
```

**Performance tip**: Process contexts in order and yield results immediately to enable streaming processing.

### 3. resolve_neighbors

```python
def resolve_neighbors(
    self,
    contexts: Iterator[Context[Vertex]],
    type_name: str,
    edge_name: str,
    parameters: Mapping[str, FieldValue],
    /,
    *args: Any,
    **kwargs: Any,
) -> Iterable[Tuple[Context[Vertex], Iterable[Vertex]]]:
```

**Purpose**: Resolve neighboring vertices across an edge.

**Parameters**:
- `contexts`: Iterator of query contexts
- `type_name`: Name of the source vertex type
- `edge_name`: Name of the edge to traverse
- `parameters`: Dictionary of edge parameters

**Returns**: An iterable of `(context, neighbors)` tuples where `neighbors` is an iterable of vertices

**Special cases**:
- If `context.active_vertex` is `None`, return an empty neighbors iterable
- If your schema has no edges (only starting edges), this method may be `raise NotImplementedError()`

**Example**:
```python
def resolve_neighbors(self, contexts, type_name, edge_name, parameters, /, *args, **kwargs):
    for context in contexts:
        vertex = context.active_vertex
        neighbors = []
        
        if vertex is not None:
            if edge_name == "posts":
                user_id = vertex["id"]
                neighbors = self.database.get_user_posts(user_id)
            elif edge_name == "followers":
                user_id = vertex["id"]
                max_followers = parameters.get("limit", 100)
                neighbors = self.database.get_followers(user_id, limit=max_followers)
            else:
                raise NotImplementedError(f"Unknown edge: {edge_name}")
        
        yield (context, neighbors)
```

**Performance tip**: Consider lazy loading neighbors when the edge might return many vertices. Use generators or iterators to avoid loading all data into memory.

### 4. resolve_coercion

```python
def resolve_coercion(
    self,
    contexts: Iterator[Context[Vertex]],
    type_name: str,
    coerce_to_type: str,
    /,
    *args: Any,
    **kwargs: Any,
) -> Iterable[Tuple[Context[Vertex], bool]]:
```

**Purpose**: Check if vertices can be coerced (narrowed) to a more specific type.

**Parameters**:
- `contexts`: Iterator of query contexts
- `type_name`: Name of the current interface type
- `coerce_to_type`: Name of the type to coerce to

**Returns**: An iterable of `(context, can_coerce)` tuples where `can_coerce` is a boolean

**Special cases**:
- If `context.active_vertex` is `None`, return `False`
- If your schema has no interfaces or type hierarchies, this method may be `raise NotImplementedError()`

**Example**:
```python
def resolve_coercion(self, contexts, type_name, coerce_to_type, /, *args, **kwargs):
    for context in contexts:
        vertex = context.active_vertex
        can_coerce = False
        
        if vertex is not None:
            vertex_type = vertex.get("__type", type_name)
            
            # Check if the vertex's actual type matches the target type
            # or implements the target interface
            if vertex_type == coerce_to_type:
                can_coerce = True
            elif coerce_to_type == "PremiumUser" and vertex.get("is_premium"):
                can_coerce = True
        
        yield (context, can_coerce)
```

## Data Types

### FieldValue

```python
FieldValue = Union[None, str, int, float, bool, list["FieldValue"]]
```

The `FieldValue` type represents all valid property values, query arguments, and result columns:
- `None` - null values
- `str` - strings
- `int` - integers (Python `int`, maps to Rust `i64`)
- `float` - floating-point numbers (Python `float`, maps to Rust `f64`)
- `bool` - booleans
- `list[FieldValue]` - lists (can be nested)

**Important**: Lists must be homogeneous - all non-null elements must have the same type.

### Context[Vertex]

The `Context` object provides access to the current vertex being processed:

```python
context.active_vertex  # The current vertex, or None
```

## Complete Example: Numbers Adapter

Here's a complete, working adapter that demonstrates all features:

```python
from typing import Any, Iterable, Iterator, Mapping, Tuple, cast
from trustfall import Adapter, Context, FieldValue

# Use int as the vertex type
Vertex = int

NUMBER_NAMES = ["zero", "one", "two", "three", "four", "five", 
                "six", "seven", "eight", "nine", "ten"]

class NumbersAdapter(Adapter[Vertex]):
    def resolve_starting_vertices(
        self, edge_name, parameters, /, *args, **kwargs
    ) -> Iterable[Vertex]:
        max_value = cast(int, parameters["max"])
        return list(range(0, max_value))
    
    def resolve_property(
        self, contexts, type_name, property_name, /, *args, **kwargs
    ) -> Iterable[Tuple[Context[Vertex], FieldValue]]:
        for context in contexts:
            vertex = context.active_vertex
            value = None
            
            if vertex is not None:
                if property_name == "value":
                    value = vertex
                elif property_name == "name":
                    if 0 <= vertex < len(NUMBER_NAMES):
                        value = NUMBER_NAMES[vertex]
                else:
                    raise NotImplementedError(f"Unknown property: {property_name}")
            
            yield (context, value)
    
    def resolve_neighbors(
        self, contexts, type_name, edge_name, parameters, /, *args, **kwargs
    ) -> Iterable[Tuple[Context[Vertex], Iterable[Vertex]]]:
        for context in contexts:
            vertex = context.active_vertex
            neighbors = []
            
            if vertex is not None:
                if edge_name == "predecessor":
                    if vertex > 0:
                        neighbors = [vertex - 1]
                elif edge_name == "successor":
                    neighbors = [vertex + 1]
                elif edge_name == "multiple":
                    if vertex > 0:
                        max_value = cast(int, parameters["max"])
                        neighbors = range(
                            2 * vertex,
                            max_value * vertex + 1,
                            vertex
                        )
                else:
                    raise NotImplementedError(f"Unknown edge: {edge_name}")
            
            yield (context, neighbors)
    
    def resolve_coercion(
        self, contexts, type_name, coerce_to_type, /, *args, **kwargs
    ) -> Iterable[Tuple[Context[Vertex], bool]]:
        # This schema doesn't use type coercion
        raise NotImplementedError()

# Usage
from trustfall import Schema, execute_query

schema = Schema("""
    schema {
        query: RootSchemaQuery
    }
    
    type RootSchemaQuery {
        Number(max: Int!): [Number!]
    }
    
    type Number {
        name: String
        value: Int!
        predecessor: Number
        successor: Number!
        multiple(max: Int!): [Number!]
    }
""")

adapter = NumbersAdapter()
query = """
{
    Number(max: 10) {
        value @output
        name @output
        
        successor {
            next: value @output
        }
    }
}
"""

results = list(execute_query(adapter, schema, query, {}))
# Results: [{"value": 0, "name": "zero", "next": 1}, ...]
```

## Error Handling

Trustfall provides several exception types for different error conditions:

```python
from trustfall import (
    ParseError,           # Invalid query syntax
    ValidationError,      # Query doesn't match schema
    FrontendError,        # Unsupported query operations
    InvalidSchemaError,   # Schema is invalid
    QueryArgumentsError,  # Query arguments are invalid
)

try:
    results = execute_query(adapter, schema, query, args)
    for result in results:
        print(result)
except ParseError as e:
    print(f"Query syntax error: {e}")
except ValidationError as e:
    print(f"Query validation error: {e}")
except QueryArgumentsError as e:
    print(f"Invalid query arguments: {e}")
except FrontendError as e:
    print(f"Query operation error: {e}")
```

## Features

### Supported Query Features

All Trustfall query features are fully supported in Python:

- ✅ **@output** - Output field values
- ✅ **@filter** - Filter vertices by property values
- ✅ **@tag** - Save values for later filtering
- ✅ **@optional** - Optional edges (LEFT JOIN)
- ✅ **@recurse** - Recursive traversal
- ✅ **@fold** - Aggregate results into lists
- ✅ **@transform** - Transform values (e.g., count)
- ✅ **Type coercion** - Narrow vertices to subtypes
- ✅ **Edge parameters** - Parameterized edge traversal
- ✅ **Query variables** - Parameterized queries

### Vertex Types

You can use any Python type as your `Vertex` type:

```python
# Dictionary vertices
class DictAdapter(Adapter[Dict[str, Any]]):
    ...

# Custom class vertices
class User:
    def __init__(self, id, name):
        self.id = id
        self.name = name

class UserAdapter(Adapter[User]):
    ...

# Simple types (int, str, etc.)
class IntAdapter(Adapter[int]):
    ...

# Dataclasses
from dataclasses import dataclass

@dataclass
class Person:
    id: int
    name: str

class PersonAdapter(Adapter[Person]):
    ...
```

### Lazy Evaluation

Trustfall uses lazy evaluation, which means:
- Query results are returned as an iterator
- Data is fetched only as needed
- Adapters can use generators for memory efficiency
- Early termination is possible (e.g., with `LIMIT` or filters)

```python
def resolve_neighbors(self, contexts, type_name, edge_name, parameters, /, *args, **kwargs):
    for context in contexts:
        if context.active_vertex is not None:
            # Use a generator for large result sets
            def neighbor_generator():
                for item in large_dataset:
                    if matches_criteria(item):
                        yield item
            
            yield (context, neighbor_generator())
        else:
            yield (context, [])
```

## Limitations

### 1. No Schema Introspection

Python adapters cannot programmatically query the schema. You must know your schema structure when implementing the adapter methods.

### 2. Limited Type Safety

Python's dynamic typing means type errors are caught at runtime rather than compile time. Use type hints and runtime checks:

```python
def resolve_property(self, contexts, type_name, property_name, /, *args, **kwargs):
    for context in contexts:
        vertex = context.active_vertex
        value = None
        
        if vertex is not None:
            # Runtime validation
            if not isinstance(vertex, dict):
                raise TypeError(f"Expected dict vertex, got {type(vertex)}")
            
            value = vertex.get(property_name)
        
        yield (context, value)
```

### 3. Iterator Consumption

Iterators passed to adapter methods (like `contexts`) can only be consumed once. Store results if you need to iterate multiple times:

```python
def resolve_property(self, contexts, type_name, property_name, /, *args, **kwargs):
    # DON'T do this - contexts can only be consumed once
    # count = sum(1 for _ in contexts)
    # for context in contexts:  # This won't work!
    #     ...
    
    # DO this instead
    for context in contexts:
        # Process each context once
        yield (context, compute_value(context))
```

### 4. No Parallel Execution

Trustfall's Python bindings execute queries in a single thread. For parallel data access, implement parallelism within your adapter:

```python
from concurrent.futures import ThreadPoolExecutor

class ParallelAdapter(Adapter[Dict[str, Any]]):
    def __init__(self):
        self.executor = ThreadPoolExecutor(max_workers=4)
    
    def resolve_neighbors(self, contexts, type_name, edge_name, parameters, /, *args, **kwargs):
        # Collect contexts (needed for parallel processing)
        context_list = list(contexts)
        
        # Fetch neighbors in parallel
        def fetch_neighbors(context):
            if context.active_vertex:
                return (context, fetch_from_api(context.active_vertex))
            return (context, [])
        
        yield from self.executor.map(fetch_neighbors, context_list)
```

### 5. FieldValue Type Restrictions

Only specific Python types are supported as field values. Custom objects cannot be used directly:

```python
# Supported
value = "string"        # ✅
value = 42             # ✅
value = 3.14           # ✅
value = True           # ✅
value = None           # ✅
value = [1, 2, 3]      # ✅
value = ["a", "b"]     # ✅

# Not supported
value = {"key": "val"} # ❌ Dicts are not FieldValue
value = SomeObject()   # ❌ Custom objects are not FieldValue
value = (1, 2)         # ❌ Tuples are not FieldValue
```

## Performance Considerations

### 1. Python-Rust Boundary Overhead

Each call from Rust to Python (and vice versa) has overhead. Minimize boundary crossings:

```python
# Less efficient: Many small calls
def resolve_property(self, contexts, type_name, property_name, /, *args, **kwargs):
    for context in contexts:
        value = None
        if context.active_vertex:
            # Each property access might cross the boundary
            value = self.fetch_property(context.active_vertex, property_name)
        yield (context, value)

# More efficient: Batch operations
def resolve_property(self, contexts, type_name, property_name, /, *args, **kwargs):
    # Collect all vertices first
    context_list = list(contexts)
    vertices = [ctx.active_vertex for ctx in context_list]
    
    # Batch fetch all values at once
    values = self.batch_fetch_properties(vertices, property_name)
    
    # Yield results
    for context, value in zip(context_list, values):
        yield (context, value)
```

**Trade-off**: Batching reduces boundary crossings but increases memory usage. Use for small to medium batches.

### 2. Iterator vs Iterable

Concrete iterables (like `list`) allow Rust-side optimizations:

```python
# Good: Returns a concrete list
def resolve_starting_vertices(self, edge_name, parameters, /, *args, **kwargs):
    return list(range(100))  # ✅ Rust can optimize this

# Less optimal: Returns a generator
def resolve_starting_vertices(self, edge_name, parameters, /, *args, **kwargs):
    return (x for x in range(100))  # Still works but less optimized
```

### 3. Lazy vs Eager Evaluation

For neighbors with many results, use lazy evaluation:

```python
def resolve_neighbors(self, contexts, type_name, edge_name, parameters, /, *args, **kwargs):
    for context in contexts:
        if context.active_vertex:
            # Lazy: Only fetches what's needed
            def lazy_neighbors():
                for item in database.stream_results():
                    yield item
            yield (context, lazy_neighbors())
        else:
            yield (context, [])
```

For small result sets, eager evaluation may be faster:

```python
def resolve_neighbors(self, contexts, type_name, edge_name, parameters, /, *args, **kwargs):
    for context in contexts:
        if context.active_vertex:
            # Eager: Fetches all at once
            neighbors = list(database.get_all_neighbors())
            yield (context, neighbors)
        else:
            yield (context, [])
```

### 4. Caching

Implement caching to avoid redundant work:

```python
from functools import lru_cache

class CachedAdapter(Adapter[Dict[str, Any]]):
    @lru_cache(maxsize=1000)
    def _fetch_user(self, user_id):
        return self.database.get_user(user_id)
    
    def resolve_starting_vertices(self, edge_name, parameters, /, *args, **kwargs):
        if edge_name == "User":
            user_id = parameters["id"]
            user = self._fetch_user(user_id)
            return [user] if user else []
        return []
```

### 5. Database Connection Pooling

For database-backed adapters, use connection pooling:

```python
import sqlite3
from contextlib import contextmanager

class DatabaseAdapter(Adapter[Dict[str, Any]]):
    def __init__(self, db_path):
        self.db_path = db_path
        self.connection_pool = []
    
    @contextmanager
    def get_connection(self):
        if self.connection_pool:
            conn = self.connection_pool.pop()
        else:
            conn = sqlite3.connect(self.db_path)
        try:
            yield conn
        finally:
            self.connection_pool.append(conn)
    
    def resolve_neighbors(self, contexts, type_name, edge_name, parameters, /, *args, **kwargs):
        with self.get_connection() as conn:
            for context in contexts:
                if context.active_vertex:
                    cursor = conn.execute(
                        "SELECT * FROM neighbors WHERE parent_id = ?",
                        (context.active_vertex["id"],)
                    )
                    yield (context, [dict(row) for row in cursor.fetchall()])
                else:
                    yield (context, [])
```

### 6. Memory Management

For large datasets, stream data rather than loading everything into memory:

```python
def resolve_neighbors(self, contexts, type_name, edge_name, parameters, /, *args, **kwargs):
    # Bad: Loads everything into memory
    all_data = self.database.fetch_all_neighbors()  # Could be millions of rows
    for context in contexts:
        neighbors = [x for x in all_data if x.parent_id == context.active_vertex["id"]]
        yield (context, neighbors)
    
    # Good: Streams data
    for context in contexts:
        if context.active_vertex:
            # Stream neighbors one at a time
            def stream_neighbors():
                for neighbor in self.database.stream_neighbors(context.active_vertex["id"]):
                    yield neighbor
            yield (context, stream_neighbors())
        else:
            yield (context, [])
```

## Best Practices

### 1. Use Type Hints

Type hints improve code clarity and enable better IDE support:

```python
from typing import Any, Iterable, Iterator, Mapping, Tuple
from trustfall import Adapter, Context, FieldValue

class MyAdapter(Adapter[Dict[str, Any]]):
    def resolve_property(
        self,
        contexts: Iterator[Context[Dict[str, Any]]],
        type_name: str,
        property_name: str,
        /,
        *args: Any,
        **kwargs: Any,
    ) -> Iterable[Tuple[Context[Dict[str, Any]], FieldValue]]:
        ...
```

### 2. Validate Inputs

Validate parameters and vertex data:

```python
def resolve_starting_vertices(self, edge_name, parameters, /, *args, **kwargs):
    if edge_name == "User":
        user_id = parameters.get("id")
        if not isinstance(user_id, int):
            raise ValueError(f"User ID must be an integer, got {type(user_id)}")
        if user_id < 0:
            raise ValueError(f"User ID must be non-negative, got {user_id}")
        
        user = self.database.get_user(user_id)
        return [user] if user else []
    
    raise NotImplementedError(f"Unknown starting edge: {edge_name}")
```

### 3. Handle None Vertices

Always check if `context.active_vertex` is `None`:

```python
def resolve_property(self, contexts, type_name, property_name, /, *args, **kwargs):
    for context in contexts:
        vertex = context.active_vertex
        
        # Always handle None case
        if vertex is None:
            yield (context, None)
            continue
        
        # Process non-None vertices
        value = vertex.get(property_name)
        yield (context, value)
```

### 4. Preserve Context Order

Always yield contexts in the same order they were received:

```python
def resolve_property(self, contexts, type_name, property_name, /, *args, **kwargs):
    # Correct: Processes and yields in order
    for context in contexts:
        value = compute_value(context)
        yield (context, value)
    
    # Incorrect: Don't collect, sort, or reorder contexts
    # context_list = list(contexts)
    # context_list.sort(key=...)  # ❌ Breaks order requirement
    # for context in context_list:
    #     yield (context, value)
```

### 5. Use Appropriate Data Structures

Choose vertex representations that match your use case:

```python
# Dictionaries: Flexible, good for dynamic data
class DictAdapter(Adapter[Dict[str, Any]]):
    ...

# Dataclasses: Typed, good for structured data
from dataclasses import dataclass

@dataclass
class User:
    id: int
    name: str
    email: str

class UserAdapter(Adapter[User]):
    ...

# Named tuples: Immutable, memory-efficient
from typing import NamedTuple

class UserTuple(NamedTuple):
    id: int
    name: str
    email: str

class TupleAdapter(Adapter[UserTuple]):
    ...
```

## Debugging Tips

### 1. Enable Query Logging

Log adapter method calls to understand query execution:

```python
import logging

logger = logging.getLogger(__name__)

class LoggingAdapter(Adapter[Dict[str, Any]]):
    def resolve_property(self, contexts, type_name, property_name, /, *args, **kwargs):
        logger.debug(f"Resolving property {type_name}.{property_name}")
        for context in contexts:
            value = self._get_value(context, property_name)
            logger.debug(f"  Vertex {context.active_vertex} -> {value}")
            yield (context, value)
```

### 2. Validate Return Types

Add runtime checks during development:

```python
def resolve_property(self, contexts, type_name, property_name, /, *args, **kwargs):
    for context in contexts:
        value = self._get_value(context, property_name)
        
        # Validate FieldValue type
        if not isinstance(value, (type(None), str, int, float, bool, list)):
            raise TypeError(f"Invalid field value type: {type(value)}")
        
        yield (context, value)
```

### 3. Test with Simple Queries

Start with simple queries and gradually increase complexity:

```python
# Test 1: Simple property access
query = """
{
    User(id: 1) {
        name @output
    }
}
"""

# Test 2: Add edge traversal
query = """
{
    User(id: 1) {
        name @output
        posts {
            title @output
        }
    }
}
"""

# Test 3: Add filtering
query = """
{
    User(id: 1) {
        name @output
        posts @filter(op: ">=", value: ["$min_likes"]) {
            title @output
        }
    }
}
"""
```

## See Also

- [Query Language Reference](../language_reference/querying/index.md) - Complete guide to writing Trustfall queries
- [Schema Guide](../language_reference/schema/index.md) - How to define Trustfall schemas
- [Query Examples](../language_reference/querying/examples.md) - Example queries demonstrating all features
