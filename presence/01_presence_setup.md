# Configuración de Phoenix Presence

Guía para configurar Phoenix.Presence para tracking de usuarios en tiempo real.

## Instalación

```bash
mix phx.gen.presence
```

## Configuración

```elixir
# lib/my_app/application.ex
children = [
  MyApp.Repo,
  {Phoenix.PubSub, name: MyApp.PubSub},
  MyAppWeb.Presence,  # Agregar aquí
  MyAppWeb.Endpoint
]
```

## Uso Básico

```elixir
defmodule MyAppWeb.ChatLive do
  use MyAppWeb, :live_view
  alias MyAppWeb.Presence

  def mount(_params, _session, socket) do
    if connected?(socket) do
      Phoenix.PubSub.subscribe(MyApp.PubSub, "room:lobby")
      
      Presence.track(self(), "room:lobby", socket.assigns.current_user.id, %{
        username: socket.assigns.current_user.name,
        online_at: System.system_time(:second)
      })
    end

    presences = Presence.list("room:lobby")
    {:ok, assign(socket, :presences, presences)}
  end

  def handle_info(%{event: "presence_diff", payload: diff}, socket) do
    presences = Presence.sync_diff(socket.assigns.presences, diff)
    {:noreply, assign(socket, :presences, presences)}
  end
end
```

## Mejores Prácticas

✅ Track solo cuando `connected?(socket)`
✅ Usar `sync_diff` para actualizaciones
✅ Mantener metadata mínima
❌ No trackear en mount sin verificar conexión
