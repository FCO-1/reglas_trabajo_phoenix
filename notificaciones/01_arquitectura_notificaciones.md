# Arquitectura de Notificaciones Push

Sistema de notificaciones en tiempo real con Phoenix PubSub.

---

## Tabla de Contenidos

1. [Componentes del Sistema](#componentes)
2. [Schema de Notificaciones](#schema)
3. [Migración](#migracion)
4. [Contexto Básico](#contexto)

---

## 1. Componentes del Sistema {#componentes}

Un sistema completo de notificaciones requiere:

1. **Schema** - Modelo de datos para persistir notificaciones
2. **Contexto** - Lógica de negocio y broadcasting
3. **LiveView** - UI para mostrar notificaciones
4. **PubSub** - Canal para enviar notificaciones en tiempo real
5. **Presence** (opcional) - Para notificaciones solo a usuarios online

---

## 2. Schema de Notificaciones {#schema}

```elixir
# lib/my_app/notifications/notification.ex
defmodule MyApp.Notifications.Notification do
  use Ecto.Schema
  import Ecto.Changeset

  schema "notifications" do
    field :type, :string
    field :title, :string
    field :body, :text
    field :read, :boolean, default: false
    field :read_at, :utc_datetime
    field :data, :map, default: %{}
    
    belongs_to :user, MyApp.Accounts.User
    belongs_to :actor, MyApp.Accounts.User
    
    timestamps()
  end

  def changeset(notification, attrs) do
    notification
    |> cast(attrs, [:type, :title, :body, :read, :read_at, :data, :user_id, :actor_id])
    |> validate_required([:type, :title, :user_id])
    |> validate_inclusion(:type, [
      "mention",
      "comment",
      "like",
      "follow",
      "system"
    ])
  end
end
```

---

## 3. Migración {#migracion}

```elixir
# priv/repo/migrations/20240101000000_create_notifications.exs
defmodule MyApp.Repo.Migrations.CreateNotifications do
  use Ecto.Migration

  def change do
    create table(:notifications) do
      add :type, :string, null: false
      add :title, :string, null: false
      add :body, :text
      add :read, :boolean, default: false, null: false
      add :read_at, :utc_datetime
      add :data, :map, default: %{}
      
      add :user_id, references(:users, on_delete: :delete_all), null: false
      add :actor_id, references(:users, on_delete: :nilify_all)
      
      timestamps()
    end

    create index(:notifications, [:user_id])
    create index(:notifications, [:user_id, :read])
    create index(:notifications, [:inserted_at])
  end
end
```

---

## 4. Contexto Básico {#contexto}

```elixir
# lib/my_app/notifications.ex
defmodule MyApp.Notifications do
  import Ecto.Query
  alias MyApp.Repo
  alias MyApp.Notifications.Notification
  alias Phoenix.PubSub

  @topic_prefix "notifications:"

  def list_notifications(user_id, opts \\ []) do
    limit = Keyword.get(opts, :limit, 50)
    unread_only = Keyword.get(opts, :unread_only, false)

    Notification
    |> where(user_id: ^user_id)
    |> maybe_filter_unread(unread_only)
    |> order_by(desc: :inserted_at)
    |> limit(^limit)
    |> preload([:actor])
    |> Repo.all()
  end

  defp maybe_filter_unread(query, true), do: where(query, read: false)
  defp maybe_filter_unread(query, false), do: query

  def count_unread(user_id) do
    Notification
    |> where(user_id: ^user_id, read: false)
    |> Repo.aggregate(:count)
  end

  def create_notification(attrs) do
    %Notification{}
    |> Notification.changeset(attrs)
    |> Repo.insert()
    |> case do
      {:ok, notification} = result ->
        notification = Repo.preload(notification, [:actor])
        broadcast_notification(notification)
        result

      error ->
        error
    end
  end

  def mark_as_read(notification_id, user_id) do
    notification = 
      Notification
      |> where(id: ^notification_id, user_id: ^user_id)
      |> Repo.one()

    case notification do
      nil ->
        {:error, :not_found}

      notif ->
        notif
        |> Notification.changeset(%{
          read: true,
          read_at: DateTime.utc_now()
        })
        |> Repo.update()
        |> tap(fn {:ok, _} -> 
          broadcast_read(notif.user_id, notif.id)
        end)
    end
  end

  def mark_all_as_read(user_id) do
    {count, _} =
      Notification
      |> where(user_id: ^user_id, read: false)
      |> Repo.update_all(set: [
        read: true,
        read_at: DateTime.utc_now()
      ])

    broadcast_all_read(user_id)
    {:ok, count}
  end

  defp broadcast_notification(notification) do
    PubSub.broadcast(
      MyApp.PubSub,
      topic(notification.user_id),
      {:new_notification, notification}
    )
  end

  defp broadcast_read(user_id, notification_id) do
    PubSub.broadcast(
      MyApp.PubSub,
      topic(user_id),
      {:notification_read, notification_id}
    )
  end

  defp broadcast_all_read(user_id) do
    PubSub.broadcast(
      MyApp.PubSub,
      topic(user_id),
      :all_notifications_read
    )
  end

  def subscribe(user_id) do
    PubSub.subscribe(MyApp.PubSub, topic(user_id))
  end

  defp topic(user_id), do: @topic_prefix <> "#{user_id}"
end
```

---

## Tipos de Notificaciones Comunes

```elixir
# Mención en comentario
%{
  type: "mention",
  title: "Te mencionaron en un comentario",
  body: "Juan te mencionó en un post",
  user_id: mentioned_user_id,
  actor_id: juan_id,post_id: 123,
    comment_id: 456
  }
}

# Nuevo seguidor
%{
  type: "follow",
  title: "Nuevo seguidor",
  body: "María comenzó a seguirte",
  user_id: followed_user_id,
  actor_id: maria_id
}

# Like en post
%{
  type: "like",
  title: "Le gustó tu post",
  body: "A Pedro le gustó tu post",
  user_id: post_author_id,
  actor_id: pedro_id,post_id: 789
  }
}

# Notificación del sistema
%{
  type: "system",
  title: "Actualización del sistema",
  body: "Nueva versión disponible",
  user_id: user_id,
  actor_id: nil
}
```

---

## Recursos

- Siguiente: [LiveView con Notificaciones](02_liveview_notificaciones.md)
- [Phoenix PubSub Docs](https://hexdocs.pm/phoenix_pubsub)
