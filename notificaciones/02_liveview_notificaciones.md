# LiveView con Notificaciones

Implementación de UI de notificaciones en tiempo real con LiveView.

---

## Tabla de Contenidos

1. [LiveView Principal](#liveview-principal)
2. [Template de Notificaciones](#template)
3. [Componente de Badge](#badge)
4. [Manejo de Eventos](#eventos)

---

## 1. LiveView Principal {#liveview-principal}

```elixir
# lib/my_app_web/live/notifications_live.ex
defmodule MyAppWeb.NotificationsLive do
  use MyAppWeb, :live_view
  alias MyApp.Notifications

  def mount(_params, _session, socket) do
    user_id = socket.assigns.current_user.id

    if connected?(socket) do
      Notifications.subscribe(user_id)
    end

    notifications = Notifications.list_notifications(user_id)
    unread_count = Notifications.count_unread(user_id)

    {:ok,
     socket
     |> assign(:notifications, notifications)
     |> assign(:unread_count, unread_count)
     |> assign(:show_dropdown, false)}
  end

  # Nueva notificación recibida
  def handle_info({:new_notification, notification}, socket) do
    {:noreply,
     socket
     |> update(:notifications, fn notifs -> [notification | notifs] end)
     |> update(:unread_count, &(&1 + 1))
     |> put_flash(:info, notification.title)}
  end

  # Notificación marcada como leída
  def handle_info({:notification_read, notification_id}, socket) do
    {:noreply,
     socket
     |> update(:notifications, fn notifs ->
       Enum.map(notifs, fn n ->
         if n.id == notification_id, do: %{n | read: true}, else: n
       end)
     end)
     |> update(:unread_count, &max(&1 - 1, 0))}
  end

  # Todas las notificaciones marcadas como leídas
  def handle_info(:all_notifications_read, socket) do
    {:noreply,
     socket
     |> update(:notifications, fn notifs ->
       Enum.map(notifs, &%{&1 | read: true})
     end)
     |> assign(:unread_count, 0)}
  end

  def handle_event("toggle_dropdown", _params, socket) do
    {:noreply, update(socket, :show_dropdown, &(!&1))}
  end

  def handle_event("mark_as_read", %{"id" => id}, socket) do
    notification_id = String.to_integer(id)
    user_id = socket.assigns.current_user.id

    case Notifications.mark_as_read(notification_id, user_id) do
      {:ok, _} ->
        {:noreply, socket}

      {:error, _} ->
        {:noreply, put_flash(socket, :error, "Error al marcar como leída")}
    end
  end

  def handle_event("mark_all_as_read", _params, socket) do
    user_id = socket.assigns.current_user.id

    case Notifications.mark_all_as_read(user_id) do
      {:ok, _count} ->
        {:noreply, put_flash(socket, :info, "Todas marcadas como leídas")}

      {:error, _} ->
        {:noreply, put_flash(socket, :error, "Error al marcar")}
    end
  end
end
```

---

## 2. Template de Notificaciones {#template}

```heex
<!-- lib/my_app_web/live/notifications_live.html.heex -->
<Layouts.app flash={@flash}>
  <div class="relative max-w-4xl mx-auto p-6">
    <!-- Header con contador -->
    <div class="flex justify-between items-center mb-6">
      <h1 class="text-3xl font-bold">
        Notificaciones
        <%= if @unread_count > 0 do %>
          <span class="ml-3 px-3 py-1 text-sm bg-red-500 text-white rounded-full">
            <%= @unread_count %>
          </span>
        <% end %>
      </h1>

      <%= if @unread_count > 0 do %>
        <button
          phx-click="mark_all_as_read"
          class="px-4 py-2 text-sm text-blue-600 hover:text-blue-800"
        >
          Marcar todas como leídas
        </button>
      <% end %>
    </div>

    <!-- Lista de notificaciones -->
    <div class="space-y-3">
      <%= if @notifications == [] do %>
        <div class="text-center py-12 text-gray-500">
          <p class="text-lg">No tienes notificaciones</p>
        </div>
      <% else %>
        <%= for notification <- @notifications do %>
          <div
            id={"notification-#{notification.id}"}
            class={[
              "p-4 rounded-lg border transition-colors",
              notification.read && "bg-white border-gray-200",
              !notification.read && "bg-blue-50 border-blue-300"
            ]}
          >
            <div class="flex items-start justify-between gap-4">
              <!-- Contenido de la notificación -->
              <div class="flex-1">
                <div class="flex items-center gap-2 mb-1">
                  <%= if !notification.read do %>
                    <div class="w-2 h-2 bg-blue-600 rounded-full"></div>
                  <% end %>
                  <h3 class="font-semibold text-gray-900">
                    <%= notification.title %>
                  </h3>
                </div>

                <%= if notification.body do %>
                  <p class="text-gray-700 text-sm">
                    <%= notification.body %>
                  </p>
                <% end %>

                <p class="text-xs text-gray-500 mt-2">
                  <%= format_time(notification.inserted_at) %>
                </p>
              </div>

              <!-- Botón marcar como leída -->
              <%= if !notification.read do %>
                <button
                  phx-click="mark_as_read"
                  phx-value-id={notification.id}
                  class="text-blue-600 hover:text-blue-800 text-sm"
                >
                  Marcar leída
                </button>
              <% end %>
            </div>
          </div>
        <% end %>
      <% end %>
    </div>
  </div>
</Layouts.app>
```

---

## 3. Componente de Badge {#badge}

```elixir
# lib/my_app_web/live/notifications_badge_live.ex
defmodule MyAppWeb.NotificationsBadgeLive do
  use MyAppWeb, :live_view
  alias MyApp.Notifications

  def mount(_params, _session, socket) do
    user_id = socket.assigns.current_user.id

    if connected?(socket) do
      Notifications.subscribe(user_id)
    end

    unread_count = Notifications.count_unread(user_id)

    {:ok, assign(socket, :unread_count, unread_count)}
  end

  def handle_info({:new_notification, _notification}, socket) do
    {:noreply, update(socket, :unread_count, &(&1 + 1))}
  end

  def handle_info({:notification_read, _id}, socket) do
    {:noreply, update(socket, :unread_count, &max(&1 - 1, 0))}
  end

  def handle_info(:all_notifications_read, socket) do
    {:noreply, assign(socket, :unread_count, 0)}
  end

  def render(assigns) do
    ~H"""
    <.link navigate={~p"/notifications"} class="relative">
      <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24">
        <path
          stroke-linecap="round"
          stroke-linejoin="round"
          stroke-width="2"
          d="M15 17h5l-1.405-1.405A2.032 2.032 0 0118 14.158V11a6.002 6.002 0 00-4-5.659V5a2 2 0 10-4 0v.341C7.67 6.165 6 8.388 6 11v3.159c0 .538-.214 1.055-.595 1.436L4 17h5m6 0v1a3 3 0 11-6 0v-1m6 0H9"
        />
      </svg>

      <%= if @unread_count > 0 do %>
        <span class="absolute -top-1 -right-1 px-1.5 py-0.5 text-xs bg-red-500 text-white rounded-full">
          <%= @unread_count %>
        </span>
      <% end %>
    </.link>
    """
  end
end
```

---

## 4. Manejo de Eventos {#eventos}

### Helper para Formato de Tiempo

```elixir
# lib/my_app_web/live/notifications_live.ex
defp format_time(datetime) do
  now = DateTime.utc_now()
  diff_seconds = DateTime.diff(now, datetime)

  cond do
    diff_seconds < 60 ->
      "Hace #{diff_seconds}s"

    diff_seconds < 3600 ->
      "Hace #{div(diff_seconds, 60)}m"

    diff_seconds < 86400 ->
      "Hace #{div(diff_seconds, 3600)}h"

    diff_seconds < 604800 ->
      "Hace #{div(diff_seconds, 86400)}d"

    true ->
      Calendar.strftime(datetime, "%d/%m/%Y")
  end
end
```

### Agregar Badge al Layout

```heex
<!-- lib/my_app_web/components/layouts/app.html.heex -->
<header class="bg-white shadow">
  <div class="flex items-center justify-between px-6 py-4">
    <h1>Mi App</h1>

    <div class="flex items-center gap-4">
      <%= if @current_user do %>
        <.live_render
          @conn,
          MyAppWeb.NotificationsBadgeLive,
          id="notifications-badge"
        />
      <% end %>
    </div>
  </div>
</header>
```

---

## Recursos

- Anterior: [Arquitectura de Notificaciones](01_arquitectura_notificaciones.md)
- Siguiente: [Notificaciones con Presence](03_notificaciones_presence.md)
