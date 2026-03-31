# PubSub Multi-Nodo en Phoenix

Guía completa para configurar y usar Phoenix.PubSub en arquitecturas distribuidas con múltiples nodos.

## 📋 Tabla de Contenidos

1. [Conceptos Básicos](#conceptos-básicos)
2. [Configuración de Adaptadores](#configuración-de-adaptadores)
3. [Clustering de Nodos](#clustering-de-nodos)
4. [Patrones de Uso](#patrones-de-uso)
5. [Monitoreo y Debugging](#monitoreo-y-debugging)
6. [Testing Multi-Nodo](#testing-multi-nodo)
7. [Mejores Prácticas](#mejores-prácticas)

---

## Conceptos Básicos

### ¿Qué es PubSub?

PubSub (Publish-Subscribe) es un patrón de mensajería donde:
- **Publishers** envían mensajes a canales (topics)
- **Subscribers** reciben mensajes de los canales a los que están suscritos
- Los mensajes se entregan a **todos los suscriptores** del canal

### PubSub en Phoenix

Phoenix.PubSub proporciona:
- Comunicación entre procesos en el **mismo nodo**
- Comunicación entre procesos en **múltiples nodos** (clustering)
- Soporte para múltiples adaptadores (PG2, Redis)
- Integración nativa con LiveView y Channels

---

## Configuración de Adaptadores

### Opción 1: PG2 (Default, sin dependencias externas)

**Ventajas:**
- Sin dependencias externas
- Simple configuración
- Ideal para desarrollo y deployments pequeños

**Limitaciones:**
- Requiere conexión directa entre nodos (mesh networking)
- No persiste mensajes

```elixir
# config/config.exs
config :my_app, MyApp.PubSub,
  name: MyApp.PubSub,
  adapter: Phoenix.PubSub.PG2

# lib/my_app/application.ex
def start(_type, _args) do
  children = [
    {Phoenix.PubSub, name: MyApp.PubSub},
    MyAppWeb.Endpoint
  ]

  opts = [strategy: :one_for_one, name: MyApp.Supervisor]
  Supervisor.start_link(children, opts)
end
```

### Opción 2: Redis (Para producción escalable)

**Ventajas:**
- No requiere mesh networking entre nodos
- Escala mejor con muchos nodos
- Puede persistir mensajes (opcional)

**Instalación:**

```elixir
# mix.exs
defp deps do
  [
    {:phoenix_pubsub_redis, "~> 3.0"}
  ]
end
```

**Configuración:**

```elixir
# config/config.exs
config :my_app, MyApp.PubSub,
  name: MyApp.PubSub,
  adapter: Phoenix.PubSub.Redis,
  host: "localhost",
  port: 6379,
  pool_size: 5

# config/runtime.exs (para producción)
if config_env() == :prod do
  redis_url = System.get_env("REDIS_URL") || 
              raise "REDIS_URL no configurado"
  
  config :my_app, MyApp.PubSub,
    url: redis_url,
    pool_size: 10
end
```

---

## Clustering de Nodos

### Configuración Básica de Nodos

```elixir
# config/runtime.exs
if config_env() == :prod do
  # Nombre único del nodo
  node_name = System.get_env("NODE_NAME") || "app"
  
  # IP del pod/container
  node_host = System.get_env("POD_IP") || "127.0.0.1"
  
  # Iniciar distributed Erlang
  :net_kernel.start([:"#{node_name}@#{node_host}"])
  
  # Cookie compartida entre todos los nodos
  Node.set_cookie(:my_app_secret_cookie)
end
```

### Auto-Discovery con libcluster

Para auto-conectar nodos en Kubernetes, Fly.io, etc:

```elixir
# mix.exs
defp deps do
  [
    {:libcluster, "~> 3.3"}
  ]
end
```

**Configuración para Kubernetes:**

```elixir
# config/runtime.exs
if config_env() == :prod do
  topologies = [
    k8s: [
      strategy: Cluster.Strategy.Kubernetes,
      config: [
        mode: :dns,
        kubernetes_node_basename: "my_app",
        kubernetes_selector: "app=my_app",
        kubernetes_namespace: "default",
        polling_interval: 10_000
      ]
    ]
  ]
  
  config :libcluster,
    topologies: topologies
end

# lib/my_app/application.ex
def start(_type, _args) do
  children = [
    {Cluster.Supervisor, [Application.get_env(:libcluster, :topologies), [name: MyApp.ClusterSupervisor]]},
    {Phoenix.PubSub, name: MyApp.PubSub},
    MyAppWeb.Endpoint
  ]

  opts = [strategy: :one_for_one, name: MyApp.Supervisor]
  Supervisor.start_link(children, opts)
end
```

**Configuración para Fly.io:**

```elixir
# config/runtime.exs
topologies = [
  fly6pn: [
    strategy: Cluster.Strategy.DNSPoll,
    config: [
      polling_interval: 5_000,
      query: "#{app_name}.internal",
      node_basename: app_name
    ]
  ]
]
```

---

## Patrones de Uso

### 1. Broadcast Simple

```elixir
# Enviar mensaje a todos los suscriptores
Phoenix.PubSub.broadcast(
  MyApp.PubSub,
  "notifications:user:#{user_id}",
  {:new_notification, notification}
)

# En LiveView
def mount(_params, _session, socket) do
  if connected?(socket) do
    user_id = socket.assigns.current_user.id
    Phoenix.PubSub.subscribe(MyApp.PubSub, "notifications:user:#{user_id}")
  end
  
  {:ok, socket}
end

def handle_info({:new_notification, notification}, socket) do
  {:noreply, 
   socket
   |> put_flash(:info, "Nueva notificación: #{notification.title}")
   |> stream_insert(:notifications, notification, at: 0)}
end
```

### 2. Broadcast desde el Nodo Local

Para mensajes que solo deben procesarse localmente:

```elixir
# Solo envía a suscriptores en el mismo nodo
Phoenix.PubSub.local_broadcast(
  MyApp.PubSub,
  "cache:invalidate",
  {:invalidate, key}
)
```

### 3. Direct Broadcast (sin pasar por el proceso actual)

```elixir
# Útil cuando estás dentro de un proceso suscrito
# y no quieres recibir tu propio mensaje
Phoenix.PubSub.broadcast_from(
  MyApp.PubSub,
  self(),
  "chat:room1",
  {:new_message, msg}
)
```

### 4. Patrón de Canal con Módulo Wrapper

```elixir
# lib/my_app/channels/chat_channel.ex
defmodule MyApp.Channels.ChatChannel do
  @pubsub MyApp.PubSub

  def broadcast_message(room_id, message) do
    Phoenix.PubSub.broadcast(
      @pubsub,
      topic(room_id),
      {:new_message, message}
    )
  end

  def broadcast_typing(room_id, user_id) do
    Phoenix.PubSub.broadcast(
      @pubsub,
      topic(room_id),
      {:user_typing, user_id}
    )
  end

  def subscribe(room_id) do
    Phoenix.PubSub.subscribe(@pubsub, topic(room_id))
  end

  def unsubscribe(room_id) do
    Phoenix.PubSub.unsubscribe(@pubsub, topic(room_id))
  end

  defp topic(room_id), do: "chat:room:#{room_id}"
end

# Uso en LiveView
def mount(%{"room_id" => room_id}, _session, socket) do
  if connected?(socket) do
    ChatChannel.subscribe(room_id)
  end
  
  {:ok, assign(socket, :room_id, room_id)}
end

def handle_info({:new_message, message}, socket) do
  {:noreply, stream_insert(socket, :messages, message)}
end
```

### 5. Patrón de Evento con Payload Estructurado

```elixir
# Definir structs para eventos
defmodule MyApp.Events do
  defmodule MessageCreated do
    defstruct [:id, :room_id, :user_id, :content, :inserted_at]
  end

  defmodule UserJoined do
    defstruct [:room_id, :user_id, :username, :joined_at]
  end
end

# Broadcast con structs
alias MyApp.Events.MessageCreated

Phoenix.PubSub.broadcast(
  MyApp.PubSub,
  "chat:room:#{room_id}",
  %MessageCreated{
    id: msg.id,
    room_id: room_id,
    user_id: user.id,
    content: msg.content,
    inserted_at: DateTime.utc_now()
  }
)

# Handle con pattern matching
def handle_info(%MessageCreated{} = event, socket) do
  message = %{
    id: event.id,
    content: event.content,
    user_id: event.user_id
  }
  
  {:noreply, stream_insert(socket, :messages, message)}
end
```

---

## Monitoreo y Debugging

### Ver Suscripciones Activas

```elixir
# En iex
Phoenix.PubSub.subscribers(MyApp.PubSub, "chat:room:1")
# => [#PID<0.234.0>, #PID<0.567.0>]

# Contar suscriptores
Phoenix.PubSub.subscribers(MyApp.PubSub, "notifications:global")
|> length()
# => 42
```

### Verificar Nodos Conectados

```elixir
# Lista de nodos conectados
Node.list()
# => [:"app@192.168.1.2", :"app@192.168.1.3"]

# Verificar cookie
Node.get_cookie()
# => :my_app_secret_cookie

# Ping a otro nodo
Node.ping(:"app@192.168.1.2")
# => :pong (si conectado) o :pang (si no)
```

### Telemetry Events

```elixir
# lib/my_app/application.ex
def start(_type, _args) do
  :telemetry.attach(
    "pubsub-broadcast",
    [:phoenix, :channel, :broadcast, :stop],
    &handle_broadcast/4,
    nil
  )
  
  # ...
end

defp handle_broadcast(_event_name, measurements, metadata, _config) do
  # Log o métricas
  Logger.debug("""
  PubSub Broadcast:
    Topic: #{metadata.topic}
    Event: #{inspect(metadata.event)}
    Duration: #{measurements.duration}
  """)
end
```

### Logger para Debugging

```elixir
# lib/my_app/pubsub_logger.ex
defmodule MyApp.PubSubLogger do
  require Logger

  def broadcast(pubsub, topic, message) do
    Logger.debug("Broadcasting to #{topic}: #{inspect(message)}")
    Phoenix.PubSub.broadcast(pubsub, topic, message)
  end

  def subscribe(pubsub, topic) do
    result = Phoenix.PubSub.subscribe(pubsub, topic)
    Logger.debug("Subscribed to #{topic} from #{inspect(self())}")
    result
  end
end
```

---

## Testing Multi-Nodo

### Testing Local (2 nodos en desarrollo)

```bash
# Terminal 1
iex --name node1@127.0.0.1 --cookie mycookie -S mix phx.server

# Terminal 2
iex --name node2@127.0.0.1 --cookie mycookie -S mix phx.server
```

```elixir
# En node1
Node.connect(:"node2@127.0.0.1")
Node.list()  # => [:"node2@127.0.0.1"]

# Suscribirse en node1
Phoenix.PubSub.subscribe(MyApp.PubSub, "test")

# Broadcast desde node2
Phoenix.PubSub.broadcast(MyApp.PubSub, "test", {:hello, "from node2"})

# Verificar mensaje en node1
flush()  # => {:hello, "from node2"}
```

### Testing en ExUnit

```elixir
# test/my_app/pubsub_test.exs
defmodule MyApp.PubSubTest do
  use ExUnit.Case
  
  @pubsub MyApp.PubSub

  setup do
    # Limpiar suscripciones
    Phoenix.PubSub.unsubscribe(@pubsub, "test:topic")
    :ok
  end

  test "broadcast entrega mensaje a suscriptores" do
    topic = "test:#{:rand.uniform(1000)}"
    Phoenix.PubSub.subscribe(@pubsub, topic)

    message = {:test_event, "data"}
    Phoenix.PubSub.broadcast(@pubsub, topic, message)

    assert_receive {:test_event, "data"}
  end

  test "broadcast_from no envía al remitente" do
    topic = "test:#{:rand.uniform(1000)}"
    Phoenix.PubSub.subscribe(@pubsub, topic)

    Phoenix.PubSub.broadcast_from(@pubsub, self(), topic, :should_not_receive)

    refute_receive :should_not_receive, 100
  end

  test "múltiples suscriptores reciben el mismo mensaje" do
    topic = "test:#{:rand.uniform(1000)}"
    
    # Crear procesos suscriptores
    parent = self()
    
    spawn(fn ->
      Phoenix.PubSub.subscribe(@pubsub, topic)
      receive do
        msg -> send(parent, {:proc1, msg})
      end
    end)
    
    spawn(fn ->
      Phoenix.PubSub.subscribe(@pubsub, topic)
      receive do
        msg -> send(parent, {:proc2, msg})
      end
    end)

    # Esperar suscripciones
    Process.sleep(10)

    # Broadcast
    Phoenix.PubSub.broadcast(@pubsub, topic, :test_message)

    # Verificar ambos recibieron
    assert_receive {:proc1, :test_message}
    assert_receive {:proc2, :test_message}
  end
end
```

### Testing con LiveView

```elixir
# test/my_app_web/live/chat_live_test.exs
defmodule MyAppWeb.ChatLiveTest do
  use MyAppWeb.ConnCase
  import Phoenix.LiveViewTest

  test "recibe mensajes broadcast", %{conn: conn} do
    {:ok, view, _html} = live(conn, "/chat/room/1")

    # Simular broadcast desde otro proceso
    Phoenix.PubSub.broadcast(
      MyApp.PubSub,
      "chat:room:1",
      {:new_message, %{id: 1, content: "Hello", user_id: 123}}
    )

    # Verificar que el LiveView renderizó el mensaje
    assert render(view) =~ "Hello"
  end
end
```

---

## Mejores Prácticas

### 1. Namespacing de Topics

```elixir
# ❌ Malo - colisiones potenciales
"user:1"
"room:1"

# ✅ Bueno - namespace claro
"notifications:user:1"
"chat:room:1"
"presence:lobby:main"
```

### 2. Wrapper Modules

Encapsula la lógica de PubSub en módulos dedicados:

```elixir
defmodule MyApp.Notifications do
  @pubsub MyApp.PubSub

  def notify_user(user_id, notification) do
    broadcast("user:#{user_id}", {:notification, notification})
  end

  def subscribe_user(user_id) do
    Phoenix.PubSub.subscribe(@pubsub, "user:#{user_id}")
  end

  defp broadcast(topic, message) do
    Phoenix.PubSub.broadcast(@pubsub, "notifications:#{topic}", message)
  end
end
```

### 3. Manejo de Errores

```elixir
def broadcast_safe(topic, message) do
  case Phoenix.PubSub.broadcast(MyApp.PubSub, topic, message) do
    :ok -> 
      :ok
    {:error, reason} -> 
      Logger.error("Broadcast failed: #{inspect(reason)}")
      {:error, reason}
  end
end
```

### 4. Limpieza de Suscripciones

```elixir
# En LiveView, auto-cleanup al desmontar
def mount(_params, _session, socket) do
  if connected?(socket) do
    Phoenix.PubSub.subscribe(MyApp.PubSub, "topic")
  end
  {:ok, socket}
end

# Phoenix automáticamente hace unsubscribe cuando el proceso muere
# Pero puedes ser explícito si necesitas:
def terminate(_reason, socket) do
  Phoenix.PubSub.unsubscribe(MyApp.PubSub, "topic")
  :ok
end
```

### 5. Rate Limiting

Para prevenir broadcast storms:

```elixir
defmodule MyApp.RateLimitedBroadcast do
  use GenServer

  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def broadcast(topic, message) do
    GenServer.cast(__MODULE__, {:broadcast, topic, message})
  end

  def init(_opts) do
    {:ok, %{last_broadcast: nil, min_interval: 100}}
  end

  def handle_cast({:broadcast, topic, message}, state) do
    now = System.monotonic_time(:millisecond)
    
    if state.last_broadcast == nil or 
       now - state.last_broadcast >= state.min_interval do
      Phoenix.PubSub.broadcast(MyApp.PubSub, topic, message)
      {:noreply, %{state | last_broadcast: now}}
    else
      {:noreply, state}
    end
  end
end
```

### 6. Monitoring de Salud

```elixir
defmodule MyApp.ClusterHealth do
  use GenServer
  require Logger

  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def init(_opts) do
    schedule_check()
    {:ok, %{}}
  end

  def handle_info(:check_health, state) do
    nodes = Node.list()
    
    if length(nodes) == 0 do
      Logger.warn("No hay otros nodos conectados al cluster")
    else
      Logger.info("Cluster health OK: #{length(nodes)} nodos conectados")
    end
    
    schedule_check()
    {:noreply, state}
  end

  defp schedule_check do
    Process.send_after(self(), :check_health, 30_000)
  end
end
```

---

## Resumen

**Key Takeaways:**

1. **PG2** para desarrollo, **Redis** para producción escalable
2. Usa **libcluster** para auto-discovery en Kubernetes/Fly.io
3. Encapsula PubSub en **wrapper modules** dedicados
4. Usa **namespacing** claro para topics
5. **Monitorea** nodos y suscripciones activamente
6. **Rate limit** broadcasts críticos
7. **Testing** multi-nodo es esencial antes de producción

**Referencias:**
- [Phoenix.PubSub Docs](https://hexdocs.pm/phoenix_pubsub/Phoenix.PubSub.html)
- [libcluster](https://hexdocs.pm/libcluster/readme.html)
- [Distributed Erlang](https://erlang.org/doc/reference_manual/distributed.html)
# PubSub Multi-Nodo

Configuración de PubSub para funcionar en múltiples nodos.

## Configuración

```elixir
# config/config.exs
config :my_app, MyApp.PubSub,
  name: MyApp.PubSub,
  adapter: Phoenix.PubSub.PG2

# lib/my_app/application.ex
children = [
  {Phoenix.PubSub, name: MyApp.PubSub},
  # ...
]
```

## Configurar Nodos

```elixir
# config/runtime.exs
if config_env() == :prod do
  node_name = System.get_env("NODE_NAME") || "app"
  node_host = System.get_env("POD_IP") || "127.0.0.1"
  
  :net_kernel.start([:"#{node_name}@#{node_host}"])
  Node.set_cookie(:my_app_cookie)
end
```

## Uso

```elixir
# Broadcast funciona automáticamente en todos los nodos
Phoenix.PubSub.broadcast(MyApp.PubSub, "chat:room1", {:new_message, msg})

# Subscribe en cualquier nodo
Phoenix.PubSub.subscribe(MyApp.PubSub, "chat:room1")
```

## Testing Multi-Nodo

```bash
# Terminal 1
iex --name node1@127.0.0.1 -S mix phx.server

# Terminal 2  
iex --name node2@127.0.0.1 -S mix phx.server

# Conectar
Node.connect(:"node1@127.0.0.1")
Node.list()  # Verificar conexión
# PubSub Multi-Nodo en Phoenix

Guía completa para configurar y usar Phoenix.PubSub en arquitecturas distribuidas con múltiples nodos.

## 📋 Tabla de Contenidos

1. [Conceptos Básicos](#conceptos-básicos)
2. [Configuración de Adaptadores](#configuración-de-adaptadores)
3. [Clustering de Nodos](#clustering-de-nodos)
4. [Patrones de Uso Avanzados](#patrones-de-uso-avanzados)
5. [Monitoreo y Debugging](#monitoreo-y-debugging)
6. [Testing Multi-Nodo](#testing-multi-nodo)
7. [Mejores Prácticas](#mejores-prácticas)
8. [Casos de Uso Reales](#casos-de-uso-reales)

---

## Conceptos Básicos

### ¿Qué es PubSub?

PubSub (Publish-Subscribe) es un patrón de mensajería donde:
- **Publishers** envían mensajes a canales (topics)
- **Subscribers** reciben mensajes de los canales a los que están suscritos
- Los mensajes se entregan a **todos los suscriptores** del canal

### PubSub en Phoenix

Phoenix.PubSub proporciona:
- Comunicación entre procesos en el **mismo nodo**
- Comunicación entre procesos en **múltiples nodos** (clustering)
- Soporte para múltiples adaptadores (PG2, Redis)
- Integración nativa con LiveView y Channels

### ¿Por qué Multi-Nodo?

**Ventajas:**
- **Alta disponibilidad**: Si un nodo cae, otros continúan
- **Escalabilidad horizontal**: Agregar más nodos según demanda
- **Balanceo de carga**: Distribuir usuarios entre nodos
- **Geo-distribución**: Nodos en diferentes regiones

---

## Configuración de Adaptadores

### Opción 1: PG2 (Default, sin dependencias externas)

**Ventajas:**
- Sin dependencias externas
- Simple configuración
- Ideal para desarrollo y deployments pequeños

**Limitaciones:**
- Requiere conexión directa entre nodos (mesh networking)
- No persiste mensajes
- No escala bien con muchos nodos (>10)

```elixir
# config/config.exs
config :my_app, MyApp.PubSub,
  name: MyApp.PubSub,
  adapter: Phoenix.PubSub.PG2

# lib/my_app/application.ex
def start(_type, _args) do
  children = [
    {Phoenix.PubSub, name: MyApp.PubSub},
    MyAppWeb.Endpoint
  ]

  opts = [strategy: :one_for_one, name: MyApp.Supervisor]
  Supervisor.start_link(children, opts)
end
```

### Opción 2: Redis (Para producción escalable)

**Ventajas:**
- No requiere mesh networking entre nodos
- Escala mejor con muchos nodos (>10)
- Puede persistir mensajes (opcional)
- Fácil de monitorear

**Instalación:**

```elixir
# mix.exs
defp deps do
  [
    {:phoenix_pubsub_redis, "~> 3.0"}
  ]
end
```

**Configuración Básica:**

```elixir
# config/config.exs
config :my_app, MyApp.PubSub,
  name: MyApp.PubSub,
  adapter: Phoenix.PubSub.Redis,
  host: "localhost",
  port: 6379,
  pool_size: 5

# config/runtime.exs (para producción)
if config_env() == :prod do
  redis_url = System.get_env("REDIS_URL") || 
              raise "REDIS_URL no configurado"
  
  config :my_app, MyApp.PubSub,
    url: redis_url,
    pool_size: 10,
    node_name: System.get_env("FLY_APP_NAME") || "my_app"
end
```

**Configuración con Sentinel (Alta Disponibilidad):**

```elixir
# config/runtime.exs
config :my_app, MyApp.PubSub,
  adapter: Phoenix.PubSub.Redis,
  sentinels: [
    [host: "sentinel1.example.com", port: 26379],
    [host: "sentinel2.example.com", port: 26379],
    [host: "sentinel3.example.com", port: 26379]
  ],
  sentinel_group: "mymaster",
  pool_size: 10
```

---

## Clustering de Nodos

### Configuración Básica de Nodos

```elixir
# config/runtime.exs
if config_env() == :prod do
  # Nombre único del nodo
  node_name = System.get_env("NODE_NAME") || "app"
  
  # IP del pod/container
  node_host = System.get_env("POD_IP") || "127.0.0.1"
  
  # Iniciar distributed Erlang
  :net_kernel.start([:"#{node_name}@#{node_host}"])
  
  # Cookie compartida entre todos los nodos (debe ser secreta)
  Node.set_cookie(:my_app_secret_cookie)
end
```

### Auto-Discovery con libcluster

Para auto-conectar nodos en Kubernetes, Fly.io, etc:

```elixir
# mix.exs
defp deps do
  [
    {:libcluster, "~> 3.3"}
  ]
end
```

**Configuración para Kubernetes:**

```elixir
# config/runtime.exs
if config_env() == :prod do
  topologies = [
    k8s: [
      strategy: Cluster.Strategy.Kubernetes,
      config: [
        mode: :dns,
        kubernetes_node_basename: "my_app",
        kubernetes_selector: "app=my_app",
        kubernetes_namespace: "default",
        polling_interval: 10_000
      ]
    ]
  ]
  
  config :libcluster,
    topologies: topologies
end

# lib/my_app/application.ex
def start(_type, _args) do
  children = [
    {Cluster.Supervisor, [Application.get_env(:libcluster, :topologies), [name: MyApp.ClusterSupervisor]]},
    {Phoenix.PubSub, name: MyApp.PubSub},
    MyAppWeb.Endpoint
  ]

  opts = [strategy: :one_for_one, name: MyApp.Supervisor]
  Supervisor.start_link(children, opts)
end
```

**Configuración para Fly.io:**

```elixir
# config/runtime.exs
if config_env() == :prod do
  app_name = System.get_env("FLY_APP_NAME") || "my_app"
  
  topologies = [
    fly6pn: [
      strategy: Cluster.Strategy.DNSPoll,
      config: [
        polling_interval: 5_000,
        query: "#{app_name}.internal",
        node_basename: app_name
      ]
    ]
  ]
  
  config :libcluster,
    topologies: topologies
end
```

**Configuración para Desarrollo Local:**

```elixir
# config/dev.exs
config :libcluster,
  topologies: [
    local: [
      strategy: Cluster.Strategy.Epmd,
      config: [hosts: [:"node1@127.0.0.1", :"node2@127.0.0.1"]]
    ]
  ]
```

---

## Patrones de Uso Avanzados

### 1. Broadcast Simple

```elixir
# Enviar mensaje a todos los suscriptores en todos los nodos
Phoenix.PubSub.broadcast(
  MyApp.PubSub,
  "notifications:user:#{user_id}",
  {:new_notification, notification}
)

# En LiveView
def mount(_params, _session, socket) do
  if connected?(socket) do
    user_id = socket.assigns.current_user.id
    Phoenix.PubSub.subscribe(MyApp.PubSub, "notifications:user:#{user_id}")
  end
  
  {:ok, socket}
end

def handle_info({:new_notification, notification}, socket) do
  {:noreply, 
   socket
   |> put_flash(:info, "Nueva notificación: #{notification.title}")
   |> stream_insert(:notifications, notification, at: 0)}
end
```

### 2. Local Broadcast (Solo Nodo Actual)

Para mensajes que solo deben procesarse localmente:

```elixir
# Solo envía a suscriptores en el mismo nodo
Phoenix.PubSub.local_broadcast(
  MyApp.PubSub,
  "cache:invalidate",
  {:invalidate, key}
)

# Ejemplo: Invalidar cache local sin afectar otros nodos
def invalidate_cache(key) do
  Phoenix.PubSub.local_broadcast(
    MyApp.PubSub,
    "cache:events",
    {:invalidate, key}
  )
end
```

### 3. Broadcast From (Excluir Remitente)

```elixir
# Útil cuando estás dentro de un proceso suscrito
# y no quieres recibir tu propio mensaje
Phoenix.PubSub.broadcast_from(
  MyApp.PubSub,
  self(),
  "chat:room:#{room_id}",
  {:new_message, message}
)

# Ejemplo: Chat donde el usuario que envió no necesita ver su mensaje broadcast
def handle_event("send_message", %{"content" => content}, socket) do
  message = create_message(socket.assigns.user_id, content)
  
  # Broadcast a todos EXCEPTO el proceso actual
  Phoenix.PubSub.broadcast_from(
    MyApp.PubSub,
    self(),
    "chat:room:#{socket.assigns.room_id}",
    {:new_message, message}
  )
  
  {:noreply, stream_insert(socket, :messages, message)}
end
```

### 4. Patrón de Canal con Módulo Wrapper

```elixir
# lib/my_app/channels/chat_channel.ex
defmodule MyApp.Channels.ChatChannel do
  @pubsub MyApp.PubSub

  def broadcast_message(room_id, message) do
    Phoenix.PubSub.broadcast(
      @pubsub,
      topic(room_id),
      {:new_message, message}
    )
  end

  def broadcast_typing(room_id, user_id) do
    Phoenix.PubSub.broadcast(
      @pubsub,
      topic(room_id),
      {:user_typing, user_id}
    )
  end

  def broadcast_user_joined(room_id, user) do
    Phoenix.PubSub.broadcast(
      @pubsub,
      topic(room_id),
      {:user_joined, user}
    )
  end

  def broadcast_user_left(room_id, user_id) do
    Phoenix.PubSub.broadcast(
      @pubsub,
      topic(room_id),
      {:user_left, user_id}
    )
  end

  def subscribe(room_id) do
    Phoenix.PubSub.subscribe(@pubsub, topic(room_id))
  end

  def unsubscribe(room_id) do
    Phoenix.PubSub.unsubscribe(@pubsub, topic(room_id))
  end

  defp topic(room_id), do: "chat:room:#{room_id}"
end

# Uso en LiveView
def mount(%{"room_id" => room_id}, _session, socket) do
  if connected?(socket) do
    ChatChannel.subscribe(room_id)
    ChatChannel.broadcast_user_joined(room_id, socket.assigns.current_user)
  end
  
  {:ok, assign(socket, :room_id, room_id)}
end

def terminate(_reason, socket) do
  ChatChannel.broadcast_user_left(socket.assigns.room_id, socket.assigns.current_user.id)
  :ok
end

def handle_info({:new_message, message}, socket) do
  {:noreply, stream_insert(socket, :messages, message)}
end

def handle_info({:user_typing, user_id}, socket) do
  {:noreply, assign(socket, :typing_user_id, user_id)}
end
```

### 5. Patrón de Evento con Payload Estructurado

```elixir
# lib/my_app/events.ex
defmodule MyApp.Events do
  defmodule MessageCreated do
    @enforce_keys [:id, :room_id, :user_id, :content]
    defstruct [:id, :room_id, :user_id, :content, :inserted_at]
  end

  defmodule UserJoined do
    @enforce_keys [:room_id, :user_id, :username]
    defstruct [:room_id, :user_id, :username, :joined_at]
  end

  defmodule UserLeft do
    @enforce_keys [:room_id, :user_id]
    defstruct [:room_id, :user_id, :left_at]
  end

  defmodule TypingStarted do
    @enforce_keys [:room_id, :user_id]
    defstruct [:room_id, :user_id, :started_at]
  end
end

# Broadcast con structs
alias MyApp.Events.MessageCreated

def create_and_broadcast_message(attrs) do
  with {:ok, message} <- Messages.create_message(attrs) do
    Phoenix.PubSub.broadcast(
      MyApp.PubSub,
      "chat:room:#{message.room_id}",
      %MessageCreated{
        id: message.id,
        room_id: message.room_id,
        user_id: message.user_id,
        content: message.content,
        inserted_at: message.inserted_at
      }
    )
    {:ok, message}
  end
end

# Handle con pattern matching
def handle_info(%MessageCreated{} = event, socket) do
  message = %{
    id: event.id,
    content: event.content,
    user_id: event.user_id,
    inserted_at: event.inserted_at
  }
  
  {:noreply, stream_insert(socket, :messages, message)}
end

def handle_info(%UserJoined{} = event, socket) do
  {:noreply, 
   socket
   |> put_flash(:info, "#{event.username} se unió a la sala")
   |> update(:online_users, &[event.user_id | &1])}
end
```

### 6. Patrón de Coordinación entre Nodos

```elixir
# lib/my_app/node_coordinator.ex
defmodule MyApp.NodeCoordinator do
  use GenServer
  require Logger

  @pubsub MyApp.PubSub

  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def request_work() do
    GenServer.call(__MODULE__, :request_work)
  end

  def init(_opts) do
    Phoenix.PubSub.subscribe(@pubsub, "cluster:coordination")
    {:ok, %{pending_work: [], claimed_work: MapSet.new()}}
  end

  def handle_call(:request_work, _from, state) do
    # Broadcast solicitud a todos los nodos
    Phoenix.PubSub.broadcast(
      @pubsub,
      "cluster:coordination",
      {:work_request, node(), self()}
    )
    
    {:reply, :ok, state}
  end

  def handle_info({:work_request, requesting_node, pid}, state) do
    if requesting_node != node() do
      # Otro nodo solicita trabajo
      work = find_available_work(state)
      
      if work do
        Phoenix.PubSub.broadcast(
          @pubsub,
          "cluster:coordination",
          {:work_offer, node(), work}
        )
      end
    end
    
    {:noreply, state}
  end

  def handle_info({:work_offer, offering_node, work}, state) do
    Logger.info("Recibida oferta de trabajo del nodo #{offering_node}")
    {:noreply, %{state | pending_work: [work | state.pending_work]}}
  end

  defp find_available_work(_state) do
    # Lógica para encontrar trabajo disponible
    nil
  end
end
```

---

## Monitoreo y Debugging

### Ver Suscripciones Activas

```elixir
# En iex
Phoenix.PubSub.subscribers(MyApp.PubSub, "chat:room:1")
# => [#PID<0.234.0>, #PID<0.567.0>]

# Contar suscriptores
Phoenix.PubSub.subscribers(MyApp.PubSub, "notifications:global")
|> length()
# => 42

# Verificar si un proceso está suscrito
Phoenix.PubSub.subscribers(MyApp.PubSub, "chat:room:1")
|> Enum.member?(self())
# => true/false
```

### Verificar Nodos Conectados

```elixir
# Lista de nodos conectados
Node.list()
# => [:"app@192.168.1.2", :"app@192.168.1.3"]

# Incluir nodo actual
[node() | Node.list()]
# => [:"app@192.168.1.1", :"app@192.168.1.2", :"app@192.168.1.3"]

# Verificar cookie
Node.get_cookie()
# => :my_app_secret_cookie

# Ping a otro nodo
Node.ping(:"app@192.168.1.2")
# => :pong (si conectado) o :pang (si no)

# Información del nodo actual
node()
# => :"app@192.168.1.1"
```

### Telemetry Events

```elixir
# lib/my_app/application.ex
def start(_type, _args) do
  :telemetry.attach(
    "pubsub-broadcast",
    [:phoenix, :channel, :broadcast, :stop],
    &handle_broadcast/4,
    nil
  )
  
  :telemetry.attach(
    "pubsub-subscribe",
    [:phoenix, :channel, :subscribe, :stop],
    &handle_subscribe/4,
    nil
  )
  
  # ...
end

defp handle_broadcast(_event_name, measurements, metadata, _config) do
  Logger.debug("""
  PubSub Broadcast:
    Topic: #{metadata.topic}
    Event: #{inspect(metadata.event)}
    Duration: #{measurements.duration}
  """)
end

defp handle_subscribe(_event_name, _measurements, metadata, _config) do
  Logger.debug("Nueva suscripción al topic: #{metadata.topic}")
end
```

### Logger para Debugging

```elixir
# lib/my_app/pubsub_logger.ex
defmodule MyApp.PubSubLogger do
  require Logger

  def broadcast(pubsub, topic, message) do
    Logger.debug("""
    Broadcasting:
      Topic: #{topic}
      Message: #{inspect(message)}
      From: #{inspect(self())}
      Node: #{node()}
    """)
    Phoenix.PubSub.broadcast(pubsub, topic, message)
  end

  def subscribe(pubsub, topic) do
    result = Phoenix.PubSub.subscribe(pubsub, topic)
    Logger.debug("""
    Subscribed:
      Topic: #{topic}
      PID: #{inspect(self())}
      Node: #{node()}
    """)
    result
  end

  def local_broadcast(pubsub, topic, message) do
    Logger.debug("""
    Local Broadcasting:
      Topic: #{topic}
      Message: #{inspect(message)}
      Node: #{node()}
    """)
    Phoenix.PubSub.local_broadcast(pubsub, topic, message)
  end
end
```

### Dashboard de Monitoreo

```elixir
# lib/my_app_web/live/cluster_dashboard_live.ex
defmodule MyAppWeb.ClusterDashboardLive do
  use MyAppWeb, :live_view

  def mount(_params, _session, socket) do
    if connected?(socket) do
      :timer.send_interval(5000, self(), :update_stats)
    end

    {:ok, assign(socket, 
      nodes: get_cluster_info(),
      topics: get_topic_stats()
    )}
  end

  def handle_info(:update_stats, socket) do
    {:noreply, assign(socket,
      nodes: get_cluster_info(),
      topics: get_topic_stats()
    )}
  end

  defp get_cluster_info do
    [node() | Node.list()]
    |> Enum.map(fn n ->
      %{
        name: n,
        alive: Node.ping(n) == :pong,
        memory: :rpc.call(n, :erlang, :memory, []),
        processes: :rpc.call(n, :erlang, :system_info, [:process_count])
      }
    end)
  end

  defp get_topic_stats do
    # Implementar según necesidades
    []
  end
end
```

---

## Testing Multi-Nodo

### Testing Local (2 nodos en desarrollo)

```bash
# Terminal 1
iex --name node1@127.0.0.1 --cookie mycookie -S mix phx.server

# Terminal 2
iex --name node2@127.0.0.1 --cookie mycookie -S mix phx.server
```

```elixir
# En node1
Node.connect(:"node2@127.0.0.1")
Node.list()  # => [:"node2@127.0.0.1"]

# Suscribirse en node1
Phoenix.PubSub.subscribe(MyApp.PubSub, "test")

# Broadcast desde node2
Phoenix.PubSub.broadcast(MyApp.PubSub, "test", {:hello, "from node2"})

# Verificar mensaje en node1
flush()  # => {:hello, "from node2"}
```

### Testing en ExUnit

```elixir
# test/my_app/pubsub_test.exs
defmodule MyApp.PubSubTest do
  use ExUnit.Case
  
  @pubsub MyApp.PubSub

  setup do
    # Limpiar suscripciones previas
    topic = "test:#{:rand.uniform(1_000_000)}"
    {:ok, topic: topic}
  end

  test "broadcast entrega mensaje a suscriptores", %{topic: topic} do
    Phoenix.PubSub.subscribe(@pubsub, topic)

    message = {:test_event, "data"}
    Phoenix.PubSub.broadcast(@pubsub, topic, message)

    assert_receive {:test_event, "data"}
  end

  test "broadcast_from no envía al remitente", %{topic: topic} do
    Phoenix.PubSub.subscribe(@pubsub, topic)

    Phoenix.PubSub.broadcast_from(@pubsub, self(), topic, :should_not_receive)

    refute_receive :should_not_receive, 100
  end

  test "local_broadcast solo en nodo actual", %{topic: topic} do
    Phoenix.PubSub.subscribe(@pubsub, topic)

    Phoenix.PubSub.local_broadcast(@pubsub, topic, :local_message)

    assert_receive :local_message
  end

  test "múltiples suscriptores reciben el mismo mensaje", %{topic: topic} do
    parent = self()
    
    # Crear procesos suscriptores
    spawn(fn ->
      Phoenix.PubSub.subscribe(@pubsub, topic)
      receive do
        msg -> send(parent, {:proc1, msg})
      end
    end)
    
    spawn(fn ->
      Phoenix.PubSub.subscribe(@pubsub, topic)
      receive do
        msg -> send(parent, {:proc2, msg})
      end
    end)

    # Esperar suscripciones
    Process.sleep(10)

    # Broadcast
    Phoenix.PubSub.broadcast(@pubsub, topic, :test_message)

    # Verificar ambos recibieron
    assert_receive {:proc1, :test_message}
    assert_receive {:proc2, :test_message}
  end

  test "unsubscribe detiene la recepción de mensajes", %{topic: topic} do
    Phoenix.PubSub.subscribe(@pubsub, topic)
    Phoenix.PubSub.unsubscribe(@pubsub, topic)

    Phoenix.PubSub.broadcast(@pubsub, topic, :should_not_receive)

    refute_receive :should_not_receive, 100
  end
end
```

### Testing con LiveView

```elixir
# test/my_app_web/live/chat_live_test.exs
defmodule MyAppWeb.ChatLiveTest do
  use MyAppWeb.ConnCase
  import Phoenix.LiveViewTest

  test "recibe mensajes broadcast", %{conn: conn} do
    {:ok, view, _html} = live(conn, "/chat/room/1")

    # Simular broadcast desde otro proceso
    Phoenix.PubSub.broadcast(
      MyApp.PubSub,
      "chat:room:1",
      {:new_message, %{id: 1, content: "Hello", user_id: 123}}
    )

    # Verificar que el LiveView renderizó el mensaje
    assert render(view) =~ "Hello"
  end

  test "desuscribe al desmontar", %{conn: conn} do
    {:ok, view, _html} = live(conn, "/chat/room/1")
    
    # Verificar suscripción activa
    subscribers = Phoenix.PubSub.subscribers(MyApp.PubSub, "chat:room:1")
    assert length(subscribers) > 0

    # Cerrar el LiveView
    close(view)

    # Esperar un momento
    Process.sleep(10)

    # Verificar que se desuscribió
    subscribers = Phoenix.PubSub.subscribers(MyApp.PubSub, "chat:room:1")
    assert length(subscribers) == 0
  end
end
```

### Testing de Integración Multi-Nodo

```elixir
# test/support/cluster_case.ex
defmodule MyApp.ClusterCase do
  use ExUnit.CaseTemplate

  using do
    quote do
      import MyApp.ClusterCase
    end
  end

  def start_nodes(nodes) do
    nodes
    |> Enum.map(&start_node/1)
    |> Enum.each(&connect_node/1)
  end

  defp start_node(name) do
    {:ok, node} = :slave.start_link(
      String.to_charlist("127.0.0.1"),
      name,
      '-setcookie mycookie'
    )
    node
  end

  defp connect_node(node) do
    true = Node.connect(node)
    :rpc.call(node, Application, :ensure_all_started, [:my_app])
  end

  def stop_nodes(nodes) do
    Enum.each(nodes, &:slave.stop/1)
  end
end

# test/my_app/cluster_pubsub_test.exs
defmodule MyApp.ClusterPubSubTest do
  use MyApp.ClusterCase
  
  @nodes [:node1, :node2]

  setup do
    start_nodes(@nodes)
    on_exit(fn -> stop_nodes(@nodes) end)
    :ok
  end

  test "broadcast entre nodos" do
    topic = "test:#{:rand.uniform(1_000_000)}"
    Phoenix.PubSub.subscribe(MyApp.PubSub, topic)

    # Broadcast desde otro nodo
    :rpc.call(
      :node1,
      Phoenix.PubSub,
      :broadcast,
      [MyApp.PubSub, topic, {:test, "from node1"}]
    )

    assert_receive {:test, "from node1"}
  end
end
```

---

## Mejores Prácticas

### 1. Namespacing de Topics

```elixir
# ❌ Malo - colisiones potenciales
"user:1"
"room:1"

# ✅ Bueno - namespace claro y jerárquico
"notifications:user:1"
"notifications:user:1:mentions"
"chat:room:1"
"chat:room:1:typing"
"presence:lobby:main"
"presence:game:123"
```

### 2. Wrapper Modules

Encapsula la lógica de PubSub en módulos dedicados:

```elixir
defmodule MyApp.Notifications.PubSub do
  @pubsub MyApp.PubSub

  def notify_user(user_id, notification) do
    broadcast_user("notifications", user_id, {:notification, notification})
  end

  def notify_mentions(user_id, mention) do
    broadcast_user("mentions", user_id, {:mention, mention})
  end

  def subscribe_user_notifications(user_id) do
    Phoenix.PubSub.subscribe(@pubsub, topic_user("notifications", user_id))
  end

  def subscribe_user_mentions(user_id) do
    Phoenix.PubSub.subscribe(@pubsub, topic_user("mentions", user_id))
  end

  defp broadcast_user(type, user_id, message) do
    Phoenix.PubSub.broadcast(@pubsub, topic_user(type, user_id), message)
  end

  defp topic_user(type, user_id) do
    "notifications:user:#{user_id}:#{type}"
  end
end
```

### 3. Manejo de Errores

```elixir
def broadcast_safe(topic, message) do
  case Phoenix.PubSub.broadcast(MyApp.PubSub, topic, message) do
    :ok -> 
      :ok
    {:error, reason} -> 
      Logger.error("Broadcast failed: #{inspect(reason)}")
      # Opcional: reintentar o reportar error
      {:error, reason}
  end
end

def broadcast_with_retry(topic, message, retries \\ 3) do
  case Phoenix.PubSub.broadcast(MyApp.PubSub, topic, message) do
    :ok ->
      :ok
    {:error, _reason} when retries > 0 ->
      Process.sleep(100)
      broadcast_with_retry(topic, message, retries - 1)
    {:error, reason} ->
      {:error, reason}
  end
end
```

### 4. Limpieza de Suscripciones

```elixir
# En LiveView, auto-cleanup al desmontar
def mount(_params, _session, socket) do
  if connected?(socket) do
    Phoenix.PubSub.subscribe(MyApp.PubSub, "topic")
  end
  {:ok, socket}
end

# Phoenix automáticamente hace unsubscribe cuando el proceso muere
# Pero puedes ser explícito si necesitas:
def terminate(_reason, socket) do
  Phoenix.PubSub.unsubscribe(MyApp.PubSub, "topic")
  :ok
end

# En GenServers
def init(state) do
  Phoenix.PubSub.subscribe(MyApp.PubSub, "events")
  {:ok, state}
end

def terminate(_reason, _state) do
  Phoenix.PubSub.unsubscribe(MyApp.PubSub, "events")
  :ok
end
```

### 5. Rate Limiting

Para prevenir broadcast storms:

```elixir
defmodule MyApp.RateLimitedBroadcast do
  use GenServer

  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def broadcast(topic, message) do
    GenServer.cast(__MODULE__, {:broadcast, topic, message})
  end

  def init(_opts) do
    {:ok, %{
      last_broadcast: %{},
      min_interval: 100,
      queue: :queue.new()
    }}
  end

  def handle_cast({:broadcast, topic, message}, state) do
    now = System.monotonic_time(:millisecond)
    last = Map.get(state.last_broadcast, topic, 0)
    
    if now - last >= state.min_interval do
      Phoenix.PubSub.broadcast(MyApp.PubSub, topic, message)
      {:noreply, %{state | last_broadcast: Map.put(state.last_broadcast, topic, now)}}
    else
      # Encolar mensaje
      queue = :queue.in({topic, message}, state.queue)
      schedule_next_broadcast(state.min_interval - (now - last))
      {:noreply, %{state | queue: queue}}
    end
  end

  def handle_info(:process_queue, state) do
    case :queue.out(state.queue) do
      {{:value, {topic, message}}, new_queue} ->
        Phoenix.PubSub.broadcast(MyApp.PubSub, topic, message)
        now = System.monotonic_time(:millisecond)
        {:noreply, %{state | 
          queue: new_queue,
          last_broadcast: Map.put(state.last_broadcast, topic, now)
        }}
      {:empty, _} ->
        {:noreply, state}
    end
  end

  defp schedule_next_broadcast(delay) do
    Process.send_after(self(), :process_queue, delay)
  end
end
```

### 6. Monitoring de Salud

```elixir
defmodule MyApp.ClusterHealth do
  use GenServer
  require Logger

  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def get_status do
    GenServer.call(__MODULE__, :get_status)
  end

  def init(_opts) do
    schedule_check()
    {:ok, %{
      last_check: nil,
      nodes_status: %{},
      alerts: []
    }}
  end

  def handle_call(:get_status, _from, state) do
    {:reply, state, state}
  end

  def handle_info(:check_health, state) do
    nodes = [node() | Node.list()]
    
    nodes_status = 
      nodes
      |> Enum.map(&check_node/1)
      |> Map.new()
    
    alerts = detect_alerts(nodes_status, state.nodes_status)
    
    if alerts != [] do
      Enum.each(alerts, &log_alert/1)
    end
    
    schedule_check()
    {:noreply, %{state | 
      last_check: DateTime.utc_now(),
      nodes_status: nodes_status,
      alerts: alerts
    }}
  end

  defp check_node(node) do
    status = %{
      alive: Node.ping(node) == :pong,
      memory: get_memory(node),
      processes: get_process_count(node)
    }
    {node, status}
  end

  defp get_memory(node) do
    case :rpc.call(node, :erlang, :memory, []) do
      {:badrpc, _} -> nil
      memory -> memory[:total]
    end
  end

  defp get_process_count(node) do
    case :rpc.call(node, :erlang, :system_info, [:process_count]) do
      {:badrpc, _} -> nil
      count -> count
    end
  end

  defp detect_alerts(current, previous) do
    current
    |> Enum.filter(fn {node, status} ->
      !status.alive or is_high_memory(status) or is_high_process_count(status)
    end)
    |> Enum.map(fn {node, status} -> build_alert(node, status) end)
  end

  defp is_high_memory(%{memory: memory}) when is_integer(memory) do
    memory > 1_000_000_000  # 1GB
  end
  defp is_high_memory(_), do: false

  defp is_high_process_count(%{processes: count}) when is_integer(count) do
    count > 100_000
  end
  defp is_high_process_count(_), do: false

  defp build_alert(node, %{alive: false}) do
    {:node_down, node}
  end
  defp build_alert(node, %{memory: memory}) when is_integer(memory) do
    {:high_memory, node, memory}
  end
  defp build_alert(node, %{processes: count}) when is_integer(count) do
    {:high_process_count, node, count}
  end

  defp log_alert({:node_down, node}) do
    Logger.warn("Nodo #{node} está caído")
  end
  defp log_alert({:high_memory, node, memory}) do
    Logger.warn("Nodo #{node} tiene alta memoria: #{div(memory, 1_000_000)}MB")
  end
  defp log_alert({:high_process_count, node, count}) do
    Logger.warn("Nodo #{node} tiene alto número de procesos: #{count}")
  end

  defp schedule_check do
    Process.send_after(self(), :check_health, 30_000)
  end
end
```

---

## Casos de Uso Reales

### 1. Sistema de Notificaciones

```elixir
defmodule MyApp.NotificationSystem do
  @pubsub MyApp.PubSub

  def notify_user(user_id, notification) do
    # Guardar en DB
    {:ok, notification} = MyApp.Notifications.create_notification(notification)
    
    # Broadcast a suscriptores
    Phoenix.PubSub.broadcast(
      @pubsub,
      "notifications:user:#{user_id}",
      {:new_notification, notification}
    )
  end

  def mark_as_read(notification_id, user_id) do
    {:ok, notification} = MyApp.Notifications.mark_as_read(notification_id)
    
    Phoenix.PubSub.broadcast(
      @pubsub,
      "notifications:user:#{user_id}",
      {:notification_read, notification_id}
    )
  end
end
```

### 2. Chat en Tiempo Real

```elixir
defmodule MyApp.Chat do
  @pubsub MyApp.PubSub

  def send_message(room_id, user_id, content) do
    {:ok, message} = MyApp.Messages.create_message(%{
      room_id: room_id,
      user_id: user_id,
      content: content
    })
    
    Phoenix.PubSub.broadcast(
      @pubsub,
      "chat:room:#{room_id}",
      {:new_message, message}
    )
  end

  def typing_started(room_id, user_id) do
    Phoenix.PubSub.broadcast(
      @pubsub,
      "chat:room:#{room_id}:typing",
      {:user_typing, user_id}
    )
  end

  def typing_stopped(room_id, user_id) do
    Phoenix.PubSub.broadcast(
      @pubsub,
      "chat:room:#{room_id}:typing",
      {:user_stopped_typing, user_id}
    )
  end
end
```

### 3. Invalidación de Cache Distribuida

```elixir
defmodule MyApp.DistributedCache do
  @pubsub MyApp.PubSub
  @cache_table :my_cache

  def get(key) do
    case :ets.lookup(@cache_table, key) do
      [{^key, value}] -> {:ok, value}
      [] -> :error
    end
  end

  def put(key, value) do
    :ets.insert(@cache_table, {key, value})
    
    # Invalidar en todos los nodos
    Phoenix.PubSub.broadcast(
      @pubsub,
      "cache:invalidate",
      {:invalidate, key}
    )
  end

  def subscribe do
    Phoenix.PubSub.subscribe(@pubsub, "cache:invalidate")
  end

  def handle_invalidation({:invalidate, key}) do
    :ets.delete(@cache_table, key)
  end
end
```

---

## Resumen

**Key Takeaways:**

1. **PG2** para desarrollo, **Redis** para producción escalable
2. Usa **libcluster** para auto-discovery en Kubernetes/Fly.io
3. Encapsula PubSub en **wrapper modules** dedicados
4. Usa **namespacing** claro y jerárquico para topics
5. **Monitorea** nodos y suscripciones activamente
6. **Rate limit** broadcasts críticos
7. **Testing** multi-nodo es esencial antes de producción
8. **Maneja errores** y reintentos apropiadamente
9. **Limpia suscripciones** en terminación de procesos
10. **Documenta** tus patrones de topics

**Referencias:**
- [Phoenix.PubSub Docs](https://hexdocs.pm/phoenix_pubsub/Phoenix.PubSub.html)
- [libcluster](https://hexdocs.pm/libcluster/readme.html)
- [Distributed Erlang](https://erlang.org/doc/reference_manual/distributed.html)
# PubSub Multi-Nodo en Phoenix

Guía completa para configurar y usar Phoenix.PubSub en arquitecturas distribuidas con múltiples nodos.

## 📋 Tabla de Contenidos

1. [Conceptos Básicos](#conceptos-básicos)
2. [Configuración de Adaptadores](#configuración-de-adaptadores)
3. [Clustering de Nodos](#clustering-de-nodos)
4. [Patrones de Uso](#patrones-de-uso)
5. [Monitoreo y Debugging](#monitoreo-y-debugging)
6. [Testing Multi-Nodo](#testing-multi-nodo)
7. [Mejores Prácticas](#mejores-prácticas)

---

## Conceptos Básicos

### ¿Qué es PubSub?

PubSub (Publish-Subscribe) es un patrón de mensajería donde:
- **Publishers** envían mensajes a canales (topics)
- **Subscribers** reciben mensajes de los canales a los que están suscritos
- Los mensajes se entregan a **todos los suscriptores** del canal

### PubSub en Phoenix

Phoenix.PubSub proporciona:
- Comunicación entre procesos en el **mismo nodo**
- Comunicación entre procesos en **múltiples nodos** (clustering)
- Soporte para múltiples adaptadores (PG2, Redis)
- Integración nativa con LiveView y Channels

---

## Configuración de Adaptadores

### Opción 1: PG2 (Default, sin dependencias externas)

**Ventajas:**
- Sin dependencias externas
- Simple configuración
- Ideal para desarrollo y deployments pequeños

**Limitaciones:**
- Requiere conexión directa entre nodos (mesh networking)
- No persiste mensajes

```elixir
# config/config.exs
config :my_app, MyApp.PubSub,
  name: MyApp.PubSub,
  adapter: Phoenix.PubSub.PG2

# lib/my_app/application.ex
def start(_type, _args) do
  children = [
    {Phoenix.PubSub, name: MyApp.PubSub},
    MyAppWeb.Endpoint
  ]

  opts = [strategy: :one_for_one, name: MyApp.Supervisor]
  Supervisor.start_link(children, opts)
end
```

### Opción 2: Redis (Para producción escalable)

**Ventajas:**
- No requiere mesh networking entre nodos
- Escala mejor con muchos nodos
- Puede persistir mensajes (opcional)

**Instalación:**

```elixir
# mix.exs
defp deps do
  [
    {:phoenix_pubsub_redis, "~> 3.0"}
  ]
end
```

**Configuración:**

```elixir
# config/config.exs
config :my_app, MyApp.PubSub,
  name: MyApp.PubSub,
  adapter: Phoenix.PubSub.Redis,
  host: "localhost",
  port: 6379,
  pool_size: 5

# config/runtime.exs (para producción)
if config_env() == :prod do
  redis_url = System.get_env("REDIS_URL") || 
              raise "REDIS_URL no configurado"
  
  config :my_app, MyApp.PubSub,
    url: redis_url,
    pool_size: 10
end
```

---

## Clustering de Nodos

### Configuración Básica de Nodos

```elixir
# config/runtime.exs
if config_env() == :prod do
  # Nombre único del nodo
  node_name = System.get_env("NODE_NAME") || "app"
  
  # IP del pod/container
  node_host = System.get_env("POD_IP") || "127.0.0.1"
  
  # Iniciar distributed Erlang
  :net_kernel.start([:"#{node_name}@#{node_host}"])
  
  # Cookie compartida entre todos los nodos
  Node.set_cookie(:my_app_secret_cookie)
end
```

### Auto-Discovery con libcluster

Para auto-conectar nodos en Kubernetes, Fly.io, etc:

```elixir
# mix.exs
defp deps do
  [
    {:libcluster, "~> 3.3"}
  ]
end
```

**Configuración para Kubernetes:**

```elixir
# config/runtime.exs
if config_env() == :prod do
  topologies = [
    k8s: [
      strategy: Cluster.Strategy.Kubernetes,
      config: [
        mode: :dns,
        kubernetes_node_basename: "my_app",
        kubernetes_selector: "app=my_app",
        kubernetes_namespace: "default",
        polling_interval: 10_000
      ]
    ]
  ]
  
  config :libcluster,
    topologies: topologies
end

# lib/my_app/application.ex
def start(_type, _args) do
  children = [
    {Cluster.Supervisor, [Application.get_env(:libcluster, :topologies), [name: MyApp.ClusterSupervisor]]},
    {Phoenix.PubSub, name: MyApp.PubSub},
    MyAppWeb.Endpoint
  ]

  opts = [strategy: :one_for_one, name: MyApp.Supervisor]
  Supervisor.start_link(children, opts)
end
```

**Configuración para Fly.io:**

```elixir
# config/runtime.exs
topologies = [
  fly6pn: [
    strategy: Cluster.Strategy.DNSPoll,
    config: [
      polling_interval: 5_000,
      query: "#{app_name}.internal",
      node_basename: app_name
    ]
  ]
]
```

---

## Patrones de Uso

### 1. Broadcast Simple

```elixir
# Enviar mensaje a todos los suscriptores
Phoenix.PubSub.broadcast(
  MyApp.PubSub,
  "notifications:user:#{user_id}",
  {:new_notification, notification}
)

# En LiveView
def mount(_params, _session, socket) do
  if connected?(socket) do
    user_id = socket.assigns.current_user.id
    Phoenix.PubSub.subscribe(MyApp.PubSub, "notifications:user:#{user_id}")
  end
  
  {:ok, socket}
end

def handle_info({:new_notification, notification}, socket) do
  {:noreply, 
   socket
   |> put_flash(:info, "Nueva notificación: #{notification.title}")
   |> stream_insert(:notifications, notification, at: 0)}
end
```

### 2. Broadcast desde el Nodo Local

Para mensajes que solo deben procesarse localmente:

```elixir
# Solo envía a suscriptores en el mismo nodo
Phoenix.PubSub.local_broadcast(
  MyApp.PubSub,
  "cache:invalidate",
  {:invalidate, key}
)
```

### 3. Direct Broadcast (sin pasar por el proceso actual)

```elixir
# Útil cuando estás dentro de un proceso suscrito
# y no quieres recibir tu propio mensaje
Phoenix.PubSub.broadcast_from(
  MyApp.PubSub,
  self(),
  "chat:room1",
  {:new_message, msg}
)
```

### 4. Patrón de Canal con Módulo Wrapper

```elixir
# lib/my_app/channels/chat_channel.ex
defmodule MyApp.Channels.ChatChannel do
  @pubsub MyApp.PubSub

  def broadcast_message(room_id, message) do
    Phoenix.PubSub.broadcast(
      @pubsub,
      topic(room_id),
      {:new_message, message}
    )
  end

  def broadcast_typing(room_id, user_id) do
    Phoenix.PubSub.broadcast(
      @pubsub,
      topic(room_id),
      {:user_typing, user_id}
    )
  end

  def subscribe(room_id) do
    Phoenix.PubSub.subscribe(@pubsub, topic(room_id))
  end

  def unsubscribe(room_id) do
    Phoenix.PubSub.unsubscribe(@pubsub, topic(room_id))
  end

  defp topic(room_id), do: "chat:room:#{room_id}"
end

# Uso en LiveView
def mount(%{"room_id" => room_id}, _session, socket) do
  if connected?(socket) do
    ChatChannel.subscribe(room_id)
  end
  
  {:ok, assign(socket, :room_id, room_id)}
end

def handle_info({:new_message, message}, socket) do
  {:noreply, stream_insert(socket, :messages, message)}
end
```

### 5. Patrón de Evento con Payload Estructurado

```elixir
# Definir structs para eventos
defmodule MyApp.Events do
  defmodule MessageCreated do
    defstruct [:id, :room_id, :user_id, :content, :inserted_at]
  end

  defmodule UserJoined do
    defstruct [:room_id, :user_id, :username, :joined_at]
  end
end

# Broadcast con structs
alias MyApp.Events.MessageCreated

Phoenix.PubSub.broadcast(
  MyApp.PubSub,
  "chat:room:#{room_id}",
  %MessageCreated{
    id: msg.id,
    room_id: room_id,
    user_id: user.id,
    content: msg.content,
    inserted_at: DateTime.utc_now()
  }
)

# Handle con pattern matching
def handle_info(%MessageCreated{} = event, socket) do
  message = %{
    id: event.id,
    content: event.content,
    user_id: event.user_id
  }
  
  {:noreply, stream_insert(socket, :messages, message)}
end
```

---

## Monitoreo y Debugging

### Ver Suscripciones Activas

```elixir
# En iex
Phoenix.PubSub.subscribers(MyApp.PubSub, "chat:room:1")
# => [#PID<0.234.0>, #PID<0.567.0>]

# Contar suscriptores
Phoenix.PubSub.subscribers(MyApp.PubSub, "notifications:global")
|> length()
# => 42
```

### Verificar Nodos Conectados

```elixir
# Lista de nodos conectados
Node.list()
# => [:"app@192.168.1.2", :"app@192.168.1.3"]

# Verificar cookie
Node.get_cookie()
# => :my_app_secret_cookie

# Ping a otro nodo
Node.ping(:"app@192.168.1.2")
# => :pong (si conectado) o :pang (si no)
```

### Telemetry Events

```elixir
# lib/my_app/application.ex
def start(_type, _args) do
  :telemetry.attach(
    "pubsub-broadcast",
    [:phoenix, :channel, :broadcast, :stop],
    &handle_broadcast/4,
    nil
  )
  
  # ...
end

defp handle_broadcast(_event_name, measurements, metadata, _config) do
  # Log o métricas
  Logger.debug("""
  PubSub Broadcast:
    Topic: #{metadata.topic}
    Event: #{inspect(metadata.event)}
    Duration: #{measurements.duration}
  """)
end
```

### Logger para Debugging

```elixir
# lib/my_app/pubsub_logger.ex
defmodule MyApp.PubSubLogger do
  require Logger

  def broadcast(pubsub, topic, message) do
    Logger.debug("Broadcasting to #{topic}: #{inspect(message)}")
    Phoenix.PubSub.broadcast(pubsub, topic, message)
  end

  def subscribe(pubsub, topic) do
    result = Phoenix.PubSub.subscribe(pubsub, topic)
    Logger.debug("Subscribed to #{topic} from #{inspect(self())}")
    result
  end
end
```

---

## Testing Multi-Nodo

### Testing Local (2 nodos en desarrollo)

```bash
# Terminal 1
iex --name node1@127.0.0.1 --cookie mycookie -S mix phx.server

# Terminal 2
iex --name node2@127.0.0.1 --cookie mycookie -S mix phx.server
```

```elixir
# En node1
Node.connect(:"node2@127.0.0.1")
Node.list()  # => [:"node2@127.0.0.1"]

# Suscribirse en node1
Phoenix.PubSub.subscribe(MyApp.PubSub, "test")

# Broadcast desde node2
Phoenix.PubSub.broadcast(MyApp.PubSub, "test", {:hello, "from node2"})

# Verificar mensaje en node1
flush()  # => {:hello, "from node2"}
```

### Testing en ExUnit

```elixir
# test/my_app/pubsub_test.exs
defmodule MyApp.PubSubTest do
  use ExUnit.Case
  
  @pubsub MyApp.PubSub

  setup do
    # Limpiar suscripciones
    Phoenix.PubSub.unsubscribe(@pubsub, "test:topic")
    :ok
  end

  test "broadcast entrega mensaje a suscriptores" do
    topic = "test:#{:rand.uniform(1000)}"
    Phoenix.PubSub.subscribe(@pubsub, topic)

    message = {:test_event, "data"}
    Phoenix.PubSub.broadcast(@pubsub, topic, message)

    assert_receive {:test_event, "data"}
  end

  test "broadcast_from no envía al remitente" do
    topic = "test:#{:rand.uniform(1000)}"
    Phoenix.PubSub.subscribe(@pubsub, topic)

    Phoenix.PubSub.broadcast_from(@pubsub, self(), topic, :should_not_receive)

    refute_receive :should_not_receive, 100
  end

  test "múltiples suscriptores reciben el mismo mensaje" do
    topic = "test:#{:rand.uniform(1000)}"
    
    # Crear procesos suscriptores
    parent = self()
    
    spawn(fn ->
      Phoenix.PubSub.subscribe(@pubsub, topic)
      receive do
        msg -> send(parent, {:proc1, msg})
      end
    end)
    
    spawn(fn ->
      Phoenix.PubSub.subscribe(@pubsub, topic)
      receive do
        msg -> send(parent, {:proc2, msg})
      end
    end)

    # Esperar suscripciones
    Process.sleep(10)

    # Broadcast
    Phoenix.PubSub.broadcast(@pubsub, topic, :test_message)

    # Verificar ambos recibieron
    assert_receive {:proc1, :test_message}
    assert_receive {:proc2, :test_message}
  end
end
```

### Testing con LiveView

```elixir
# test/my_app_web/live/chat_live_test.exs
defmodule MyAppWeb.ChatLiveTest do
  use MyAppWeb.ConnCase
  import Phoenix.LiveViewTest

  test "recibe mensajes broadcast", %{conn: conn} do
    {:ok, view, _html} = live(conn, "/chat/room/1")

    # Simular broadcast desde otro proceso
    Phoenix.PubSub.broadcast(
      MyApp.PubSub,
      "chat:room:1",
      {:new_message, %{id: 1, content: "Hello", user_id: 123}}
    )

    # Verificar que el LiveView renderizó el mensaje
    assert render(view) =~ "Hello"
  end
end
```

---

## Mejores Prácticas

### 1. Namespacing de Topics

```elixir
# ❌ Malo - colisiones potenciales
"user:1"
"room:1"

# ✅ Bueno - namespace claro
"notifications:user:1"
"chat:room:1"
"presence:lobby:main"
```

### 2. Wrapper Modules

Encapsula la lógica de PubSub en módulos dedicados:

```elixir
defmodule MyApp.Notifications do
  @pubsub MyApp.PubSub

  def notify_user(user_id, notification) do
    broadcast("user:#{user_id}", {:notification, notification})
  end

  def subscribe_user(user_id) do
    Phoenix.PubSub.subscribe(@pubsub, "user:#{user_id}")
  end

  defp broadcast(topic, message) do
    Phoenix.PubSub.broadcast(@pubsub, "notifications:#{topic}", message)
  end
end
```

### 3. Manejo de Errores

```elixir
def broadcast_safe(topic, message) do
  case Phoenix.PubSub.broadcast(MyApp.PubSub, topic, message) do
    :ok -> 
      :ok
    {:error, reason} -> 
      Logger.error("Broadcast failed: #{inspect(reason)}")
      {:error, reason}
  end
end
```

### 4. Limpieza de Suscripciones

```elixir
# En LiveView, auto-cleanup al desmontar
def mount(_params, _session, socket) do
  if connected?(socket) do
    Phoenix.PubSub.subscribe(MyApp.PubSub, "topic")
  end
  {:ok, socket}
end

# Phoenix automáticamente hace unsubscribe cuando el proceso muere
# Pero puedes ser explícito si necesitas:
def terminate(_reason, socket) do
  Phoenix.PubSub.unsubscribe(MyApp.PubSub, "topic")
  :ok
end
```

### 5. Rate Limiting

Para prevenir broadcast storms:

```elixir
defmodule MyApp.RateLimitedBroadcast do
  use GenServer

  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def broadcast(topic, message) do
    GenServer.cast(__MODULE__, {:broadcast, topic, message})
  end

  def init(_opts) do
    {:ok, %{last_broadcast: nil, min_interval: 100}}
  end

  def handle_cast({:broadcast, topic, message}, state) do
    now = System.monotonic_time(:millisecond)
    
    if state.last_broadcast == nil or 
       now - state.last_broadcast >= state.min_interval do
      Phoenix.PubSub.broadcast(MyApp.PubSub, topic, message)
      {:noreply, %{state | last_broadcast: now}}
    else
      {:noreply, state}
    end
  end
end
```

### 6. Monitoring de Salud

```elixir
defmodule MyApp.ClusterHealth do
  use GenServer
  require Logger

  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def init(_opts) do
    schedule_check()
    {:ok, %{}}
  end

  def handle_info(:check_health, state) do
    nodes = Node.list()
    
    if length(nodes) == 0 do
      Logger.warn("No hay otros nodos conectados al cluster")
    else
      Logger.info("Cluster health OK: #{length(nodes)} nodos conectados")
    end
    
    schedule_check()
    {:noreply, state}
  end

  defp schedule_check do
    Process.send_after(self(), :check_health, 30_000)
  end
end
```

---

## Resumen

**Key Takeaways:**

1. **PG2** para desarrollo, **Redis** para producción escalable
2. Usa **libcluster** para auto-discovery en Kubernetes/Fly.io
3. Encapsula PubSub en **wrapper modules** dedicados
4. Usa **namespacing** claro para topics
5. **Monitorea** nodos y suscripciones activamente
6. **Rate limit** broadcasts críticos
7. **Testing** multi-nodo es esencial antes de producción

**Referencias:**
- [Phoenix.PubSub Docs](https://hexdocs.pm/phoenix_pubsub/Phoenix.PubSub.html)
- [libcluster](https://hexdocs.pm/libcluster/readme.html)
- [Distributed Erlang](https://erlang.org/doc/reference_manual/distributed.html)
