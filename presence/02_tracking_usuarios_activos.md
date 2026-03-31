# Tracking de Usuarios Activos con Presence

Implementación de tracking de usuarios online/offline en tiempo real.

---

## Tabla de Contenidos

1. [Estados de Usuario](#estados-usuario)
2. [LiveView con Tracking](#liveview-tracking)
3. [Indicadores Visuales](#indicadores-visuales)
4. [Actualización de Estado](#actualizacion-estado)

---

## 1. Estados de Usuario {#estados-usuario}

### Metadatos Comunes

```elixir
# Metadata básica
%{
  user_id: user.id,
  username: user.name,
  online_at: System.system_time(:second)
}

# Metadata extendida
%{
  user_id: user.id,
  username: user.name,
  avatar: user.avatar_url,
  status: "online",  # online, away, busy, offline
  current_page: "/dashboard",
  last_activity: DateTime.utc_now(),
  device: "web"  # web, mobile, desktop
}
```

### Estados Posibles

```elixir
defmodule MyApp.UserStatus do
  @statuses ["online", "away", "busy", "offline"]

  def valid_status?(status), do: status in @statuses

  def status_color(status) do
    case status do
      "online" -> "bg-green-500"
      "away" -> "bg-yellow-500"
      "busy" -> "bg-red-500"
      "offline" -> "bg-gray-400"
    end
  end

  def status_label(status) do
    case status do
      "online" -> "En línea"
      "away" -> "Ausente"
      "busy" -> "Ocupado"
      "offline" -> "Desconectado"
    end
  end
end
```

---

## 2. LiveView con Tracking {#liveview-tracking}

### LiveView Principal

```elixir
# lib/my_app_web/live/users_online_live.ex
defmodule MyAppWeb.UsersOnlineLive do
  use MyAppWeb, :live_view
  alias MyAppWeb.Presence
  alias Phoenix.PubSub

  @topic "users:online"

  def mount(_params, _session, socket) do
    current_user = socket.assigns.current_user

    if connected?(socket) do
      PubSub.subscribe(MyApp.PubSub, @topic)
      
      # Track usuario actual
      {:ok, _} = Presence.track(self(), @topic, "#{current_user.id}", %{
        user_id: current_user.id,
        username: current_user.name,
        avatar: current_user.avatar_url,
        status: "online",
        joined_at: System.system_time(:second)
      })
    end

    presences = Presence.list(@topic)

    {:ok,
     socket
     |> assign(:presences, presences)
     |> assign(:online_users, list_online_users(presences))
     |> assign(:current_user_id, current_user.id)}
  end

  def handle_info(%{event: "presence_diff", payload: diff}, socket) do
    presences = Presence.sync_diff(socket.assigns.presences, diff)
    online_users = list_online_users(presences)

    {:noreply,
     socket
     |> assign(:presences, presences)
     |> assign(:online_users, online_users)}
  end

  defp list_online_users(presences) do
    presences
    |> Enum.map(fn {_id, %{metas: metas}} ->
      List.first(metas)
    end)
    |> Enum.sort_by(& &1.username)
  end
end
```

### Template

```heex
<!-- lib/my_app_web/live/users_online_live.html.heex -->
<Layouts.app flash={@flash}>
  <div class="max-w-4xl mx-auto p-6">
    <h1 class="text-3xl font-bold mb-6">
      Usuarios en Línea (<%= length(@online_users) %>)
    </h1>

    <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
      <%= for user <- @online_users do %>
        <div class={[
          "p-4 rounded-lg border transition-all",
          user.user_id == @current_user_id && "ring-2 ring-blue-500"
        ]}>
          <div class="flex items-center gap-3">
            <!-- Avatar -->
            <div class="relative">
              <%= if user.avatar do %>
                <img 
                  src={user.avatar} 
                  alt={user.username}
                  class="w-12 h-12 rounded-full"
                />
              <% else %>
                <div class="w-12 h-12 rounded-full bg-gray-300 flex items-center justify-center">
                  <span class="text-xl font-bold text-gray-600">
                    <%= String.first(user.username) %>
                  </span>
                </div>
              <% end %>
              
              <!-- Status indicator -->
              <div class={[
                "absolute bottom-0 right-0 w-4 h-4 rounded-full border-2 border-white",
                status_color(user[:status] || "online")
              ]}></div>
            </div>

            <!-- User info -->
            <div class="flex-1">
              <p class="font-semibold">
                <%= user.username %>
                <%= if user.user_id == @current_user_id do %>
                  <span class="text-sm text-gray-500">(tú)</span>
                <% end %>
              </p>
              <p class="text-sm text-gray-600">
                <%= status_label(user[:status] || "online") %>
              </p>
            </div>
          </div>

          <!-- Tiempo online -->
          <div class="mt-3 text-xs text-gray-500">
            En línea desde hace <%= format_duration(user.joined_at) %>
          </div>
        </div>
      <% end %>
    </div>
  </div>
</Layouts.app>
```

---

## 3. Indicadores Visuales {#indicadores-visuales}

### Componente de Badge de Usuario

```elixir
# lib/my_app_web/components/user_badge.ex
defmodule MyAppWeb.Components.UserBadge do
  use Phoenix.Component

  attr :user, :map, required: true
  attr :show_status, :boolean, default: true
  attr :size, :string, default: "md"

  def user_badge(assigns) do
    ~H"""
    <div class="flex items-center gap-2">
      <div class="relative">
        <img 
          src={@user.avatar || "/images/default-avatar.png"}
          alt={@user.name}
          class={avatar_size(@size)}
        />
        <%= if @show_status do %>
          <div class={[
            "absolute bottom-0 right-0 rounded-full border-2 border-white",
            status_indicator_size(@size),
            user_status_color(@user)
          ]}></div>
        <% end %>
      </div>
      <span class={username_size(@size)}><%= @user.name %></span>
    </div>
    """
  end

  defp avatar_size("sm"), do: "w-8 h-8 rounded-full"
  defp avatar_size("md"), do: "w-12 h-12 rounded-full"
  defp avatar_size("lg"), do: "w-16 h-16 rounded-full"

  defp status_indicator_size("sm"), do: "w-2 h-2"
  defp status_indicator_size("md"), do: "w-3 h-3"
  defp status_indicator_size("lg"), do: "w-4 h-4"

  defp username_size("sm"), do: "text-sm"
  defp username_size("md"), do: "text-base"
  defp username_size("lg"), do: "text-lg font-semibold"

  defp user_status_color(user) do
    if user_online?(user.id) do
      "bg-green-500"
    else
      "bg-gray-400"
    end
  end

  defp user_online?(user_id) do
    "users:online"
    |> MyAppWeb.Presence.list()
    |> Map.has_key?("#{user_id}")
  end
end
```

### Lista Compacta de Usuarios Online

```heex
<div class="fixed bottom-4 right-4 bg-white shadow-lg rounded-lg p-4 max-w-xs">
  <h3 class="font-bold mb-3">En Línea Ahora</h3>
  
  <div class="space-y-2 max-h-64 overflow-y-auto">
    <%= for user <- @online_users do %>
      <div class="flex items-center gap-2 hover:bg-gray-50 p-1 rounded">
        <div class="w-2 h-2 bg-green-500 rounded-full"></div>
        <span class="text-sm"><%= user.username %></span>
      </div>
    <% end %>
  </div>
</div>
```

---

## 4. Actualización de Estado {#actualizacion-estado}

### Cambiar Estado Manualmente

```elixir
def handle_event("change_status", %{"status" => new_status}, socket) do
  user_id = socket.assigns.current_user.id

  Presence.update(self(), "users:online", "#{user_id}", fn meta ->
    Map.put(meta, :status, new_status)
  end)

  {:noreply, put_flash(socket, :info, "Estado actualizado")}
end
```

### Selector de Estado

```heex
<div class="flex gap-2">
  <%= for status <- ["online", "away", "busy"] do %>
    <button
      phx-click="change_status"
      phx-value-status={status}
      class={[
        "px-3 py-1 rounded-lg text-sm",
        "hover:ring-2 ring-blue-300"
      ]}
    >
      <div class="flex items-center gap-2">
        <div class={"w-2 h-2 rounded-full #{status_color(status)}"}></div>
        <%= status_label(status) %>
      </div>
    </button>
  <% end %>
</div>
```

### Auto-away por Inactividad

```elixir
# assets/js/app.js
let Hooks = {}

Hooks.ActivityTracker = {
  mounted() {
    this.timeout = null
    this.awayTimeout = 5 * 60 * 1000  // 5 minutos

    const resetTimer = () => {
      clearTimeout(this.timeout)
      
      this.timeout = setTimeout(() => {
        this.pushEvent("set_away", {})
      }, this.awayTimeout)
      
      this.pushEvent("set_active", {})
    }

    window.addEventListener("mousemove", resetTimer)
    window.addEventListener("keypress", resetTimer)
    
    resetTimer()
  }
}
```

```elixir
def handle_event("set_away", _params, socket) do
  user_id = socket.assigns.current_user.id

  Presence.update(self(), "users:online", "#{user_id}", fn meta ->
    Map.put(meta, :status, "away")
  end)

  {:noreply, socket}
end

def handle_event("set_active", _params, socket) do
  user_id = socket.assigns.current_user.id

  Presence.update(self(), "users:online", "#{user_id}", fn meta ->
    Map.put(meta, :status, "online")
  end)

  {:noreply, socket}
end
```

---

## Helpers

```elixir
defp format_duration(timestamp) do
  now = System.system_time(:second)
  diff = now - timestamp

  cond do
    diff < 60 -> "#{diff}s"
    diff < 3600 -> "#{div(diff, 60)}m"
    diff < 86400 -> "#{div(diff, 3600)}h"
    true -> "#{div(diff, 86400)}d"
  end
end

defp status_color(status) do
  case status do
    "online" -> "bg-green-500"
    "away" -> "bg-yellow-500"
    "busy" -> "bg-red-500"
    _ -> "bg-gray-400"
  end
end

defp status_label(status) do
  case status do
    "online" -> "En línea"
    "away" -> "Ausente"
    "busy" -> "Ocupado"
    _ -> "Desconectado"
  end
end
```

---

## Recursos

- Anterior: [Setup de Presence](01_presence_setup.md)
- Siguiente: [Presence por Sala/Canal](03_presence_salas.md)
