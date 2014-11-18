###Templates

Templates are what they sound like they should be - files into which we pass data to form complete HTTP responses. For a web application these responses would typically be full HTML documents. For an API, they would most often be JSON or possibly XML. The majority of the code in template files is often markup, but there will also be sections of Elixir code for Phoenix to compile and evaluate.

EEx is the default template system in Phoenix, and it is quite similar to ERB in Ruby. It is actually part of Elixir itself, and Phoenix uses EEx templates to create files like the router and the main application view while generating a new application.

As we learned in the View Guide, by default, templates live in the `web/templates` directory, organized into directories named after a view. Each directory has its own view module to render the templates in it. We can change the template root directory by specifying a new one in the `root: "web/templates"` declaration in the main application view.

###Examples

We've already seen several ways in which templates are used, notably in the Adding Pages Guide and the View Guide. We may cover some of the same territory here, but we will certainly add some new information.

##### Functions in the Main View

Phoenix generates a main application view at `web/view.ex`. Functions we define there are visible to every view module and every template in our application.

Let's make some additions to our application so we can experiment a little.

First, let's define a new route in `web/router.ex` within the `scope "/" do` block.

```elixir
get "/test", Test.PageController, :test
```

Now, let's define the controller action we specified in the route. We'll add a `test/2` action in the `web/controllers/page_controller.ex` file.

```elixir
def test(conn, _params) do
  render conn, "test.html"
end

```

We're going to create a function that tells us which controller and action are handling our request.

To do that, we need to import the `action_name/1` and `controller_module/1` functions from `Phoenix.Controller` in the main view.

```elixir
defmodule Test.View do
  use Phoenix.View, root: "web/templates"

  # The quoted expression returned by this block is applied
  # to this module and all other views that use this module.
  using do
    quote do
      # Import common functionality
      import Test.I18n
      import Test.Router.Helpers
      import Phoenix.Controller, only: [action_name: 1, controller_module: 1] # Add This Import Statement

. . .
```

Next, let's define a `handler_info/1` function at the bottom of the main view which makes use of the `controller_module/1` and `action_name/1` functions we just imported.

```elixir
. . .

  # Functions defined here are available to all other views/templates
  def handler_info(conn) do
    "Request Handled By: #{controller_module conn}.#{action_name conn}"
  end
end

```
We have a route. We created a new controller action. We have made modifications to the main application view. Now all we need is a new template to display the string we get from `handler_info/1`. Let's create a new one at `web/templates/page/test.html.eex`.

```elixir
<div class="jumbotron">
  <p><%= handler_info @conn %></p>
</div>

```

Notice that `@conn` is available to us in the template for free via the `assigns` map.

If we visit [localhost:4000/test](http://localhost:4000/test), we will see that our page is brought to us by `Elixir.HelloPhoenix.PageController.test`.

##### Functions in an Individual View

We can define functions in any individual view in `web/views`. Functions defined in an individual view will only be available to templates which that view renders. For example, functions defined in the `PageView` will only be available to templates in `web/templates/page`.

Let's set ourselves up for the next section by creating a function in `web/views/page_view.ex` to return a list of keys for the `conn` struct.

```elixir
def connection_keys(conn) do
  Map.from_struct(conn)
  |> Map.keys
end

```

Note, it's important to put parenthesis around the `conn` argument to `from_struct/1` because otherwise the pipeline operator will bind more tightly to it than `Map.from_struct/1` causing an error like this one.

```console
**(FunctionClauseError) no function clause matching in Map.from_struct/1

Stacktrace

    (elixir) lib/map.ex:60: Map.from_struct([:__struct__, :adapter, :assigns, :before_send, :cookies, :halted, :host, :method, :params, :path_info, :peer, :port, :private, :query_string, :remote_ip, :req_cookies, :req_headers, :resp_body, :resp_cookies, :resp_headers, :scheme, :script_name, :secret_key_base, :state, :status])
    (test) web/views/page_view.ex:4: Test.PageView.render/2
    (phoenix) lib/phoenix/view.ex:247: Phoenix.View.render_within/3
    (phoenix) lib/phoenix/view.ex:262: Phoenix.View.render_to_iodata/3
    (phoenix) lib/phoenix/controller.ex:451: Phoenix.Controller.render/4
    (test) web/controllers/page_controller.ex:1: Test.PageController.phoenix_controller_stack/2
    (phoenix) lib/phoenix/router/adapter.ex:132: Phoenix.Router.Adapter.dispatch/2
    (test) lib/phoenix/router.ex:2: Test.Router.call/2

```

##### Displaying Lists

So far, we've only displayed singular values in our templates - strings here, and integers in other guides. How would we approach displaying all the elements of a list?

The answer is that we can use Elixir's list comprehensions.

Now that we have a function, visible to our template, that returns a list of keys in the `conn` struct, all we need to do is modify our `web/templates/page/test.html.eex` template a bit do display them.

We can add a header and a list comprehension like this.

```elixir
<div class="jumbotron">
  <p><%= handler_info @conn %></p>

  <h3>Keys for the conn Struct</h3>

  <%= for key <- connection_keys @conn do %>
    <p><%= key %></p>
  <% end %>
</div>
```

We use the list of keys returned by the `connection_keys` function as the source list to iterate over. Note that we need the `=` in both `<%=` - one for the top line of the list comprehension and the other to display the key. Without them, nothing would actually be displayed.

When we visit [localhost:4000/test](http://localhost:4000/test) again, we see all the keys displayed.

##### Partials

In our list comprehension example above, the part that actually displays the values is quite simple.

```elixir
<p><%= key %></p>
```

We are probably fine with leaving this in place. Quite often, however, this display code is somewhat more complex, and putting it in the middle of a list comprehension makes our templates harder to read.

That's where partials come in. Partials are templates, usually quite small, which are rendered within other templates. This is simply a continuation of the rendering chain we have already seen. Layouts are templates into which regular templates are rendered. Regular templates may have partial templates rendered into them.

Let's turn this display snippet into a partial. Create a new template file at `web/templates/page/_key.html.eex`, like this.

```elixir
<p><%= @key %></p>
```
We need to change `key` to `@key` here because this is a new template, not part of a list comprehension. The way we pass data into a template is by the `assigns` map, and the way we get the values out of the `assigns` map is by referencing the keys with a preceeding `@`.

Now that we have a template, we simply render it within our list comprehension in the `test.html.eex` template.

```elixir
<%= for key <- connection_keys @conn do %>
  <%= render "_key.html", key: key %>
<% end %>
```

Let's take a look at [localhost:4000/test](http://localhost:4000/test) again. The page should look exactly as it did before.

##### Shared Partials

Often, we find that small pieces of data need to be rendered the same way in different parts of the application. It's a good practice to move these partials into their own shared directory to indicate that they ought to be available anywhere in the app.

Let's convert our partial into a shared partial.

`_key.html.eex` is currently rendered by the `HelloPhoenix.PageView` module, but we use a render call which assumes that the current view model is what we want to render with. We could make that explicit, and re-write it like this:

```elixir
<%= for key <- connection_keys @conn do %>
  <%= render HelloPhoenix.PageView, "_key.html", key: key %>
<% end %>
```

Since we want this to live in a new `web/templates/shared` directory, we need a new individual view to render templates in that directory, `web/views/shared_view.ex`.

```elixir
defmodule HelloPhoenix.SharedView do
  use HelloPhoenix.View
end
```

Now we can move `_key.html.eex` from the `web/templates/page` directory into the `web/templates/shared` directory. Once that happens, we can change the render call to use the new `HelloPhoenix.SharedView`.

```elixir
<%= for key <- connection_keys @conn do %>
  <%= render HelloPhoenix.SharedView, "_key.html", key: key %>
<% end %>
```
Going back to [localhost:4000/test](http://localhost:4000/test) again. The page should look exactly as it did before.

##### Configuring a New Template Engine
TODO
Phoenix has the concept of a template engine, a module that receives a template path and transforms its source code into Elixir quoted expressions.
