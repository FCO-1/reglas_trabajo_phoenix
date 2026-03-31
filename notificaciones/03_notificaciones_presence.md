# Notificaciones con Presence

Enviar notificaciones solo a usuarios online usando Phoenix.Presence.

---

## Tabla de Contenidos

1. [¿Por qué usar Presence?](#por-que)
2. [Notificaciones Solo Online](#solo-online)
3. [Sistema Híbrido](#sistema-hibrido)
4. [Ejemplos Prácticos](#ejemplos)

---

## 1. ¿Por qué usar Presence? {#por-que}

**Ventajas de combinar Notificaciones + Presence:**

✅ Enviar notificaciones en tiempo real solo a usuarios conectados
✅ Evitar broadcasts innecesarios a usuarios offline
✅ Saber qué usuarios están activos para delivery inmediato
✅ Guardar en BD solo si el usuario está offline

---

## 2. Notificaciones Solo Online {#solo-online}

```elixir
# lib/my_app/notifications.ex
defmodule MyApp.Notifications do
  alias MyApp.Repo
  alias MyApp.Notifications.Notification
  alias MyAppWeb.Presence
  alias Phoenix.PubSub

  def notify_user(user_id, attrs) do
    if user_online?(user_id) do
      # Usuario online: enviar solo por PubSub
      broadcast_notification(user_id, attrs)
      :ok
    else
      # Usuario offline: guardar en BD
      create_notification(Map.put(attrs, :user_id, user_id))
    end
  end

  defp user_online?(user_id) do
    "users:online"
    |> Presence.list()
    |> Map.has_key?("#{user_id}")
  end

  defp broadcast_notification(user_id, attrs) do
    PubSub.broadcast(
      MyApp.PubSub,
      "notifications:#{user_id}",
      {:live_notification, attrs}
    )
  end

  defp create_notification(attrs) do
    %Notification{}
    |> Notification.changeset(attrs)
    |> Repo.insert()
  end
end
```

---

## 3. Sistema Híbrido {#sistema-hibrido}

Mejor práctica: **Guardar siempre en BD** pero broadcast solo si online.

```elixir
defmodule MyApp.Notifications do
  alias MyApp.Repo
  alias MyApp.Notifications.Notification
  alias MyAppWeb.Presence
  alias Phoenix.PubSub

  def create_and_notify(attrs) do
    with {:ok, notification} <- create_notification(attrs) do
      notification = Repo.preload(notification, [:actor])
      
      # Broadcast solo si usuario está online
      if user_online?(notification.user_id) do
        broadcast_notification(notification)
      end
      
      {:ok, notification}
    end
  end

  defp create_notification(attrs) do
    %Notification{}
    |> Notification.changeset(attrs)
    |> Repo.insert()
  end

  defp user_online?(user_id) do
    "users:online"
    |> Presence.list()
    |> Map.has_key?("#{user_id}")
  end

  defp broadcast_notification(notification) do
    PubSub.broadcast(
      MyApp.PubSub,
      "notifications:#{notification.user_id}",
      {:new_notification, notification}
    )
  end

  # Cargar notificaciones pendientes al conectarse
  def load_pending_notifications(user_id) do
    Notification
    |> where(user_id: ^user_id, read: false)
    |> order_by(desc: :inserted_at)
    |> limit(50)
    |> preload([:actor])
    |> Repo.all()
  end
end
```

---

## 4. Ejemplos Prácticos {#ejemplos}

### Notificar Mención en Comentario

```elixir
defmodule MyApp.Comments do
  alias MyApp.Notifications

  def create_comment(attrs) do
    with {:ok, comment} <- do_create_comment(attrs) do
      # Extraer menciones del texto
      mentioned_users = extract_mentions(comment.body)
      
      # Notificar a cada usuario mencionado
      Enum.each(mentioned_users, fn user_id ->
        Notifications.create_and_notify(%{
          type: "mention",
          title: "Te mencionaron",
          body: "#{comment.author.name} te mencionó",
          user_id: user_id,
          actor_id: comment.author_id,
          data: %{
            comment_id: comment.id,
            post_id: comment.post_id
          }
        })
      end)
      
      {:ok, comment}
    end
  end

  defp extract_mentions(text) do
    ~r/@(\w+)/
    |> Regex.scan(text)
    |> Enum.map(fn [_, username] -> 
      find_user_by_username(username)
    end)
    |> Enum.reject(&is_nil/1)
    |> Enum.map(& &1.id)
  end
end
```

### LiveView con Carga de Pendientes

```elixir
defmodule MyAppWeb.DashboardLive do
  use MyAppWeb, :live_view
  alias MyApp.Notifications
  alias MyAppWeb.Presence

  def mount(_params, _session, socket) do
    user_id = socket.assigns.current_user.id

    if connected?(socket) do
      # Suscribirse a notificaciones
      Notifications.subscribe(user_id)
      
      # Trackear como online
      Presence.track(self(), "users:online", "#{user_id}", %{
        user_id: user_id
      })
      
      # Cargar notificaciones pendientes de cuando estaba offline
      pending = Notifications.load_pending_notifications(user_id)
      
      # Mostrar notificaciones pendientes
      if pending != [] do
        send(self(), {:show_pending, pending})
      end
    end

    {:ok, assign(socket, :notifications, [])}
  end

  def handle_info({:show_pending, notifications}, socket) do
    {:noreply,
     socket
     |> assign(:notifications, notifications)
     |> put_flash(:info, "Tienes #{length(notifications)} notificaciones pendientes")}
  end

  def handle_info({:new_notification, notification}, socket) do
    {:noreply,
     socket
     |> update(:notifications, fn list -> [notification | list] end)
     |> put_flash(:info, notification.title)}
  end
end
```

### Notificar Solo Usuarios en Sala de Chat

```elixir
defmodule MyAppWeb.ChatLive do
  use MyAppWeb, :live_view
  alias MyAppWeb.Presence

  def notify_typing(room_id, user_id, username) do
    # Obtener usuarios presentes en esta sala
    online_users = 
      "chat:#{room_id}"
      |> Presence.list()
      |> Map.keys()
      |> Enum.reject(&(&1 == "#{user_id}"))

    # Broadcast solo a usuarios en esta sala
    Phoenix.PubSub.broadcast(
      MyApp.PubSub,
      "chat:#{room_id}",
      {:user_typing, username}
    )
  end
end
```

### Notification Aggregation

```elixir
defmodule MyApp.Notifications do
  # Agrupar notificaciones similares
  def create_or_aggregate(attrs) do
    recent_similar = find_recent_similar(attrs)

    case recent_similar do
      nil ->
        # No hay similar reciente, crear nueva
        create_and_notify(attrs)

      existing ->
        # Ya existe una similar, actualizar contador
        update_notification(existing, %{
          data: increment_counter(existing.data),
          inserted_at: DateTime.utc_now()
        })
    end
  end

  defp find_recent_similar(attrs) do
    cutoff = DateTime.add(DateTime.utc_now(), -300, :second)

    Notification
    |> where(
      user_id: ^attrs.user_id,
      type: ^attrs.type,
      actor_id: ^attrs.actor_id
    )
    |> where([n], n.inserted_at > ^cutoff)
    |> Repo.one()
  end

  defp increment_counter(data) do
    Map.update(data, :count, 2, &(&1 + 1))
  end
end
```

---

## Mejores Prácticas

✅ **Siempre guardar en BD** - Para historial y usuarios offline  
✅ **Broadcast solo si online** - Ahorra recursos  
✅ **Cargar pendientes al conectarse** - UX fluida  
✅ **Agrupar notificaciones similares** - Evitar spam  
✅ **Usar Presence para delivery selectivo** - Solo a usuarios relevantes

❌ **No confiar solo en Presence** - Usuarios pueden desconectarse  
❌ **No enviar demasiadas notificaciones** - Agrupar cuando sea posible  
❌ **No broadcast global** - Usar topics específicos por usuario

---

## Recursos

- Anterior: [LiveView con Notificaciones](02_liveview_notificaciones.md)
- Siguiente: [Testing de Notificaciones](04_testing_notificaciones.md)
- [Phoenix Presence Docs](https://hexdocs.pm/phoenix/Phoenix.Presence.html)
