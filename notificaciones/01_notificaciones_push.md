# Notificaciones Push en Phoenix

Sistema de notificaciones en tiempo real con PubSub.

## Schema

```elixir
create table(:notifications) do
  add :user_id, references(:users), null: false
  add :type, :string, null: false
  add :title, :string, null: false
  add :body, :text
  add :read, :boolean, default: false
  timestamps()
end
```

## Contexto

```elixir
defmodule MyApp.Notifications do
  alias Phoenix.PubSub

  def create_notification(attrs) do
    with {:ok, notif} <- do_create(attrs) do
      broadcast_notification(notif)
      {:ok, notif}
    end
  end

  defp broadcast_notification(notif) do
    PubSub.broadcast(
      MyApp.PubSub,
      "notifications:#{notif.user_id}",
      {:new_notification, notif}
    )
  end

  def subscribe(user_id) do
    PubSub.subscribe(MyApp.PubSub, "notifications:#{user_id}")
  end
end
```

## LiveView

```elixir
def mount(_params, _session, socket) do
  if connected?(socket) do
    Notifications.subscribe(socket.assigns.current_user.id)
  end

  {:ok, assign(socket, :notifications, [])}
end

def handle_info({:new_notification, notif}, socket) do
  {:noreply,
   socket
   |> update(:notifications, fn list -> [notif | list] end)
   |> put_flash(:info, notif.title)}
end
```
