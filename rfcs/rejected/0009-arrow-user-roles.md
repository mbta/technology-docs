- Feature Name: `arrow-user-roles`
- Start Date: 2021-10-06
- RFC PR: [mbta/technology-docs#9](https://github.com/mbta/technology-docs/pull/9)
- Asana task: n/a
- Status: Rejected

# Summary

[summary]: #summary

This is a proposed method of adding user roles and permissions levels to Arrow, perhaps to be used
by other CTD apps.

# Motivation

[motivation]: #motivation

Arrow is currently only accessible to admins, but we want a read-only view for "viewers", and will
likely have other roles in the future.

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

The user account in Arrow is a struct, `%Accounts.User{}` with fields `:id` (an email from Active
Directory), and `:groups`, the roles they've been assigned to, via Cognito.

The current request's user is always available as `:current_user` assigned in the Plug.

If you want to check if the current user's role allows them to do something, you can use the
`Arrow.Permissions` module, like so:

```ex
def show(conn, params) do
  can_foo? = Permissions.authorized?(conn.assigns.current_user, :do_foo)
  # ...
end
```

If you want to just block a controller action entirely, you can use a [Phoenix Controller plug
pipeline](https://hexdocs.pm/phoenix/Phoenix.Controller.html#module-plug-pipeline):

```ex
defmodule ArrowWeb.BarController do
  use ArrowWeb, :controller

  plug ArrowWeb.CheckPermission, :delete_bar when action in [:delete]

  # ...
end
```

# Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

## User Account and Permissions

A new module will be introduced to represent user accounts:

```ex
defmodule Arrow.Accounts.User do
  defstruct [:id, groups: []]
end
```

and one to work with permissions. This module will centralize all authorization
questions.

```ex
defmodule Arrow.Permissions do
  alias Arrow.Accounts.User

  @required_groups [
    delete_bar: [:admin],
    do_foo: [:admin, :viewer],
  ]

  for {action, groups} <- @required_groups do
    quote do
      def authorized?(%User{} = user, unquote(action)) do
        matches_group(user.groups, groups)
      end
    end
  end

  defp matches_group(user_groups, required_groups) do
    Enum.any?(user_groups, & &1 in required_groups)
  end
end
```

## Load user

A new plug is created and added to the end of the `SignsUiWeb.AuthManager.Pipeline`:

```diff
defmodule SignsUiWeb.AuthManager.Pipeline do
  @moduledoc false

  use Guardian.Plug.Pipeline,
    otp_app: :signs_ui,
    error_handler: SignsUiWeb.AuthManager.ErrorHandler,
    module: SignsUiWeb.AuthManager

  plug(Guardian.Plug.VerifySession, claims: %{"typ" => "access"})
  plug(Guardian.Plug.VerifyHeader, claims: %{"typ" => "access"})
  plug(Guardian.Plug.LoadResource, allow_blank: true)
+ plug(ArrowWeb.Plug.LoadUser)
end
```

```ex
defmodule ArrowWeb.Plug.LoadUser do
  import Plug.Conn

  @cognito_groups %{
    "arrow-viewer" => :viewer,
    "arrow-admin" => :admin
  }

  def init(options), do: options

  def call(conn, _opts) do
    claims = Guardian.Plug.current_claims(conn)

    id = claims["sub"]

    groups =
      claims
      |> Map.get("groups", [])
      |> Enum.flat_map(fn g ->
        Map.get(@cognito_groups, g, [])
      end)

    conn.assign(
      :current_user,
      %Accounts.User{
        id: id,
        groups: groups
      }
    )
  end
end
```

## Group Authorization plug

A new plug is created that can gate entire controllers or actions:

```ex
defmodule ArrowWeb.Plugs.CheckPermission do
  import Plug.Conn

  def init(options), do: options

  def call(%{assigns: %{current_user: current_user}}, action) do
    if Permissions.authorized?(current_user, action) do
      conn
    else
      conn
      |> put_status(401)
      |> redirect(to: "/")
      |> halt()
    end
  end
end
```
