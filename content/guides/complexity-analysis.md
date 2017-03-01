---
title:Complexity Analysis
order: 12
---

A misbehaving client might send a very complex GraphQL query that would require
considerably resources to handle. In order to protect against this scenario the
complexity of a query can be estimated before it is resolved and limited to a
maximum.

To enable complexity analysis and limit the complexity to 50 using Absinthe
directly:

```elixir
Absinthe.run(doc, MyApp.Schema, analyze_complexity: true, max_complexity: 50)
```

Or with Absinthe.Plug:

```elixir
plug Absinthe.Plug,
  schema: MyApp.Schema, analyze_complexity: true, max_complexity: 50
```

By default each field in a query will increase the complexity by 1. However it
can be useful to customize the complexity. For example when a field is a list,
the complexity is usually correlated to the size of the list. To prevent large
selections, a field can use a limit argument with a suitable default. Here is a
schema that supports analyzing and limiting the complexity in this way:

```elixir
# filename: myapp/schema.ex
defmodule MyApp.Schema do

  use Absinthe.Schema

  query do
    field :people, list_of(:person) do
      arg :limit, :integer, default_value: 10
      complexity fn %{limit: limit}, child_complexity ->
        # set complexity based on maximum number of items in the list and
        # complexity of a child.
        limit * child_complexity
      end
    end
  end

  object :person do
    field :name, :string
    field :age, :integer
    # constant complexity for this object
    complexity 3
  end

end
```
