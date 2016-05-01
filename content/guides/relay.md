---
title: Relay
order: 11
---

While GraphQL specifies what queries, mutations, and object types should look like,
Relay is a client-side implementation of an efficient data storage and (re-)fetching system
designed to work with a GraphQL server.

To allow Relay to work its magic on the client side, all GraphQL queries and mutations need to
follow certain conventions. `Absinthe.Relay` provides utilities to help you make your
server-side schemas Relay-compatible while requiring only minimal changes to your
existing code.

`Absinthe.Relay` currently provides support for the three fundamental pieces to the Relay puzzle:
*nodes*, which are normal GraphQL objects with a unique ID assigned to them; *mutations*, which
in Relay conform to a certain input and output format; and *connections*, which provide enhanced
functionality around many-to-one lists.

## Nodes
To enable Relay to be clever about caching and (re-)fetching data objects, your server must assign
a globally unique ID to each object before sending it down the wire. Absinthe will take care of this
for you if you add a `node interface` to the top level of your schema:

```elixir
defmodule YourApp.Schema do
  use Absinthe.Schema
  use Absinthe.Relay.Schema
  import_types YourApp.Schema.Types

  node interface do
    resolve_type fn
      %{age: _}, _ -> :person
      %{employee_count: _}, _ -> :business
      _, _ -> nil  
    end
  end

  # ... mutations, queries ...
```

For instance, if your query or mutation resolver returns

`%{id: 19, business_name: "ACME Corp.", employee_count: 52}`,

Absinthe will pattern-match this to obtain the object type `:business`.
The `:business` object type, in turn, will generate the global ID and write it into the object's `:id` field,
resulting in something like

`%{id: "UWxf59AcjK=", business_name: "ACME Corp.", employee_count: 52}`.

To give the `:business` object type the power to do this, macro-prefix its type definition with `node`, i.e.

```elixir
node object :business do  # <-- notice the macro prefix "node"
  field :id, :id  # the internal id (e.g. from Ecto)
  field :business_name, non_null(:string)
  field :employee_count, :integer
end
```

<p class="notice">
<strong>Important:</strong> the global ID is generated based on the object's unique identifier, which by default
is <strong>the value of its existing `:id` field.</strong> This is convenient, because if you are using Ecto,
the primary-key `:id` database field is typically enough to uniquely identify an object of a given type.
It also means, however, that **the internal `:id` of a node object will be overwritten with its
global ID.** If you wish to generate your global IDs based on something other than the
existing `:id` field, provide the `id_fetcher` option (see the [documentation](https://hexdocs.pm/absinthe_relay/Absinthe.Relay.Node.html)).
</p>

### Node query field
Relay expects you to provide a query field called `node` that accepts a global ID and returns the
corresponding node object. Absinthe makes it easy to set this up ­– see the
[documentation](https://hexdocs.pm/absinthe_relay/Absinthe.Relay.Node.html) for more info.

```elixir
query do
  # ...
  node field do  # <-- notice the macro prefix "node"
    resolve fn
      %{type: :person, id: id}, _ ->
        YourApp.Resolver.Person.find(%{id: id}, %{})  # get the person from the DB somehow
      %{type: :business, id: id}, _ ->
        {:ok, Map.get(@businesses, id)}  # get the business from @businesses
      # etc.
    end
  end
  # ... more queries ...
end
```

This will create a `node` query field that will not only accept a global ID, but also convert it
directly into a map of the shape `%{type: :the_node_type, id: the_internal_id}`. As shown in the example,
you can use the `:type` for pattern matching, and the `:id` to pass to your resolver
– which should be a function returning `{:ok, _}` or `{:error, _}` (e.g. an Ecto query result).

## Mutations
Relay expects mutations to accept exactly one argument named `input`. Furthermore, it will add a
field `clientMutationId` to this `input` object and expect to get it back in the result.
The existing [documentation on Absinthe.Relay.Mutation](https://hexdocs.pm/absinthe_relay/Absinthe.Relay.Mutation.html)
covers all of those points.

One important aspect is that mutation input fields cannot be of one of your object types; only
scalars and arrays are allowed. To specify that an array of certain elements is expected,
you can use `list_of` in combination with the `input_object` macro, e.g.

```elixir
defmodule YourApp.Schema
  # ...

  input_object :person_input_object do
    field :first_name, non_null(:string)
    field :last_name, non_null(:string)
    field :age, :integer
  end

  mutation do
    @desc "A mutation that inserts a list of persons into the database"
    payload field :bulk_create_persons do
      input do
        field :persons, list_of(:person_input_object)
      end
      output do
        # whatever you need
      end
      resolve &Resolver.Person.bulk_create/2
    end

    # ... more mutations ...
  end
end
```
Indeed, using `input_object` will help keep your mutations uncluttered even when you aren't
using `list_of`. Of course, fields within `input_object`s can refer to other `input_object`s as well.

### Referencing existing nodes in mutation inputs
Occasionally, your client may wish to make reference to an existing node in the mutation input (this happens
particularly when manipulating the connection edges of a parent node).

* if you specify an `:integer`
input field to be provided with the node's *internal* `:id`, you must ensure that the
client's nodes hold copies of those internal `:id`s alongside their global ID, so the client can use
them as mutation inputs.
* if, on the other hand, you specify a `:string` input field to be supplied
with a global ID, you will need to explicitly convert this to an internal `:id` inside of
your resolver function like so:

```elixir
# description: Converting a global id to an internal one
def bulk_create(%{persons: new_persons, group: global_group_id}, _) do
  {:ok, %{id: internal_group_id}} = Absinthe.Relay.Node.from_global_id(global_group_id, YourApp.Schema)`
  # ... manipulate your DB using internal_group_id
```

## Connections
No examples yet – check the [documentation](https://hexdocs.pm/absinthe_relay/Absinthe.Relay.Connection.html).
