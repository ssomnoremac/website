---
title: Relay
order: 11
---

While GraphQL specifies what queries, mutations, and object types should look
like, Relay is a client-side implementation of an efficient data storage and
(re-)fetching system that is designed to work with a GraphQL server.

To allow Relay to work its magic on the client side, all GraphQL queries and
mutations need to follow certain conventions. `Absinthe.Relay` provides
utilities to help you make your server-side schemas Relay-compatible while
requiring only minimal changes to your existing code.

`Absinthe.Relay` supports three fundamental pieces of the Relay puzzle: *nodes*,
which are normal GraphQL objects with a unique global ID scheme; *mutations*,
which in Relay conform to a certain input and output structure; and
*connections*, which provide enhanced functionality around many-to-one lists
(most notably pagination).

## Using Absinthe.Relay

Make sure you have the [absinthe_relay](https://hex.pm/packages/absinthe_relay)
package [configured](https://github.com/absinthe-graphql/absinthe_relay#installation)
as a dependency for your application.

To add Relay support schemas should start with `use Absinthe.Relay.Schema`, eg:

```elixir
defmodule Schema do
  use Absinthe.Schema
  use Absinthe.Relay.Schema

  # ...

end
```

If you're defining your types in a separate type module that you're using via
`import_types` in your schema, use the `Notation` module instead:

```elixir
defmodule Schema.Types do
  use Absinthe.Schema.Notation
  use Absinthe.Relay.Schema.Notation

  # ...

end
```

Now you're ready to implement the Relay features you need.

## Nodes

To enable Relay to be clever about caching and (re-)fetching data objects, your
server must assign a globally unique ID to each object before sending it down
the wire. Absinthe will take care of this for you if you provide some additional
information in your schema.

First of all, you must define a `:node` interface in your schema. Rather than
do this manually, `Absinthe.Relay` provides a macro so most of the configuration
is handled for you.

Use `node interface` in your schema:

```elixir
node interface do
  resolve_type fn
    %YourApp.Model.Person{}, _ ->
      :person
    %YourApp.Model.Business{}, _ ->
      :business
    _, _ ->
      nil
  end
end

# ... mutations, queries ...
```

For instance, if your query or mutation resolver returns:

```elixir
{:ok, %YourApp.Model.Business{id: 19, business_name: "ACME Corp.", employee_count: 52}}
```

Absinthe will pattern-match the value to determine that the object type is
`:business`. This becomes important when you configure your `:business` type as a `node`:

```elixir
# description: Notice the macro prefix, `node`
node object :business do  # <-- notice the macro prefix "node"
  field :business_name, non_null(:string)
  field :employee_count, :integer
end
```

While it may appear that your `:business` object type only has two fields,
`:business_name` and `:employee_count`, it actually has _three_. An `:id` field
is configured for you because you used the `node object` macro, and because the
`:node` interface knows how to identify the values returned from your resolvers,
that `:id` field is automatically set-up to convert internal (in this case,
numeric) IDs to the global ID scheme -- an opaque string (like `"UWxf59AcjK="`)
will be returned instead.


<p class="notice">
<strong>Important:</strong> the global ID is generated based on the object's
unique identifier, which by default is <strong>the value of its existing <code>:id</code>
field</strong>. This is convenient, because if you are using Ecto, the
primary key <code>:id</code> database field is typically enough to uniquely identify an
object of a given type. It also means, however, that <i>the internal <code>:id</code> of a
node object will not be available to be queried as <code>:id</code>.</i>
</p>

- If you wish to generate your global IDs based on something other than the
  existing `:id` field (if, for instance, your internal IDs are returned as `_id`),
  provide the `:id_fetcher` option (see the [documentation](https://hexdocs.pm/absinthe_relay/Absinthe.Relay.Node.html)).
- If you wish to make your internal ID queryable, you must return it as a
  different field (eg, you could define an `:internal_id` field whose resolver
  extracts the raw, internal `:id` value from the source map/struct).

### Node query field

Ok, so your node objects provide a global `:id`. How does Relay use it?

Relay expects you to provide a query field called `node` that accepts a global
ID (as arg `:id`) and returns the corresponding node object. Absinthe makes it
easy to set this up -- use the `node field` macro inside your `query`.

```elixir
# description: Notice the macro prefix, `node`
query do
  # ...
  node field do
    resolve fn
      %{type: :person, id: id}, _ ->
        # Get the person from the DB somehow, returning a tuple
        YourApp.Resolver.Person.find(%{id: id}, %{})
      %{type: :business, id: id}, _ ->
        # Get the business from @businesses
        {:ok, Map.get(@businesses, id)}
      # etc.
    end
  end
  # ... more queries ...
end
```

Notice that the resolver for `node field` expects the first (args) argument to
contain a `:type` and `:id`. These are the node object type identifier and the
internal (non-global) ID, automatically parsed from the global ID. The resolver
looks up the correct value using the internal ID and returns a tuple, as usual.

For more information, see the [documentation](https://hexdocs.pm/absinthe_relay/Absinthe.Relay.Node.html).

### Converting node IDs to internal IDs for resolvers

If you need to parse a node (global) ID for use in a resolver, there is a
helful utility, `parsing_node_ids/2` that is automatically imported for you.
Here's an example of how it works.

Let's assume we have a field, `:employees`, that returns a list of `:person`
objects for a given `:business_id` -- a node ID:

```elixir
# description: Somewhere in our schema
query do
  field :employees, list_of(:people) do
    arg :business_id, :id
    resolve &resolve_employees/2
  end
end

def resolve_employees(%{business_id: global_id}, _) do
  # But I need an internal ID to look-up the employees!
end
```

In `resolve_employees/2`, we could certainly parse out the internal ID manually.
Here's how that would look:

```elixir
# description: Manually converting the node ID to an internal ID
def resolve_employees(%{business_id: global_id}, _) do
  {:ok, %{type: :business, id: internal_id}} = Absinthe.Relay.Node.from_global_id(global_id, YourApp.Schema)
  # TODO: find employees using internal_id, return tuple
end
```

Obviously this can get a bit tedious if we have to do it often. Instead, we can
use `parsing_node_ids/2` to _wrap_ our resolver function to do the parsing for
us, invoking our function with the internal ID instead. We just have to tell the
`parsing_node_ids/2` what ID field arguments to parse and what the associated
types should be:

```elixir
# description: Somewhere in our schema
query do
  field :employees, list_of(:people) do
    arg :business_id, :id
    resolve parsing_node_ids(&resolve_employees/2, business_id: :business)
  end
end

def resolve_employees(%{business_id: internal_id}, _) do
  # We have an internal ID!
end
```

This leaves our resolver function virtually unchanged, and keeps our code much
cleaner.

## Mutations

Relay sets some specific constraints around the way arguments and results for
mutations are structured.

Relay expects mutations to accept exactly one argument, `input`, an
`InputObject`. On the JavaScript side, it automatically populates a field on the
input, `clientMutationId`, and expects to get it back, unchanged, as part of the
result. Thankfully `Absinthe.Relay` abstracts these details away from the schema
designer, allowing them to focus on any _other_ arguments needed or results
expected.

<p class="notice">
  <strong>Important:</strong> Remember that input fields (and arguments in
  general) cannot be of one of your <code>object</code> types. Use <code>input_object</code> to
  model complex argument types.
</p>

In this example, we accept a list of multiple `:person_input_object` values to
insert people into a database.

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
        # fields in the result
      end
      resolve &Resolver.Person.bulk_create/2
    end

    # ... more mutations ...
  end
end
```

Note the `payload` macro introduces a Relay mutation, `input` defines the fields
(inside the `input` argument), and `output` defines the fields available as part
of the result.

See the [documentation on Absinthe.Relay.Mutation](https://hexdocs.pm/absinthe_relay/Absinthe.Relay.Mutation.html)
for more information.

### Referencing existing nodes in mutation inputs

Occasionally, your client may wish to make reference to an existing node in the
mutation input (this happens particularly when manipulating the connection edges
of a parent node).

Incoming IDs for node types may have to be converted to their internal
equivalents so you can persist changes to your backend. For this purpose, you
can use `Absinthe.Relay.Node.from_global_id/2` to parse node (global) IDs
manually.

```elixir
# description: Converting a global id to an internal one
def bulk_create(%{persons: new_persons, group: global_group_id}, _) do
  {:ok, %{type: :group, id: internal_group_id}} = Absinthe.Relay.Node.from_global_id(global_group_id, YourApp.Schema)`
  # ... manipulate your DB using internal_group_id
end
```

If, of course, your client knows the internal IDs (in a peer field to `:id`, eg,
`:internal_id`), you can depends on that ID -- but we recommend that you use
node IDs as they are opaque values and it's the more conventional practice.

<p class="notice">
  <strong>Important:</strong> When using <code>from_global_id</code>, remember to always
  match the <code>:type</code> value to ensure the internal ID is for the type you expect,
  and a global ID for the wrong type of node hasn't been mistakenly sent to the
  server.
</p>

## Connections

One of the more popular features of Relay is the rich pagination support provided by its
connections. [This medium post](https://dev-blog.apollodata.com/explaining-graphql-connections-c48b7c3d6976)
has a good explaination of the full feature set and nomenclature.

You can create a connection type to paginate by with:

`connection node_type: :location`

This will automatically define two new types: `:location_connection` and `:location_edge`.

We define a field that uses these types to paginate associated records by using 
`connection field`. Here, for instance, we support paginating a business’s locations:

```
# description: somewhere in types.ex

object :business do
  field :short_name, :string
  connection field :locations, node_type: :location do
    resolve fn
      pagination_args, %{source: business} ->
        Location
        |> where(business_id: ^business.id)
        |> order_by(:inserted_at)
        |> Connection.from_query(&Repo.all/1, pagination_args)
    end
  end
end
```

We are piping a query for the associated locations into `from_query/3` along with the default
relay pagination arguments that allow for pagination. For example, to get just the first 10
locations, use the `first` argument:

```
query{
  business(id:"9ea6605e-e6c8-44ea-98d0-1fe6276e193d"){
    shortName
    locations(first:10){
      edges{
        node{
          address1
          city
        }
      }
    }
  }
}
```
    
Check the [documentation](https://hexdocs.pm/absinthe_relay/Absinthe.Relay.Connection.html)
for more details on connections.

<p class="notice">
  <strong>Note:</strong> These features do not require using Relay on the client as Apollo
  and other client implementations generally support Relay connection configuration.
</p>
