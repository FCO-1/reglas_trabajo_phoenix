# Presence por Salas y Canales

Implementación de tracking de usuarios por sala/canal específico.

---

## Tabla de Contenidos

1. [Conceptos Básicos](#conceptos)
2. [Chat Room con Presence](#chat-room)
3. [Documento Colaborativo](#documento-colaborativo)
4. [Juego Multijugador](#juego-multijugador)

---

## 1. Conceptos Básicos {#conceptos}

### Topics por Sala

```elixir
# Patrón común para topics
"room:#{room_id}"           # Chat room
"document:#{doc_id}"        # Documento colaborativo
"game:#{game_id}"           # Juego
"channel:#{channel_id}"     # Canal de video/audio
```

### Múltiples Presences

Un usuario puede estar tracked en múltiples salas simultáneamente:

```elixir
# Usuario en múltiples rooms
Presence.track(self(), "room:lobby", user_id, %{...})
Presence.track(self(), "room:general", user_id, %{...})
Presence.track(self(), "document:123", user_id, %{...})
```

---

## 2. Chat Room con Presence {#chat-room}

### LiveView

```elixir
# lib/my_app_web/live/chat_room_live.ex
defmodule MyAppWeb.ChatRoomLive do
  use MyAppWeb, :live_view
  alias MyAppWeb.Presence
  alias MyApp.Chat
  alias Phoenix.PubSub

  def mount(%{"room_id" => room_id}, _session, socket) do
    room = Chat.get_room!(room_id)
    topic = "room:#{room_id}"
    current_user = socket.assigns.current_user

    if connected?(socket) do
      # Suscribirse a mensajes y presence
      PubSub.subscribe(MyApp.PubSub, topic)
      
      # Track usuario en esta sala
      {:ok, _} = Presence.track(self(), topic, "#{current_user.id}", %{
        user_id: current_user.id,
        username: current_user.name,
        avatar: current_user.avatar_url,
        joined_at: System.system_time(:second)
      })
    end

    messages = Chat.list_messages(room_id)
    presences = Presence.list(topic)

    {:ok,
     socket
     |> assign(:room, room)
     |> assign(:topic, topic)
     |> assign(:messages, messages)
     |> assign(:presences, presences)
     |> assign(:online_users, list_users(presences))
     |> assign(:message_form, to_form(%{"content" => ""}))}
  end

  # Nuevo mensaje
  def handle_info({:new_message, message}, socket) do
    {:noreply, update(socket, :messages, fn msgs -> [message | msgs] end)}
  end

  # Cambios de presence
  def handle_info(%{event: "presence_diff", payload: diff}, socket) do
    presences = Presence.sync_diff(socket.assigns.presences, diff)

    {:noreply,
     socket
     |> assign(:presences, presences)
     |> assign(:online_users, list_users(presences))}
  end

  # Enviar mensaje
  def handle_event("send_message", %{"content" => content}, socket) do
    case Chat.create_message(socket.assigns.room.id, %{
      content: content,
      user_id: socket.assigns.current_user.id
    }) do
      {:ok, _message} ->
        {:noreply, assign(socket, :message_form, to_form(%{"content" => ""}))}

      {:error, _changeset} ->
        {:noreply, put_flash(socket, :error, "Error al enviar mensaje")}
    end
  end

  defp list_users(presences) do
    presences
    |> Enum.map(fn {_id, %{metas: metas}} -> List.first(metas) end)
    |> Enum.sort_by(& &1.username)
  end
end
```

### Template

```heex
<!-- lib/my_app_web/live/chat_room_live.html.heex -->
<Layouts.app flash={@flash}>
  <div class="flex h-screen">
    <!-- Sidebar con usuarios online -->
    <div class="w-64 bg-gray-100 p-4 overflow-y-auto">
      <h2 class="font-bold mb-4">
        En línea (<%= length(@online_users) %>)
      </h2>
      
      <div class="space-y-2">
        <%= for user <- @online_users do %>
          <div class="flex items-center gap-2 p-2 hover:bg-gray-200 rounded">
            <div class="w-2 h-2 bg-green-500 rounded-full"></div>
            <img 
              src={user.avatar} 
              alt={user.username}
              class="w-8 h-8 rounded-full"
            />
            <span class="text-sm"><%= user.username %></span>
          </div>
        <% end %>
      </div>
    </div>

    <!-- Chat principal -->
    <div class="flex-1 flex flex-col">
      <!-- Header -->
      <div class="bg-white border-b p-4">
        <h1 class="text-xl font-bold"><%= @room.name %></h1>
      </div>

      <!-- Mensajes -->
      <div class="flex-1 overflow-y-auto p-4 space-y-4">
        <%= for message <- Enum.reverse(@messages) do %>
          <div class="flex gap-3">
            <img 
              src={message.user.avatar}
              alt={message.user.name}
              class="w-10 h-10 rounded-full"
            />
            <div>
              <div class="flex items-baseline gap-2">
                <span class="font-semibold"><%= message.user.name %></span>
                <span class="text-xs text-gray-500">
                  <%= Calendar.strftime(message.inserted_at, "%H:%M") %>
                </span>
              </div>
              <p class="mt-1"><%= message.content %></p>
            </div>
          </div>
        <% end %>
      </div>

      <!-- Input de mensaje -->
      <div class="bg-white border-t p-4">
        <.form for={@message_form} phx-submit="send_message" class="flex gap-2">
          <input 
            type="text"
            name="content"
            value={@message_form[:content].value}
            placeholder="Escribe un mensaje..."
            class="flex-1 px-4 py-2 border rounded-lg"
            autocomplete="off"
          />
          <button 
            type="submit"
            class="px-6 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700"
          >
            Enviar
          </button>
        </.form>
      </div>
    </div>
  </div>
</Layouts.app>
```

---

## 3. Documento Colaborativo {#documento-colaborativo}

### LiveView con Cursores

```elixir
# lib/my_app_web/live/document_live.ex
defmodule MyAppWeb.DocumentLive do
  use MyAppWeb, :live_view
  alias MyAppWeb.Presence

  def mount(%{"id" => doc_id}, _session, socket) do
    document = Documents.get_document!(doc_id)
    topic = "document:#{doc_id}"
    current_user = socket.assigns.current_user

    if connected?(socket) do
      Phoenix.PubSub.subscribe(MyApp.PubSub, topic)
      
      {:ok, _} = Presence.track(self(), topic, "#{current_user.id}", %{
        user_id: current_user.id,
        username: current_user.name,
        color: random_color(),
        cursor: nil,
        selection: nil
      })
    end

    presences = Presence.list(topic)

    {:ok,
     socket
     |> assign(:document, document)
     |> assign(:topic, topic)
     |> assign(:presences, presences)
     |> assign(:viewers, list_viewers(presences))}
  end

  # Actualizar posición del cursor
  def handle_event("cursor_move", %{"x" => x, "y" => y}, socket) do
    user_id = socket.assigns.current_user.id

    Presence.update(self(), socket.assigns.topic, "#{user_id}", fn meta ->
      Map.put(meta, :cursor, %{x: x, y: y})
    end)

    {:noreply, socket}
  end

  # Actualizar selección de texto
  def handle_event("text_select", %{"start" => start, "end" => end_pos}, socket) do
    user_id = socket.assigns.current_user.id

    Presence.update(self(), socket.assigns.topic, "#{user_id}", fn meta ->
      Map.put(meta, :selection, %{start: start, end: end_pos})
    end)

    {:noreply, socket}
  end

  def handle_info(%{event: "presence_diff"}, socket) do
    presences = Presence.list(socket.assigns.topic)

    {:noreply,
     socket
     |> assign(:presences, presences)
     |> assign(:viewers, list_viewers(presences))}
  end

  defp list_viewers(presences) do
    presences
    |> Enum.map(fn {_id, %{metas: metas}} -> List.first(metas) end)
  end

  defp random_color do
    colors = ["#EF4444", "#10B981", "#3B82F6", "#F59E0B", "#8B5CF6"]
    Enum.random(colors)
  end
end
```

### Template con Cursores

```heex
<Layouts.app flash={@flash}>
  <div class="max-w-6xl mx-auto p-6">
    <!-- Viewers online -->
    <div class="mb-4 flex items-center gap-2">
      <span class="text-sm text-gray-600">Viendo ahora:</span>
      <%= for viewer <- @viewers do %>
        <div 
          class="flex items-center gap-1 px-2 py-1 rounded-full text-sm"
          style={"background-color: #{viewer.color}20; color: #{viewer.color}"}
        >
          <%= viewer.username %>
        </div>
      <% end %>
    </div>

    <!-- Documento con cursores -->
    <div 
      id="document-editor"
      phx-hook="DocumentEditor"
      class="relative bg-white p-8 rounded-lg shadow min-h-screen"
    >
      <%= @document.content %>

      <!-- Cursores de otros usuarios -->
      <%= for viewer <- @viewers do %>
        <%= if viewer.cursor && viewer.user_id != @current_user.id do %>
          <div 
            class="absolute w-0.5 h-5 animate-pulse"
            style={"left: #{viewer.cursor.x}px; top: #{viewer.cursor.y}px; background-color: #{viewer.color}"}
          >
            <span 
              class="absolute top-0 left-2 text-xs whitespace-nowrap px-1 rounded"
              style={"color: white; background-color: #{viewer.color}"}
            >
              <%= viewer.username %>
            </span>
          </div>
        <% end %>
      <% end %>
    </div>
  </div>
</Layouts.app>
```

### JS Hook para Cursores

```javascript
// assets/js/app.js
Hooks.DocumentEditor = {
  mounted() {
    this.el.addEventListener('mousemove', (e) => {
      const rect = this.el.getBoundingClientRect()
      this.pushEvent('cursor_move', {
        x: e.clientX - rect.left,
        y: e.clientY - rect.top
      })
    })

    this.el.addEventListener('mouseup', () => {
      const selection = window.getSelection()
      if (selection.rangeCount > 0) {
        const range = selection.getRangeAt(0)
        this.pushEvent('text_select', {
          start: range.startOffset,
          end: range.endOffset
        })
      }
    })
  }
}
```

---

## 4. Juego Multijugador {#juego-multijugador}

### LiveView de Juego

```elixir
# lib/my_app_web/live/game_live.ex
defmodule MyAppWeb.GameLive do
  use MyAppWeb, :live_view
  alias MyAppWeb.Presence

  def mount(%{"game_id" => game_id}, _session, socket) do
    game = Games.get_game!(game_id)
    topic = "game:#{game_id}"
    current_user = socket.assigns.current_user

    if connected?(socket) do
      Phoenix.PubSub.subscribe(MyApp.PubSub, topic)
      
      {:ok, _} = Presence.track(self(), topic, "#{current_user.id}", %{
        user_id: current_user.id,
        username: current_user.name,
        ready: false,
        score: 0,
        position: %{x: 0, y: 0}
      })
    end

    presences = Presence.list(topic)

    {:ok,
     socket
     |> assign(:game, game)
     |> assign(:topic, topic)
     |> assign(:presences, presences)
     |> assign(:players, list_players(presences))}
  end

  # Marcar como listo
  def handle_event("toggle_ready", _params, socket) do
    user_id = socket.assigns.current_user.id

    Presence.update(self(), socket.assigns.topic, "#{user_id}", fn meta ->
      Map.update!(meta, :ready, &(!&1))
    end)

    {:noreply, socket}
  end

  # Mover jugador
  def handle_event("move", %{"x" => x, "y" => y}, socket) do
    user_id = socket.assigns.current_user.id

    Presence.update(self(), socket.assigns.topic, "#{user_id}", fn meta ->
      Map.put(meta, :position, %{x: x, y: y})
    end)

    {:noreply, socket}
  end

  def handle_info(%{event: "presence_diff"}, socket) do
    presences = Presence.list(socket.assigns.topic)
    players = list_players(presences)

    # Iniciar juego si todos están listos
    socket = 
      if all_ready?(players) && length(players) >= 2 do
        push_event(socket, "start_game", %{})
      else
        socket
      end

    {:noreply,
     socket
     |> assign(:presences, presences)
     |> assign(:players, players)}
  end

  defp list_players(presences) do
    presences
    |> Enum.map(fn {_id, %{metas: metas}} -> List.first(metas) end)
  end

  defp all_ready?(players) do
    Enum.all?(players, & &1.ready)
  end
end
```

### Template de Juego

```heex
<Layouts.app flash={@flash}>
  <div class="max-w-4xl mx-auto p-6">
    <h1 class="text-3xl font-bold mb-6"><%= @game.name %></h1>

    <!-- Jugadores -->
    <div class="mb-6 grid grid-cols-2 gap-4">
      <%= for player <- @players do %>
        <div class={[
          "p-4 rounded-lg border",
          player.ready && "border-green-500 bg-green-50"
        ]}>
          <div class="flex items-center justify-between">
            <div>
              <p class="font-semibold"><%= player.username %></p>
              <p class="text-sm text-gray-600">Score: <%= player.score %></p>
            </div>
            <%= if player.ready do %>
              <span class="text-green-600 font-semibold">✓ Listo</span>
            <% else %>
              <span class="text-gray-400">Esperando...</span>
            <% end %>
          </div>
        </div>
      <% end %>
    </div>

    <!-- Botón Ready -->
    <button
      phx-click="toggle_ready"
      class="w-full px-6 py-3 bg-blue-600 text-white rounded-lg hover:bg-blue-700"
    >
      <%= if my_ready?(@players, @current_user.id) do %>
        Cancelar
      <% else %>
        Estoy Listo
      <% end %>
    </button>

    <!-- Canvas del juego -->
    <div 
      id="game-canvas"
      phx-hook="GameCanvas"
      class="mt-6 bg-gray-900 rounded-lg"
      style="width: 800px; height: 600px;"
    >
      <!-- Los jugadores se renderizan aquí -->
      <%= for player <- @players do %>
        <div 
          class="absolute w-8 h-8 rounded-full bg-blue-500"
          style={"left: #{player.position.x}px; top: #{player.position.y}px"}
        >
          <span class="text-xs text-white"><%= String.first(player.username) %></span>
        </div>
      <% end %>
    </div>
  </div>
</Layouts.app>
```

---

## Helpers

```elixir
defp my_ready?(players, current_user_id) do
  player = Enum.find(players, &(&1.user_id == current_user_id))
  player && player.ready
end
```

---

## Recursos

- Anterior: [Tracking de Usuarios](02_tracking_usuarios_activos.md)
- [Phoenix Presence Docs](https://hexdocs.pm/phoenix/Phoenix.Presence.html)
