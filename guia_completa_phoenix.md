# Guía Completa de Trabajo con Phoenix Framework

Esta guía cubre las mejores prácticas para trabajar con Phoenix, incluyendo validaciones, testing, LiveView con PubSub multi-nodo, y consumo de servicios RPC.

---

## Tabla de Contenidos

1. [Validaciones en HTML y LiveView](#validaciones)
2. [Testing con Datos Relacionados (FK)](#testing-datos-relacionados)
3. [Testing de HTML Estático vs LiveView](#testing-html)
4. [LiveView con PubSub en Múltiples Nodos](#liveview-pubsub-multinodo)
5. [Consumo de Servicios RPC/EPMD y Refresco de Vistas](#rpc-refresco-vistas)
6. [Patrones de Refresco de Vistas HTML](#patrones-refresco)

---

## 1. Validaciones en HTML y LiveView {#validaciones}

### 1.1 Validaciones del Lado del Servidor con Ecto.Changeset

**Regla fundamental**: TODAS las validaciones DEBEN ocurrir en el servidor mediante `Ecto.Changeset`.

```elixir
defmodule MyApp.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset

  schema "users" do
    field :email, :string
    field :name, :string
    field :age, :integer
    timestamps()
  end

  def changeset(user, attrs) do
    user
    |> cast(attrs, [:email, :name, :age])
    |> validate_required([:email, :name])
    |> validate_format(:email, ~r/@/)
    |> validate_length(:name, min: 2, max: 100)
    |> validate_number(:age, greater_than: 0, less_than: 150)
    |> unique_constraint(:email)
  end
end
```

### 1.2 Validaciones en Tiempo Real con LiveView

**Patrón correcto**: Usar `phx-change` para validaciones mientras el usuario escribe.

```elixir
defmodule MyAppWeb.UserLive.Form do
  use MyAppWeb, :live_view

  def mount(_params, _session, socket) do
    changeset = Accounts.change_user(%User{})
    
    {:ok,
     socket
     |> assign(:form, to_form(changeset))
     |> assign(:user, nil)}
  end

  # Validación en tiempo real mientras el usuario escribe
  def handle_event("validate", %{"user" => user_params}, socket) do
    changeset =
      %User{}
      |> Accounts.change_user(user_params)
      |> Map.put(:action, :validate)

    {:noreply, assign(socket, form: to_form(changeset))}
  end

  # Guardado final con validación completa
  def handle_event("save", %{"user" => user_params}, socket) do
    case Accounts.create_user(user_params) do
      {:ok, user} ->
        {:noreply,
         socket
         |> put_flash(:info, "Usuario creado exitosamente")
         |> push_navigate(to: ~p"/users/#{user}")}

      {:error, %Ecto.Changeset{} = changeset} ->
        {:noreply, assign(socket, form: to_form(changeset))}
    end
  end
end
```

### 1.3 Template HTML para Validaciones

**Reglas importantes**:
- SIEMPRE usar `to_form/2` en el LiveView
- SIEMPRE usar `<.form>` y `<.input>` en la template
- NUNCA acceder al changeset directamente en la template
- SIEMPRE dar un ID único al form

```heex
<Layouts.app flash={@flash}>
  <div class="mx-auto max-w-2xl">
    <h1 class="text-3xl font-bold mb-8">Nuevo Usuario</h1>

    <.form 
      for={@form} 
      id="user-form" 
      phx-change="validate" 
      phx-submit="save"
      class="space-y-6"
    >
      <.input 
        field={@form[:name]} 
        type="text" 
        label="Nombre completo"
        placeholder="Juan Pérez"
        required
      />

      <.input 
        field={@form[:email]} 
        type="email" 
        label="Email"
        placeholder="juan@ejemplo.com"
        required
      />

      <.input 
        field={@form[:age]} 
        type="number" 
        label="Edad"
        min="1"
        max="150"
      />

      <button 
        type="submit"
        class="w-full px-6 py-3 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition"
      >
        Guardar Usuario
      </button>
    </.form>
  </div>
</Layouts.app>
```

### 1.4 Validaciones Personalizadas

```elixir
defmodule MyApp.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset

  # ... schema ...

  def changeset(user, attrs) do
    user
    |> cast(attrs, [:email, :name, :birth_date])
    |> validate_required([:email, :name])
    |> validate_adult()
    |> validate_email_domain()
  end

  defp validate_adult(changeset) do
    validate_change(changeset, :birth_date, fn :birth_date, birth_date ->
      today = Date.utc_today()
      age = Date.diff(today, birth_date) / 365.25

      if age >= 18 do
        []
      else
        [birth_date: "debes ser mayor de 18 años"]
      end
    end)
  end

  defp validate_email_domain(changeset) do
    validate_change(changeset, :email, fn :email, email ->
      allowed_domains = ["empresa.com", "ejemplo.com"]
      [_name, domain] = String.split(email, "@")

      if domain in allowed_domains do
        []
      else
        [email: "el dominio debe ser #{Enum.join(allowed_domains, " o ")}"]
      end
    end)
  end
end
```

---

## 2. Testing con Datos Relacionados (FK) {#testing-datos-relacionados}

### 2.1 Configuración de ExMachina para Factories

Primero, agregar la dependencia en `mix.exs`:

```elixir
defp deps do
  [
    {:ex_machina, "~> 2.7", only: :test}
  ]
end
```

### 2.2 Crear Factory con Relaciones

```elixir
# test/support/factory.ex
defmodule MyApp.Factory do
  use ExMachina.Ecto, repo: MyApp.Repo

  def user_factory do
    %MyApp.Accounts.User{
      name: sequence(:name, &"Usuario #{&1}"),
      email: sequence(:email, &"user#{&1}@ejemplo.com"),
      age: 25
    }
  end

  def post_factory do
    %MyApp.Blog.Post{
      title: sequence(:title, &"Post #{&1}"),
      body: "Contenido del post",
      published: false,
      # Relación FK - crea automáticamente un user
      user: build(:user)
    }
  end

  def comment_factory do
    %MyApp.Blog.Comment{
      body: "Este es un comentario",
      # Relaciones FK múltiples
      user: build(:user),
      post: build(:post)
    }
  end

  def category_factory do
    %MyApp.Blog.Category{
      name: sequence(:name, &"Categoría #{&1}")
    }
  end

  # Factory con relación many_to_many
  def post_with_categories_factory do
    %MyApp.Blog.Post{
      title: "Post con categorías",
      body: "Contenido",
      user: build(:user),
      categories: build_list(3, :category)
    }
  end
end
```

### 2.3 Testing de Contextos con Datos Relacionados

```elixir
# test/my_app/blog_test.exs
defmodule MyApp.BlogTest do
  use MyApp.DataCase
  import MyApp.Factory

  alias MyApp.Blog

  describe "posts" do
    test "create_post/1 crea un post asociado a un usuario" do
      user = insert(:user)
      
      attrs = %{
        title: "Mi primer post",
        body: "Contenido del post",
        user_id: user.id
      }

      assert {:ok, post} = Blog.create_post(attrs)
      assert post.user_id == user.id
      assert post.title == "Mi primer post"
    end

    test "create_post/1 falla sin user_id válido" do
      attrs = %{
        title: "Post sin usuario",
        body: "Contenido"
      }

      assert {:error, changeset} = Blog.create_post(attrs)
      assert "can't be blank" in errors_on(changeset).user_id
    end

    test "list_posts_with_user/0 precarga el usuario" do
      user = insert(:user, name: "Juan")
      insert(:post, user: user, title: "Post de Juan")
      insert(:post, user: user, title: "Otro post de Juan")

      posts = Blog.list_posts_with_user()

      assert length(posts) == 2
      assert Enum.all?(posts, fn post -> 
        Ecto.assoc_loaded?(post.user) 
      end)
      assert Enum.all?(posts, fn post -> 
        post.user.name == "Juan" 
      end)
    end
  end

  describe "comments con relaciones encadenadas" do
    test "crear comentario requiere user y post válidos" do
      user = insert(:user)
      post = insert(:post)

      attrs = %{
        body: "Gran post!",
        user_id: user.id,
        post_id: post.id
      }

      assert {:ok, comment} = Blog.create_comment(attrs)
      assert comment.user_id == user.id
      assert comment.post_id == post.id
    end

    test "list_comments_with_associations/0 precarga todo" do
      # Crear datos relacionados
      user1 = insert(:user, name: "María")
      user2 = insert(:user, name: "Pedro")
      post = insert(:post, user: user1, title: "Post de María")
      
      insert(:comment, user: user2, post: post, body: "Comentario de Pedro")
      insert(:comment, user: user1, post: post, body: "Respuesta de María")

      comments = Blog.list_comments_with_associations()

      assert length(comments) == 2
      
      # Verificar que las asociaciones están precargadas
      Enum.each(comments, fn comment ->
        assert Ecto.assoc_loaded?(comment.user)
        assert Ecto.assoc_loaded?(comment.post)
        assert Ecto.assoc_loaded?(comment.post.user)
      end)
    end

    test "eliminar post elimina comentarios en cascada" do
      post = insert(:post)
      insert_list(3, :comment, post: post)

      assert Repo.aggregate(Blog.Comment, :count) == 3

      Blog.delete_post(post)

      assert Repo.aggregate(Blog.Comment, :count) == 0
    end
  end

  describe "relaciones many_to_many" do
    test "asociar categorías a un post" do
      post = insert(:post)
      categories = insert_list(3, :category)
      category_ids = Enum.map(categories, & &1.id)

      {:ok, updated_post} = Blog.update_post_categories(post, category_ids)

      post_with_categories = Blog.get_post_with_categories!(updated_post.id)
      assert length(post_with_categories.categories) == 3
    end
  end
end
```

### 2.4 Helpers para Testing

```elixir
# test/support/data_case.ex
defmodule MyApp.DataCase do
  use ExUnit.CaseTemplate

  using do
    quote do
      alias MyApp.Repo
      import Ecto
      import Ecto.Changeset
      import Ecto.Query
      import MyApp.DataCase
      import MyApp.Factory
    end
  end

  setup tags do
    MyApp.DataCase.setup_sandbox(tags)
    :ok
  end

  def setup_sandbox(tags) do
    pid = Ecto.Adapters.SQL.Sandbox.start_owner!(MyApp.Repo, shared: not tags[:async])
    on_exit(fn -> Ecto.Adapters.SQL.Sandbox.stop_owner(pid) end)
  end

  def errors_on(changeset) do
    Ecto.Changeset.traverse_errors(changeset, fn {message, opts} ->
      Regex.replace(~r"%{(\w+)}", message, fn _, key ->
        opts |> Keyword.get(String.to_existing_atom(key), key) |> to_string()
      end)
    end)
  end
end
```

---

## 3. Testing de HTML Estático vs LiveView {#testing-html}

### 3.1 Testing de Controladores Estáticos

```elixir
# test/my_app_web/controllers/page_controller_test.exs
defmodule MyAppWeb.PageControllerTest do
  use MyAppWeb.ConnCase
  import MyApp.Factory

  describe "GET /" do
    test "muestra la página de inicio", %{conn: conn} do
      conn = get(conn, ~p"/")
      
      assert html_response(conn, 200)
      assert html = html_response(conn, 200)
      assert html =~ "Bienvenido"
    end
  end

  describe "GET /about" do
    test "muestra la página sobre nosotros", %{conn: conn} do
      conn = get(conn, ~p"/about")
      
      assert html_response(conn, 200) =~ "Sobre Nosotros"
    end
  end
end
```

### 3.2 Testing de LiveView

**Reglas importantes**:
- Usar `Phoenix.LiveViewTest`
- Usar `element/2` y `has_element?/2` en lugar de probar HTML crudo
- Usar IDs únicos en elementos clave
- Usar `Floki` para selectores complejos

```elixir
# test/my_app_web/live/user_live_test.exs
defmodule MyAppWeb.UserLiveTest do
  use MyAppWeb.ConnCase
  import Phoenix.LiveViewTest
  import MyApp.Factory

  describe "Index" do
    test "muestra todos los usuarios", %{conn: conn} do
      users = insert_list(3, :user)

      {:ok, view, html} = live(conn, ~p"/users")

      assert html =~ "Listado de Usuarios"
      
      # Verificar que cada usuario aparece
      Enum.each(users, fn user ->
        assert has_element?(view, "#user-#{user.id}")
        assert html =~ user.name
      end)
    end

    test "permite crear un nuevo usuario", %{conn: conn} do
      {:ok, view, _html} = live(conn, ~p"/users")

      # Click en botón nuevo
      assert view |> element("a", "Nuevo Usuario") |> render_click()

      # Llenar formulario
      assert view
             |> form("#user-form", user: %{name: "", email: ""})
             |> render_change() =~ "can&#39;t be blank"

      # Submit válido
      assert view
             |> form("#user-form", user: %{
               name: "Juan Pérez",
               email: "juan@ejemplo.com",
               age: 30
             })
             |> render_submit()

      # Verificar flash y redirección
      assert_redirected(view, ~p"/users/#{user_id}")
      
      # Verificar que se creó en BD
      user = Repo.get_by!(User, email: "juan@ejemplo.com")
      assert user.name == "Juan Pérez"
    end

    test "permite eliminar un usuario", %{conn: conn} do
      user = insert(:user)

      {:ok, view, _html} = live(conn, ~p"/users")

      # Click en eliminar
      assert view
             |> element("#user-#{user.id} button[phx-click='delete']")
             |> render_click()

      # Verificar que no existe más
      refute has_element?(view, "#user-#{user.id}")
      assert Repo.get(User, user.id) == nil
    end
  end

  describe "Show" do
    test "muestra un usuario con sus posts", %{conn: conn} do
      user = insert(:user, name: "María")
      posts = insert_list(3, :post, user: user)

      {:ok, view, html} = live(conn, ~p"/users/#{user}")

      assert html =~ "María"
      
      Enum.each(posts, fn post ->
        assert has_element?(view, "#post-#{post.id}")
        assert html =~ post.title
      end)
    end
  end

  describe "Form validaciones en tiempo real" do
    test "valida email en tiempo real", %{conn: conn} do
      {:ok, view, _html} = live(conn, ~p"/users/new")

      # Email inválido
      assert view
             |> form("#user-form", user: %{email: "invalido"})
             |> render_change() =~ "tiene un formato inválido"

      # Email válido
      refute view
             |> form("#user-form", user: %{email: "valido@ejemplo.com"})
             |> render_change() =~ "tiene un formato inválido"
    end
  end
end
```

### 3.3 Testing con Floki para Selectores Complejos

```elixir
test "verifica estructura compleja del DOM", %{conn: conn} do
  user = insert(:user)
  {:ok, _view, html} = live(conn, ~p"/users/#{user}")

  {:ok, document} = Floki.parse_document(html)

  # Buscar elementos específicos
  assert [_element] = Floki.find(document, "#user-profile .user-name")
  assert [_element] = Floki.find(document, "button[phx-click='edit']")
  
  # Verificar atributos
  [button] = Floki.find(document, "#edit-button")
  assert Floki.attribute(button, "phx-click") == ["edit"]
  
  # Verificar texto
  assert Floki.find(document, ".user-email") 
         |> Floki.text() 
         |> String.contains?(user.email)
end
```

### 3.4 Testing de Streams en LiveView

```elixir
test "actualiza stream cuando se agrega item", %{conn: conn} do
  {:ok, view, _html} = live(conn, ~p"/messages")

  # Estado inicial vacío
  refute has_element?(view, "#messages [id^='messages-']")

  # Enviar mensaje
  view
  |> form("#message-form", message: %{content: "Hola mundo"})
  |> render_submit()

  # Verificar que aparece en el stream
  assert has_element?(view, "#messages [id^='messages-']")
  assert render(view) =~ "Hola mundo"
end
```

---

## 4. LiveView con PubSub en Múltiples Nodos {#liveview-pubsub-multinodo}

### 4.1 Configuración de PubSub para Multi-Nodo

```elixir
# config/config.exs
config :my_app, MyApp.PubSub,
  name: MyApp.PubSub,
  adapter: Phoenix.PubSub.PG2

# lib/my_app/application.ex
defmodule MyApp.Application do
  use Application

  def start(_type, _args) do
    children = [
      # PubSub para comunicación multi-nodo
      {Phoenix.PubSub, name: MyApp.PubSub},
      MyApp.Repo,
      MyAppWeb.Endpoint
    ]

    opts = [strategy: :one_for_one, name: MyApp.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

### 4.2 Configuración de Nodos (Clustering)

```elixir
# config/runtime.exs
import Config

if config_env() == :prod do
  # Configurar nombre del nodo
  node_name = System.get_env("NODE_NAME") || "myapp"
  node_host = System.get_env("NODE_HOST") || "localhost"
  
  # Formato: nombre@host
  :net_kernel.start([:"#{node_name}@#{node_host}"])
  
  # Cookie para autenticación entre nodos
  Node.set_cookie(:my_secret_cookie)
end
```

### 4.3 LiveView con PubSub - Patrón Completo

```elixir
defmodule MyAppWeb.ChatLive do
  use MyAppWeb, :live_view
  alias MyApp.Chat
  alias Phoenix.PubSub

  @topic "chat:messages"

  def mount(_params, _session, socket) do
    if connected?(socket) do
      # Suscribirse al topic cuando el socket está conectado
      PubSub.subscribe(MyApp.PubSub, @topic)
    end

    messages = Chat.list_recent_messages()

    {:ok,
     socket
     |> assign(:messages_empty?, messages == [])
     |> stream(:messages, messages)}
  end

  def handle_event("send_message", %{"message" => message_params}, socket) do
    case Chat.create_message(message_params) do
      {:ok, message} ->
        # Broadcast a TODOS los nodos suscritos
        PubSub.broadcast(
          MyApp.PubSub,
          @topic,
          {:new_message, message}
        )

        {:noreply,
         socket
         |> put_flash(:info, "Mensaje enviado")
         |> assign(:message_form, to_form(Chat.change_message(%Message{})))}

      {:error, changeset} ->
        {:noreply, assign(socket, message_form: to_form(changeset))}
    end
  end

  # Recibir broadcasts de CUALQUIER nodo
  def handle_info({:new_message, message}, socket) do
    {:noreply,
     socket
     |> assign(:messages_empty?, false)
     |> stream_insert(:messages, message, at: 0)}
  end

  # Manejar mensaje de usuario eliminado
  def handle_info({:message_deleted, message_id}, socket) do
    message = %{id: message_id}
    {:noreply, stream_delete(socket, :messages, message)}
  end
end
```

### 4.4 Contexto con PubSub Broadcasting

```elixir
defmodule MyApp.Chat do
  alias MyApp.Repo
  alias MyApp.Chat.Message
  alias Phoenix.PubSub

  @topic "chat:messages"

  def create_message(attrs) do
    %Message{}
    |> Message.changeset(attrs)
    |> Repo.insert()
    |> case do
      {:ok, message} = result ->
        # Broadcast automático después de crear
        broadcast({:new_message, message})
        result

      error ->
        error
    end
  end

  def delete_message(%Message{} = message) do
    Repo.delete(message)
    |> case do
      {:ok, deleted_message} = result ->
        broadcast({:message_deleted, deleted_message.id})
        result

      error ->
        error
    end
  end

  def update_message(%Message{} = message, attrs) do
    message
    |> Message.changeset(attrs)
    |> Repo.update()
    |> case do
      {:ok, updated_message} = result ->
        broadcast({:message_updated, updated_message})
        result

      error ->
        error
    end
  end

  # Función privada para broadcast
  defp broadcast(message) do
    PubSub.broadcast(MyApp.PubSub, @topic, message)
  end

  def subscribe do
    PubSub.subscribe(MyApp.PubSub, @topic)
  end
end
```

### 4.5 Testing de PubSub

```elixir
defmodule MyApp.ChatTest do
  use MyApp.DataCase
  import MyApp.Factory

  alias MyApp.Chat
  alias Phoenix.PubSub

  describe "PubSub broadcasting" do
    test "crear mensaje hace broadcast" do
      # Suscribirse al topic
      :ok = Chat.subscribe()

      user = insert(:user)
      attrs = %{content: "Hola!", user_id: user.id}

      {:ok, message} = Chat.create_message(attrs)

      # Verificar que recibimos el broadcast
      assert_receive {:new_message, ^message}
    end

    test "eliminar mensaje hace broadcast del ID" do
      :ok = Chat.subscribe()

      message = insert(:message)
      {:ok, _} = Chat.delete_message(message)

      assert_receive {:message_deleted, message_id}
      assert message_id == message.id
    end
  end
end
```

### 4.6 Conectar Nodos Manualmente (para desarrollo/testing)

```elixir
# En iex de nodo 1
iex --name node1@localhost -S mix phx.server

# En iex de nodo 2  
iex --name node2@localhost -S mix phx.server

# En node2, conectar a node1
Node.connect(:"node1@localhost")

# Verificar conexión
Node.list()
# => [:"node1@localhost"]

# Verificar PubSub funciona entre nodos
Phoenix.PubSub.subscribe(MyApp.PubSub, "test")
Phoenix.PubSub.broadcast(MyApp.PubSub, "test", {:mensaje, "hola desde node2"})
# En node1 deberías recibir el mensaje
```

---

## 5. Consumo de Servicios RPC/EPMD y Refresco de Vistas {#rpc-refresco-vistas}

### 5.1 Conceptos de RPC en Elixir

EPMD (Erlang Port Mapper Daemon) permite que nodos Erlang/Elixir se descubran y comuniquen.

### 5.2 Llamadas RPC Básicas

```elixir
defmodule MyApp.RemoteService do
  @doc """
  Llama a una función en un nodo remoto
  """
  def call_remote(node, module, function, args) do
    case :rpc.call(node, module, function, args) do
      {:badrpc, reason} ->
        {:error, reason}

      result ->
        {:ok, result}
    end
  end

  @doc """
  Obtiene datos de usuarios desde nodo remoto
  """
  def get_remote_users(remote_node) do
    call_remote(
      remote_node,
      MyApp.Accounts,
      :list_users,
      []
    )
  end
end
```

### 5.3 GenServer para Sincronización con Nodo Remoto

```elixir
defmodule MyApp.RemoteSync do
  use GenServer
  alias Phoenix.PubSub
  require Logger

  @sync_interval :timer.seconds(30)

  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def init(opts) do
    remote_node = Keyword.get(opts, :remote_node)
    
    # Programar primera sincronización
    schedule_sync()

    {:ok, %{remote_node: remote_node, last_sync: nil}}
  end

  def handle_info(:sync, state) do
    Logger.info("Sincronizando con nodo remoto: #{state.remote_node}")

    case sync_data(state.remote_node) do
      {:ok, data} ->
        # Broadcast de datos actualizados
        PubSub.broadcast(
          MyApp.PubSub,
          "remote:sync",
          {:data_updated, data}
        )

        schedule_sync()
        {:noreply, %{state | last_sync: DateTime.utc_now()}}

      {:error, reason} ->
        Logger.error("Error en sincronización: #{inspect(reason)}")
        schedule_sync()
        {:noreply, state}
    end
  end

  defp sync_data(remote_node) do
    case :rpc.call(remote_node, MyApp.DataService, :get_latest_data, []) do
      {:badrpc, reason} ->
        {:error, reason}

      data ->
        # Guardar datos localmente
        MyApp.Cache.put(:remote_data, data)
        {:ok, data}
    end
  end

  defp schedule_sync do
    Process.send_after(self(), :sync, @sync_interval)
  end
end
```

### 5.4 LiveView que Refresca con Datos RPC

```elixir
defmodule MyAppWeb.DashboardLive do
  use MyAppWeb, :live_view
  alias Phoenix.PubSub

  def mount(_params, _session, socket) do
    if connected?(socket) do
      # Suscribirse a actualizaciones remotas
      PubSub.subscribe(MyApp.PubSub, "remote:sync")
    end

    # Obtener datos iniciales (posiblemente del cache)
    remote_data = MyApp.Cache.get(:remote_data, [])

    {:ok,
     socket
     |> assign(:remote_data, remote_data)
     |> assign(:last_update, nil)
     |> assign(:sync_status, :idle)}
  end

  # Refrescar cuando llegan datos del nodo remoto
  def handle_info({:data_updated, data}, socket) do
    {:noreply,
     socket
     |> assign(:remote_data, data)
     |> assign(:last_update, DateTime.utc_now())
     |> put_flash(:info, "Datos actualizados desde servidor remoto")}
  end

  # Permitir refrescar manualmente
  def handle_event("refresh", _params, socket) do
    send(self(), :manual_refresh)
    {:noreply, assign(socket, :sync_status, :syncing)}
  end

  def handle_info(:manual_refresh, socket) do
    remote_node = Application.get_env(:my_app, :remote_node)

    case MyApp.RemoteService.get_remote_users(remote_node) do
      {:ok, users} ->
        {:noreply,
         socket
         |> assign(:remote_data, users)
         |> assign(:sync_status, :idle)
         |> put_flash(:info, "Datos actualizados")}

      {:error, reason} ->
        {:noreply,
         socket
         |> assign(:sync_status, :error)
         |> put_flash(:error, "Error al sincronizar: #{inspect(reason)}")}
    end
  end
end
```

### 5.5 Template para Dashboard con Datos Remotos

```heex
<Layouts.app flash={@flash}>
  <div class="max-w-7xl mx-auto">
    <div class="flex justify-between items-center mb-8">
      <h1 class="text-3xl font-bold">Dashboard - Datos Remotos</h1>
      
      <button
        phx-click="refresh"
        class="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700"
        disabled={@sync_status == :syncing}
      >
        <%= if @sync_status == :syncing do %>
          Sincronizando...
        <% else %>
          Refrescar
        <% end %>
      </button>
    </div>

    <%= if @last_update do %>
      <p class="text-sm text-gray-600 mb-4">
        Última actualización: <%= Calendar.strftime(@last_update, "%H:%M:%S") %>
      </p>
    <% end %>

    <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
      <%= for item <- @remote_data do %>
        <div class="bg-white p-6 rounded-lg shadow">
          <h3 class="font-bold text-lg mb-2"><%= item.name %></h3>
          <p class="text-gray-600"><%= item.description %></p>
        </div>
      <% end %>
    </div>
  </div>
</Layouts.app>
```

### 5.6 Configuración de Nodos Remotos

```elixir
# config/runtime.exs
import Config

if config_env() == :prod do
  # Nodo remoto para RPC
  config :my_app,
    remote_node: System.get_env("REMOTE_NODE") || :"remote@hostname"
end

# lib/my_app/application.ex
defmodule MyApp.Application do
  use Application

  def start(_type, _args) do
    children = [
      MyApp.Repo,
      {Phoenix.PubSub, name: MyApp.PubSub},
      MyAppWeb.Endpoint,
      # Sincronización con nodo remoto
      {MyApp.RemoteSync, remote_node: remote_node()}
    ]

    opts = [strategy: :one_for_one, name: MyApp.Supervisor]
    Supervisor.start_link(children, opts)
  end

  defp remote_node do
    Application.get_env(:my_app, :remote_node, :nonode@nohost)
  end
end
```

---

## 6. Patrones de Refresco de Vistas HTML {#patrones-refresco}

### 6.1 Refresco Automático con Streams

**Mejor práctica**: Usar streams para colecciones grandes.

```elixir
defmodule MyAppWeb.TimelineLive do
  use MyAppWeb, :live_view
  alias Phoenix.PubSub

  def mount(_params, _session, socket) do
    if connected?(socket) do
      PubSub.subscribe(MyApp.PubSub, "timeline:posts")
    end

    posts = MyApp.Social.list_timeline_posts()

    {:ok,
     socket
     |> assign(:posts_empty?, posts == [])
     |> stream(:posts, posts)}
  end

  # Nuevo post - insertar al inicio del stream
  def handle_info({:new_post, post}, socket) do
    {:noreply,
     socket
     |> assign(:posts_empty?, false)
     |> stream_insert(:posts, post, at: 0)}
  end

  # Post actualizado - reemplazar en stream
  def handle_info({:post_updated, post}, socket) do
    {:noreply, stream_insert(socket, :posts, post)}
  end

  # Post eliminado - remover del stream
  def handle_info({:post_deleted, post_id}, socket) do
    post = %{id: post_id}
    {:noreply, stream_delete(socket, :posts, post)}
  end
end
```

### 6.2 Template para Streams

```heex
<Layouts.app flash={@flash}>
  <div class="max-w-3xl mx-auto">
    <h1 class="text-3xl font-bold mb-8">Timeline</h1>

    <%= if @posts_empty? do %>
      <p class="text-gray-500 text-center py-12">
        No hay posts todavía
      </p>
    <% else %>
      <div id="posts" phx-update="stream" class="space-y-6">
        <%= for {id, post} <- @streams.posts do %>
          <div id={id} class="bg-white p-6 rounded-lg shadow">
            <div class="flex items-start justify-between">
              <div>
                <h3 class="font-bold text-lg"><%= post.title %></h3>
                <p class="text-sm text-gray-600 mt-1">
                  Por <%= post.user.name %> · 
                  <%= Calendar.strftime(post.inserted_at, "%d/%m/%Y %H:%M") %>
                </p>
              </div>
              
              <button 
                phx-click="delete_post" 
                phx-value-id={post.id}
                class="text-red-600 hover:text-red-800"
              >
                Eliminar
              </button>
            </div>
            
            <p class="mt-4 text-gray-800"><%= post.body %></p>
          </div>
        <% end %>
      </div>
    <% end %>
  </div>
</Layouts.app>
```

### 6.3 Refresco con Asignaciones Simples

Para datos pequeños, usar `assign/2`:

```elixir
defmodule MyAppWeb.CounterLive do
  use MyAppWeb, :live_view
  alias Phoenix.PubSub

  def mount(_params, _session, socket) do
    if connected?(socket) do
      PubSub.subscribe(MyApp.PubSub, "counter")
    end

    {:ok, assign(socket, :count, get_current_count())}
  end

  def handle_info({:count_updated, new_count}, socket) do
    {:noreply, assign(socket, :count, new_count)}
  end

  def handle_event("increment", _params, socket) do
    new_count = socket.assigns.count + 1
    
    PubSub.broadcast(MyApp.PubSub, "counter", {:count_updated, new_count})
    
    {:noreply, assign(socket, :count, new_count)}
  end

  defp get_current_count do
    MyApp.Counter.get_count()
  end
end
```

### 6.4 Refresco con JS Hooks para Scroll

```elixir
# assets/js/app.js
let Hooks = {}

Hooks.InfiniteScroll = {
  mounted() {
    this.pending = false
    
    this.observer = new IntersectionObserver(entries => {
      const target = entries[0]
      if (target.isIntersecting && !this.pending) {
        this.pending = true
        this.pushEvent("load-more", {}, () => {
          this.pending = false
        })
      }
    })
    
    this.observer.observe(this.el)
  },
  
  destroyed() {
    this.observer.disconnect()
  }
}

let liveSocket = new LiveSocket("/live", Socket, {
  params: {_csrf_token: csrfToken},
  hooks: Hooks
})
```

```elixir
# LiveView con scroll infinito
defmodule MyAppWeb.FeedLive do
  use MyAppWeb, :live_view

  @per_page 20

  def mount(_params, _session, socket) do
    {:ok,
     socket
     |> assign(:page, 1)
     |> stream(:posts, load_posts(1))}
  end

  def handle_event("load-more", _params, socket) do
    next_page = socket.assigns.page + 1
    posts = load_posts(next_page)

    {:noreply,
     socket
     |> assign(:page, next_page)
     |> stream(:posts, posts)}
  end

  defp load_posts(page) do
    offset = (page - 1) * @per_page
    MyApp.Social.list_posts(limit: @per_page, offset: offset)
  end
end
```

```heex
<div id="posts" phx-update="stream">
  <%= for {id, post} <- @streams.posts do %>
    <div id={id}><%= post.title %></div>
  <% end %>
</div>

<!-- Trigger para infinite scroll -->
<div id="infinite-scroll-marker" phx-hook="InfiniteScroll"></div>
```

### 6.5 Optimización: Actualizar Solo lo Necesario

```elixir
defmodule MyAppWeb.ProductLive do
  use MyAppWeb, :live_view

  def mount(%{"id" => id}, _session, socket) do
    if connected?(socket) do
      PubSub.subscribe(MyApp.PubSub, "product:#{id}")
    end

    product = MyApp.Catalog.get_product!(id)

    {:ok,
     socket
     |> assign(:product, product)
     |> assign(:stock, product.stock)
     |> assign(:price, product.price)}
  end

  # Solo actualizar stock (no re-renderizar todo el producto)
  def handle_info({:stock_updated, new_stock}, socket) do
    {:noreply, assign(socket, :stock, new_stock)}
  end

  # Solo actualizar precio
  def handle_info({:price_updated, new_price}, socket) do
    {:noreply, assign(socket, :price, new_price)}
  end
end
```

```heex
<Layouts.app flash={@flash}>
  <div class="max-w-4xl mx-auto">
    <h1 class="text-3xl font-bold"><%= @product.name %></h1>
    
    <!-- Solo esta parte se re-renderiza cuando cambia @price -->
    <p class="text-2xl font-bold text-green-600 mt-4">
      $<%= @price %>
    </p>
    
    <!-- Solo esta parte se re-renderiza cuando cambia @stock -->
    <p class="text-sm text-gray-600 mt-2">
      Stock disponible: <%= @stock %> unidades
    </p>
    
    <!-- Esto no se re-renderiza a menos que @product cambie -->
    <div class="mt-6">
      <p class="text-gray-800"><%= @product.description %></p>
    </div>
  </div>
</Layouts.app>
```

---

## Resumen de Mejores Prácticas

### Validaciones
✅ SIEMPRE validar en el servidor con `Ecto.Changeset`
✅ Usar `phx-change` para validaciones en tiempo real
✅ Usar `to_form/2` en LiveView, `<.form>` y `<.input>` en templates
✅ Nunca acceder changeset directamente en templates

### Testing
✅ Usar ExMachina para factories con relaciones FK
✅ Usar `Phoenix.LiveViewTest` para LiveViews
✅ Usar `element/2` y `has_element?/2` en lugar de HTML crudo
✅ Dar IDs únicos a elementos clave para testing
✅ Verificar asociaciones precargadas en tests

### PubSub Multi-Nodo
✅ Usar `Phoenix.PubSub.PG2` para distribuir mensajes
✅ Suscribirse en `mount/3` cuando `connected?(socket)`
✅ Hacer broadcast después de operaciones exitosas
✅ Configurar nombres de nodo y cookies para clustering

### RPC y Sincronización
✅ Usar `:rpc.call/4` para llamadas remotas
✅ Implementar GenServer para polling periódico
✅ Usar PubSub para notificar actualizaciones
✅ Manejar errores de conexión gracefully

### Refresco de Vistas
✅ Usar streams para colecciones grandes
✅ Usar assigns simples para datos pequeños
✅ Implementar JS Hooks para scroll infinito
✅ Actualizar solo lo necesario (granularidad)

---

## Recursos Adicionales

- [Phoenix LiveView Docs](https://hexdocs.pm/phoenix_live_view)
- [Ecto Changeset Docs](https://hexdocs.pm/ecto/Ecto.Changeset.html)
- [Phoenix PubSub Docs](https://hexdocs.pm/phoenix_pubsub)
- [Distributed Elixir](https://elixir-lang.org/getting-started/mix-otp/distributed-tasks.html)
# Guía Completa de Trabajo con Phoenix Framework

Esta guía cubre las mejores prácticas para trabajar con Phoenix, incluyendo validaciones, testing, LiveView con PubSub multi-nodo, y consumo de servicios RPC.

---

## Tabla de Contenidos

1. [Validaciones en HTML y LiveView](#validaciones)
2. [Testing con Datos Relacionados (FK)](#testing-datos-relacionados)
3. [Testing de HTML Estático vs LiveView](#testing-html)
4. [LiveView con PubSub en Múltiples Nodos](#liveview-pubsub-multinodo)
5. [Consumo de Servicios RPC/EPMD y Refresco de Vistas](#rpc-refresco-vistas)
6. [Patrones de Refresco de Vistas HTML](#patrones-refresco)

---

## 1. Validaciones en HTML y LiveView {#validaciones}

### 1.1 Validaciones del Lado del Servidor con Ecto.Changeset

**Regla fundamental**: TODAS las validaciones DEBEN ocurrir en el servidor mediante `Ecto.Changeset`.

```elixir
defmodule MyApp.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset

  schema "users" do
    field :email, :string
    field :name, :string
    field :age, :integer
    timestamps()
  end

  def changeset(user, attrs) do
    user
    |> cast(attrs, [:email, :name, :age])
    |> validate_required([:email, :name])
    |> validate_format(:email, ~r/@/)
    |> validate_length(:name, min: 2, max: 100)
    |> validate_number(:age, greater_than: 0, less_than: 150)
    |> unique_constraint(:email)
  end
end
```

### 1.2 Validaciones en Tiempo Real con LiveView

**Patrón correcto**: Usar `phx-change` para validaciones mientras el usuario escribe.

```elixir
defmodule MyAppWeb.UserLive.Form do
  use MyAppWeb, :live_view

  def mount(_params, _session, socket) do
    changeset = Accounts.change_user(%User{})
    
    {:ok,
     socket
     |> assign(:form, to_form(changeset))
     |> assign(:user, nil)}
  end

  # Validación en tiempo real mientras el usuario escribe
  def handle_event("validate", %{"user" => user_params}, socket) do
    changeset =
      %User{}
      |> Accounts.change_user(user_params)
      |> Map.put(:action, :validate)

    {:noreply, assign(socket, form: to_form(changeset))}
  end

  # Guardado final con validación completa
  def handle_event("save", %{"user" => user_params}, socket) do
    case Accounts.create_user(user_params) do
      {:ok, user} ->
        {:noreply,
         socket
         |> put_flash(:info, "Usuario creado exitosamente")
         |> push_navigate(to: ~p"/users/#{user}")}

      {:error, %Ecto.Changeset{} = changeset} ->
        {:noreply, assign(socket, form: to_form(changeset))}
    end
  end
end
```

### 1.3 Template HTML para Validaciones

**Reglas importantes**:
- SIEMPRE usar `to_form/2` en el LiveView
- SIEMPRE usar `<.form>` y `<.input>` en la template
- NUNCA acceder al changeset directamente en la template
- SIEMPRE dar un ID único al form

```heex
<Layouts.app flash={@flash}>
  <div class="mx-auto max-w-2xl">
    <h1 class="text-3xl font-bold mb-8">Nuevo Usuario</h1>

    <.form 
      for={@form} 
      id="user-form" 
      phx-change="validate" 
      phx-submit="save"
      class="space-y-6"
    >
      <.input 
        field={@form[:name]} 
        type="text" 
        label="Nombre completo"
        placeholder="Juan Pérez"
        required
      />

      <.input 
        field={@form[:email]} 
        type="email" 
        label="Email"
        placeholder="juan@ejemplo.com"
        required
      />

      <.input 
        field={@form[:age]} 
        type="number" 
        label="Edad"
        min="1"
        max="150"
      />

      <button 
        type="submit"
        class="w-full px-6 py-3 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition"
      >
        Guardar Usuario
      </button>
    </.form>
  </div>
</Layouts.app>
```

### 1.4 Validaciones Personalizadas

```elixir
defmodule MyApp.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset

  # ... schema ...

  def changeset(user, attrs) do
    user
    |> cast(attrs, [:email, :name, :birth_date])
    |> validate_required([:email, :name])
    |> validate_adult()
    |> validate_email_domain()
  end

  defp validate_adult(changeset) do
    validate_change(changeset, :birth_date, fn :birth_date, birth_date ->
      today = Date.utc_today()
      age = Date.diff(today, birth_date) / 365.25

      if age >= 18 do
        []
      else
        [birth_date: "debes ser mayor de 18 años"]
      end
    end)
  end

  defp validate_email_domain(changeset) do
    validate_change(changeset, :email, fn :email, email ->
      allowed_domains = ["empresa.com", "ejemplo.com"]
      [_name, domain] = String.split(email, "@")

      if domain in allowed_domains do
        []
      else
        [email: "el dominio debe ser #{Enum.join(allowed_domains, " o ")}"]
      end
    end)
  end
end
```

---

## 2. Testing con Datos Relacionados (FK) {#testing-datos-relacionados}

### 2.1 Configuración de ExMachina para Factories

Primero, agregar la dependencia en `mix.exs`:

```elixir
defp deps do
  [
    {:ex_machina, "~> 2.7", only: :test}
  ]
end
```

### 2.2 Crear Factory con Relaciones

```elixir
# test/support/factory.ex
defmodule MyApp.Factory do
  use ExMachina.Ecto, repo: MyApp.Repo

  def user_factory do
    %MyApp.Accounts.User{
      name: sequence(:name, &"Usuario #{&1}"),
      email: sequence(:email, &"user#{&1}@ejemplo.com"),
      age: 25
    }
  end

  def post_factory do
    %MyApp.Blog.Post{
      title: sequence(:title, &"Post #{&1}"),
      body: "Contenido del post",
      published: false,
      # Relación FK - crea automáticamente un user
      user: build(:user)
    }
  end

  def comment_factory do
    %MyApp.Blog.Comment{
      body: "Este es un comentario",
      # Relaciones FK múltiples
      user: build(:user),
      post: build(:post)
    }
  end

  def category_factory do
    %MyApp.Blog.Category{
      name: sequence(:name, &"Categoría #{&1}")
    }
  end

  # Factory con relación many_to_many
  def post_with_categories_factory do
    %MyApp.Blog.Post{
      title: "Post con categorías",
      body: "Contenido",
      user: build(:user),
      categories: build_list(3, :category)
    }
  end
end
```

### 2.3 Testing de Contextos con Datos Relacionados

```elixir
# test/my_app/blog_test.exs
defmodule MyApp.BlogTest do
  use MyApp.DataCase
  import MyApp.Factory

  alias MyApp.Blog

  describe "posts" do
    test "create_post/1 crea un post asociado a un usuario" do
      user = insert(:user)
      
      attrs = %{
        title: "Mi primer post",
        body: "Contenido del post",
        user_id: user.id
      }

      assert {:ok, post} = Blog.create_post(attrs)
      assert post.user_id == user.id
      assert post.title == "Mi primer post"
    end

    test "create_post/1 falla sin user_id válido" do
      attrs = %{
        title: "Post sin usuario",
        body: "Contenido"
      }

      assert {:error, changeset} = Blog.create_post(attrs)
      assert "can't be blank" in errors_on(changeset).user_id
    end

    test "list_posts_with_user/0 precarga el usuario" do
      user = insert(:user, name: "Juan")
      insert(:post, user: user, title: "Post de Juan")
      insert(:post, user: user, title: "Otro post de Juan")

      posts = Blog.list_posts_with_user()

      assert length(posts) == 2
      assert Enum.all?(posts, fn post -> 
        Ecto.assoc_loaded?(post.user) 
      end)
      assert Enum.all?(posts, fn post -> 
        post.user.name == "Juan" 
      end)
    end
  end

  describe "comments con relaciones encadenadas" do
    test "crear comentario requiere user y post válidos" do
      user = insert(:user)
      post = insert(:post)

      attrs = %{
        body: "Gran post!",
        user_id: user.id,
        post_id: post.id
      }

      assert {:ok, comment} = Blog.create_comment(attrs)
      assert comment.user_id == user.id
      assert comment.post_id == post.id
    end

    test "list_comments_with_associations/0 precarga todo" do
      # Crear datos relacionados
      user1 = insert(:user, name: "María")
      user2 = insert(:user, name: "Pedro")
      post = insert(:post, user: user1, title: "Post de María")
      
      insert(:comment, user: user2, post: post, body: "Comentario de Pedro")
      insert(:comment, user: user1, post: post, body: "Respuesta de María")

      comments = Blog.list_comments_with_associations()

      assert length(comments) == 2
      
      # Verificar que las asociaciones están precargadas
      Enum.each(comments, fn comment ->
        assert Ecto.assoc_loaded?(comment.user)
        assert Ecto.assoc_loaded?(comment.post)
        assert Ecto.assoc_loaded?(comment.post.user)
      end)
    end

    test "eliminar post elimina comentarios en cascada" do
      post = insert(:post)
      insert_list(3, :comment, post: post)

      assert Repo.aggregate(Blog.Comment, :count) == 3

      Blog.delete_post(post)

      assert Repo.aggregate(Blog.Comment, :count) == 0
    end
  end

  describe "relaciones many_to_many" do
    test "asociar categorías a un post" do
      post = insert(:post)
      categories = insert_list(3, :category)
      category_ids = Enum.map(categories, & &1.id)

      {:ok, updated_post} = Blog.update_post_categories(post, category_ids)

      post_with_categories = Blog.get_post_with_categories!(updated_post.id)
      assert length(post_with_categories.categories) == 3
    end
  end
end
```

### 2.4 Helpers para Testing

```elixir
# test/support/data_case.ex
defmodule MyApp.DataCase do
  use ExUnit.CaseTemplate

  using do
    quote do
      alias MyApp.Repo
      import Ecto
      import Ecto.Changeset
      import Ecto.Query
      import MyApp.DataCase
      import MyApp.Factory
    end
  end

  setup tags do
    MyApp.DataCase.setup_sandbox(tags)
    :ok
  end

  def setup_sandbox(tags) do
    pid = Ecto.Adapters.SQL.Sandbox.start_owner!(MyApp.Repo, shared: not tags[:async])
    on_exit(fn -> Ecto.Adapters.SQL.Sandbox.stop_owner(pid) end)
  end

  def errors_on(changeset) do
    Ecto.Changeset.traverse_errors(changeset, fn {message, opts} ->
      Regex.replace(~r"%{(\w+)}", message, fn _, key ->
        opts |> Keyword.get(String.to_existing_atom(key), key) |> to_string()
      end)
    end)
  end
end
```

---

## 3. Testing de HTML Estático vs LiveView {#testing-html}

### 3.1 Testing de Controladores Estáticos

```elixir
# test/my_app_web/controllers/page_controller_test.exs
defmodule MyAppWeb.PageControllerTest do
  use MyAppWeb.ConnCase
  import MyApp.Factory

  describe "GET /" do
    test "muestra la página de inicio", %{conn: conn} do
      conn = get(conn, ~p"/")
      
      assert html_response(conn, 200)
      assert html = html_response(conn, 200)
      assert html =~ "Bienvenido"
    end
  end

  describe "GET /about" do
    test "muestra la página sobre nosotros", %{conn: conn} do
      conn = get(conn, ~p"/about")
      
      assert html_response(conn, 200) =~ "Sobre Nosotros"
    end
  end
end
```

### 3.2 Testing de LiveView

**Reglas importantes**:
- Usar `Phoenix.LiveViewTest`
- Usar `element/2` y `has_element?/2` en lugar de probar HTML crudo
- Usar IDs únicos en elementos clave
- Usar `Floki` para selectores complejos

```elixir
# test/my_app_web/live/user_live_test.exs
defmodule MyAppWeb.UserLiveTest do
  use MyAppWeb.ConnCase
  import Phoenix.LiveViewTest
  import MyApp.Factory

  describe "Index" do
    test "muestra todos los usuarios", %{conn: conn} do
      users = insert_list(3, :user)

      {:ok, view, html} = live(conn, ~p"/users")

      assert html =~ "Listado de Usuarios"
      
      # Verificar que cada usuario aparece
      Enum.each(users, fn user ->
        assert has_element?(view, "#user-#{user.id}")
        assert html =~ user.name
      end)
    end

    test "permite crear un nuevo usuario", %{conn: conn} do
      {:ok, view, _html} = live(conn, ~p"/users")

      # Click en botón nuevo
      assert view |> element("a", "Nuevo Usuario") |> render_click()

      # Llenar formulario
      assert view
             |> form("#user-form", user: %{name: "", email: ""})
             |> render_change() =~ "can&#39;t be blank"

      # Submit válido
      assert view
             |> form("#user-form", user: %{
               name: "Juan Pérez",
               email: "juan@ejemplo.com",
               age: 30
             })
             |> render_submit()

      # Verificar flash y redirección
      assert_redirected(view, ~p"/users/#{user_id}")
      
      # Verificar que se creó en BD
      user = Repo.get_by!(User, email: "juan@ejemplo.com")
      assert user.name == "Juan Pérez"
    end

    test "permite eliminar un usuario", %{conn: conn} do
      user = insert(:user)

      {:ok, view, _html} = live(conn, ~p"/users")

      # Click en eliminar
      assert view
             |> element("#user-#{user.id} button[phx-click='delete']")
             |> render_click()

      # Verificar que no existe más
      refute has_element?(view, "#user-#{user.id}")
      assert Repo.get(User, user.id) == nil
    end
  end

  describe "Show" do
    test "muestra un usuario con sus posts", %{conn: conn} do
      user = insert(:user, name: "María")
      posts = insert_list(3, :post, user: user)

      {:ok, view, html} = live(conn, ~p"/users/#{user}")

      assert html =~ "María"
      
      Enum.each(posts, fn post ->
        assert has_element?(view, "#post-#{post.id}")
        assert html =~ post.title
      end)
    end
  end

  describe "Form validaciones en tiempo real" do
    test "valida email en tiempo real", %{conn: conn} do
      {:ok, view, _html} = live(conn, ~p"/users/new")

      # Email inválido
      assert view
             |> form("#user-form", user: %{email: "invalido"})
             |> render_change() =~ "tiene un formato inválido"

      # Email válido
      refute view
             |> form("#user-form", user: %{email: "valido@ejemplo.com"})
             |> render_change() =~ "tiene un formato inválido"
    end
  end
end
```

### 3.3 Testing con Floki para Selectores Complejos

```elixir
test "verifica estructura compleja del DOM", %{conn: conn} do
  user = insert(:user)
  {:ok, _view, html} = live(conn, ~p"/users/#{user}")

  {:ok, document} = Floki.parse_document(html)

  # Buscar elementos específicos
  assert [_element] = Floki.find(document, "#user-profile .user-name")
  assert [_element] = Floki.find(document, "button[phx-click='edit']")
  
  # Verificar atributos
  [button] = Floki.find(document, "#edit-button")
  assert Floki.attribute(button, "phx-click") == ["edit"]
  
  # Verificar texto
  assert Floki.find(document, ".user-email") 
         |> Floki.text() 
         |> String.contains?(user.email)
end
```

### 3.4 Testing de Streams en LiveView

```elixir
test "actualiza stream cuando se agrega item", %{conn: conn} do
  {:ok, view, _html} = live(conn, ~p"/messages")

  # Estado inicial vacío
  refute has_element?(view, "#messages [id^='messages-']")

  # Enviar mensaje
  view
  |> form("#message-form", message: %{content: "Hola mundo"})
  |> render_submit()

  # Verificar que aparece en el stream
  assert has_element?(view, "#messages [id^='messages-']")
  assert render(view) =~ "Hola mundo"
end
```

---

## 4. LiveView con PubSub en Múltiples Nodos {#liveview-pubsub-multinodo}

### 4.1 Configuración de PubSub para Multi-Nodo

```elixir
# config/config.exs
config :my_app, MyApp.PubSub,
  name: MyApp.PubSub,
  adapter: Phoenix.PubSub.PG2

# lib/my_app/application.ex
defmodule MyApp.Application do
  use Application

  def start(_type, _args) do
    children = [
      # PubSub para comunicación multi-nodo
      {Phoenix.PubSub, name: MyApp.PubSub},
      MyApp.Repo,
      MyAppWeb.Endpoint
    ]

    opts = [strategy: :one_for_one, name: MyApp.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

### 4.2 Configuración de Nodos (Clustering)

```elixir
# config/runtime.exs
import Config

if config_env() == :prod do
  # Configurar nombre del nodo
  node_name = System.get_env("NODE_NAME") || "myapp"
  node_host = System.get_env("NODE_HOST") || "localhost"
  
  # Formato: nombre@host
  :net_kernel.start([:"#{node_name}@#{node_host}"])
  
  # Cookie para autenticación entre nodos
  Node.set_cookie(:my_secret_cookie)
end
```

### 4.3 LiveView con PubSub - Patrón Completo

```elixir
defmodule MyAppWeb.ChatLive do
  use MyAppWeb, :live_view
  alias MyApp.Chat
  alias Phoenix.PubSub

  @topic "chat:messages"

  def mount(_params, _session, socket) do
    if connected?(socket) do
      # Suscribirse al topic cuando el socket está conectado
      PubSub.subscribe(MyApp.PubSub, @topic)
    end

    messages = Chat.list_recent_messages()

    {:ok,
     socket
     |> assign(:messages_empty?, messages == [])
     |> stream(:messages, messages)}
  end

  def handle_event("send_message", %{"message" => message_params}, socket) do
    case Chat.create_message(message_params) do
      {:ok, message} ->
        # Broadcast a TODOS los nodos suscritos
        PubSub.broadcast(
          MyApp.PubSub,
          @topic,
          {:new_message, message}
        )

        {:noreply,
         socket
         |> put_flash(:info, "Mensaje enviado")
         |> assign(:message_form, to_form(Chat.change_message(%Message{})))}

      {:error, changeset} ->
        {:noreply, assign(socket, message_form: to_form(changeset))}
    end
  end

  # Recibir broadcasts de CUALQUIER nodo
  def handle_info({:new_message, message}, socket) do
    {:noreply,
     socket
     |> assign(:messages_empty?, false)
     |> stream_insert(:messages, message, at: 0)}
  end

  # Manejar mensaje de usuario eliminado
  def handle_info({:message_deleted, message_id}, socket) do
    message = %{id: message_id}
    {:noreply, stream_delete(socket, :messages, message)}
  end
end
```

### 4.4 Contexto con PubSub Broadcasting

```elixir
defmodule MyApp.Chat do
  alias MyApp.Repo
  alias MyApp.Chat.Message
  alias Phoenix.PubSub

  @topic "chat:messages"

  def create_message(attrs) do
    %Message{}
    |> Message.changeset(attrs)
    |> Repo.insert()
    |> case do
      {:ok, message} = result ->
        # Broadcast automático después de crear
        broadcast({:new_message, message})
        result

      error ->
        error
    end
  end

  def delete_message(%Message{} = message) do
    Repo.delete(message)
    |> case do
      {:ok, deleted_message} = result ->
        broadcast({:message_deleted, deleted_message.id})
        result

      error ->
        error
    end
  end

  def update_message(%Message{} = message, attrs) do
    message
    |> Message.changeset(attrs)
    |> Repo.update()
    |> case do
      {:ok, updated_message} = result ->
        broadcast({:message_updated, updated_message})
        result

      error ->
        error
    end
  end

  # Función privada para broadcast
  defp broadcast(message) do
    PubSub.broadcast(MyApp.PubSub, @topic, message)
  end

  def subscribe do
    PubSub.subscribe(MyApp.PubSub, @topic)
  end
end
```

### 4.5 Testing de PubSub

```elixir
defmodule MyApp.ChatTest do
  use MyApp.DataCase
  import MyApp.Factory

  alias MyApp.Chat
  alias Phoenix.PubSub

  describe "PubSub broadcasting" do
    test "crear mensaje hace broadcast" do
      # Suscribirse al topic
      :ok = Chat.subscribe()

      user = insert(:user)
      attrs = %{content: "Hola!", user_id: user.id}

      {:ok, message} = Chat.create_message(attrs)

      # Verificar que recibimos el broadcast
      assert_receive {:new_message, ^message}
    end

    test "eliminar mensaje hace broadcast del ID" do
      :ok = Chat.subscribe()

      message = insert(:message)
      {:ok, _} = Chat.delete_message(message)

      assert_receive {:message_deleted, message_id}
      assert message_id == message.id
    end
  end
end
```

### 4.6 Conectar Nodos Manualmente (para desarrollo/testing)

```elixir
# En iex de nodo 1
iex --name node1@localhost -S mix phx.server

# En iex de nodo 2  
iex --name node2@localhost -S mix phx.server

# En node2, conectar a node1
Node.connect(:"node1@localhost")

# Verificar conexión
Node.list()
# => [:"node1@localhost"]

# Verificar PubSub funciona entre nodos
Phoenix.PubSub.subscribe(MyApp.PubSub, "test")
Phoenix.PubSub.broadcast(MyApp.PubSub, "test", {:mensaje, "hola desde node2"})
# En node1 deberías recibir el mensaje
```

---

## 5. Consumo de Servicios RPC/EPMD y Refresco de Vistas {#rpc-refresco-vistas}

### 5.1 Conceptos de RPC en Elixir

EPMD (Erlang Port Mapper Daemon) permite que nodos Erlang/Elixir se descubran y comuniquen.

### 5.2 Llamadas RPC Básicas

```elixir
defmodule MyApp.RemoteService do
  @doc """
  Llama a una función en un nodo remoto
  """
  def call_remote(node, module, function, args) do
    case :rpc.call(node, module, function, args) do
      {:badrpc, reason} ->
        {:error, reason}

      result ->
        {:ok, result}
    end
  end

  @doc """
  Obtiene datos de usuarios desde nodo remoto
  """
  def get_remote_users(remote_node) do
    call_remote(
      remote_node,
      MyApp.Accounts,
      :list_users,
      []
    )
  end
end
```

### 5.3 GenServer para Sincronización con Nodo Remoto

```elixir
defmodule MyApp.RemoteSync do
  use GenServer
  alias Phoenix.PubSub
  require Logger

  @sync_interval :timer.seconds(30)

  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def init(opts) do
    remote_node = Keyword.get(opts, :remote_node)
    
    # Programar primera sincronización
    schedule_sync()

    {:ok, %{remote_node: remote_node, last_sync: nil}}
  end

  def handle_info(:sync, state) do
    Logger.info("Sincronizando con nodo remoto: #{state.remote_node}")

    case sync_data(state.remote_node) do
      {:ok, data} ->
        # Broadcast de datos actualizados
        PubSub.broadcast(
          MyApp.PubSub,
          "remote:sync",
          {:data_updated, data}
        )

        schedule_sync()
        {:noreply, %{state | last_sync: DateTime.utc_now()}}

      {:error, reason} ->
        Logger.error("Error en sincronización: #{inspect(reason)}")
        schedule_sync()
        {:noreply, state}
    end
  end

  defp sync_data(remote_node) do
    case :rpc.call(remote_node, MyApp.DataService, :get_latest_data, []) do
      {:badrpc, reason} ->
        {:error, reason}

      data ->
        # Guardar datos localmente
        MyApp.Cache.put(:remote_data, data)
        {:ok, data}
    end
  end

  defp schedule_sync do
    Process.send_after(self(), :sync, @sync_interval)
  end
end
```

### 5.4 LiveView que Refresca con Datos RPC

```elixir
defmodule MyAppWeb.DashboardLive do
  use MyAppWeb, :live_view
  alias Phoenix.PubSub

  def mount(_params, _session, socket) do
    if connected?(socket) do
      # Suscribirse a actualizaciones remotas
      PubSub.subscribe(MyApp.PubSub, "remote:sync")
    end

    # Obtener datos iniciales (posiblemente del cache)
    remote_data = MyApp.Cache.get(:remote_data, [])

    {:ok,
     socket
     |> assign(:remote_data, remote_data)
     |> assign(:last_update, nil)
     |> assign(:sync_status, :idle)}
  end

  # Refrescar cuando llegan datos del nodo remoto
  def handle_info({:data_updated, data}, socket) do
    {:noreply,
     socket
     |> assign(:remote_data, data)
     |> assign(:last_update, DateTime.utc_now())
     |> put_flash(:info, "Datos actualizados desde servidor remoto")}
  end

  # Permitir refrescar manualmente
  def handle_event("refresh", _params, socket) do
    send(self(), :manual_refresh)
    {:noreply, assign(socket, :sync_status, :syncing)}
  end

  def handle_info(:manual_refresh, socket) do
    remote_node = Application.get_env(:my_app, :remote_node)

    case MyApp.RemoteService.get_remote_users(remote_node) do
      {:ok, users} ->
        {:noreply,
         socket
         |> assign(:remote_data, users)
         |> assign(:sync_status, :idle)
         |> put_flash(:info, "Datos actualizados")}

      {:error, reason} ->
        {:noreply,
         socket
         |> assign(:sync_status, :error)
         |> put_flash(:error, "Error al sincronizar: #{inspect(reason)}")}
    end
  end
end
```

### 5.5 Template para Dashboard con Datos Remotos

```heex
<Layouts.app flash={@flash}>
  <div class="max-w-7xl mx-auto">
    <div class="flex justify-between items-center mb-8">
      <h1 class="text-3xl font-bold">Dashboard - Datos Remotos</h1>
      
      <button
        phx-click="refresh"
        class="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700"
        disabled={@sync_status == :syncing}
      >
        <%= if @sync_status == :syncing do %>
          Sincronizando...
        <% else %>
          Refrescar
        <% end %>
      </button>
    </div>

    <%= if @last_update do %>
      <p class="text-sm text-gray-600 mb-4">
        Última actualización: <%= Calendar.strftime(@last_update, "%H:%M:%S") %>
      </p>
    <% end %>

    <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
      <%= for item <- @remote_data do %>
        <div class="bg-white p-6 rounded-lg shadow">
          <h3 class="font-bold text-lg mb-2"><%= item.name %></h3>
          <p class="text-gray-600"><%= item.description %></p>
        </div>
      <% end %>
    </div>
  </div>
</Layouts.app>
```

### 5.6 Configuración de Nodos Remotos

```elixir
# config/runtime.exs
import Config

if config_env() == :prod do
  # Nodo remoto para RPC
  config :my_app,
    remote_node: System.get_env("REMOTE_NODE") || :"remote@hostname"
end

# lib/my_app/application.ex
defmodule MyApp.Application do
  use Application

  def start(_type, _args) do
    children = [
      MyApp.Repo,
      {Phoenix.PubSub, name: MyApp.PubSub},
      MyAppWeb.Endpoint,
      # Sincronización con nodo remoto
      {MyApp.RemoteSync, remote_node: remote_node()}
    ]

    opts = [strategy: :one_for_one, name: MyApp.Supervisor]
    Supervisor.start_link(children, opts)
  end

  defp remote_node do
    Application.get_env(:my_app, :remote_node, :nonode@nohost)
  end
end
```

---

## 6. Patrones de Refresco de Vistas HTML {#patrones-refresco}

### 6.1 Refresco Automático con Streams

**Mejor práctica**: Usar streams para colecciones grandes.

```elixir
defmodule MyAppWeb.TimelineLive do
  use MyAppWeb, :live_view
  alias Phoenix.PubSub

  def mount(_params, _session, socket) do
    if connected?(socket) do
      PubSub.subscribe(MyApp.PubSub, "timeline:posts")
    end

    posts = MyApp.Social.list_timeline_posts()

    {:ok,
     socket
     |> assign(:posts_empty?, posts == [])
     |> stream(:posts, posts)}
  end

  # Nuevo post - insertar al inicio del stream
  def handle_info({:new_post, post}, socket) do
    {:noreply,
     socket
     |> assign(:posts_empty?, false)
     |> stream_insert(:posts, post, at: 0)}
  end

  # Post actualizado - reemplazar en stream
  def handle_info({:post_updated, post}, socket) do
    {:noreply, stream_insert(socket, :posts, post)}
  end

  # Post eliminado - remover del stream
  def handle_info({:post_deleted, post_id}, socket) do
    post = %{id: post_id}
    {:noreply, stream_delete(socket, :posts, post)}
  end
end
```

### 6.2 Template para Streams

```heex
<Layouts.app flash={@flash}>
  <div class="max-w-3xl mx-auto">
    <h1 class="text-3xl font-bold mb-8">Timeline</h1>

    <%= if @posts_empty? do %>
      <p class="text-gray-500 text-center py-12">
        No hay posts todavía
      </p>
    <% else %>
      <div id="posts" phx-update="stream" class="space-y-6">
        <%= for {id, post} <- @streams.posts do %>
          <div id={id} class="bg-white p-6 rounded-lg shadow">
            <div class="flex items-start justify-between">
              <div>
                <h3 class="font-bold text-lg"><%= post.title %></h3>
                <p class="text-sm text-gray-600 mt-1">
                  Por <%= post.user.name %> · 
                  <%= Calendar.strftime(post.inserted_at, "%d/%m/%Y %H:%M") %>
                </p>
              </div>
              
              <button 
                phx-click="delete_post" 
                phx-value-id={post.id}
                class="text-red-600 hover:text-red-800"
              >
                Eliminar
              </button>
            </div>
            
            <p class="mt-4 text-gray-800"><%= post.body %></p>
          </div>
        <% end %>
      </div>
    <% end %>
  </div>
</Layouts.app>
```

### 6.3 Refresco con Asignaciones Simples

Para datos pequeños, usar `assign/2`:

```elixir
defmodule MyAppWeb.CounterLive do
  use MyAppWeb, :live_view
  alias Phoenix.PubSub

  def mount(_params, _session, socket) do
    if connected?(socket) do
      PubSub.subscribe(MyApp.PubSub, "counter")
    end

    {:ok, assign(socket, :count, get_current_count())}
  end

  def handle_info({:count_updated, new_count}, socket) do
    {:noreply, assign(socket, :count, new_count)}
  end

  def handle_event("increment", _params, socket) do
    new_count = socket.assigns.count + 1
    
    PubSub.broadcast(MyApp.PubSub, "counter", {:count_updated, new_count})
    
    {:noreply, assign(socket, :count, new_count)}
  end

  defp get_current_count do
    MyApp.Counter.get_count()
  end
end
```

### 6.4 Refresco con JS Hooks para Scroll

```elixir
# assets/js/app.js
let Hooks = {}

Hooks.InfiniteScroll = {
  mounted() {
    this.pending = false
    
    this.observer = new IntersectionObserver(entries => {
      const target = entries[0]
      if (target.isIntersecting && !this.pending) {
        this.pending = true
        this.pushEvent("load-more", {}, () => {
          this.pending = false
        })
      }
    })
    
    this.observer.observe(this.el)
  },
  
  destroyed() {
    this.observer.disconnect()
  }
}

let liveSocket = new LiveSocket("/live", Socket, {
  params: {_csrf_token: csrfToken},
  hooks: Hooks
})
```

```elixir
# LiveView con scroll infinito
defmodule MyAppWeb.FeedLive do
  use MyAppWeb, :live_view

  @per_page 20

  def mount(_params, _session, socket) do
    {:ok,
     socket
     |> assign(:page, 1)
     |> stream(:posts, load_posts(1))}
  end

  def handle_event("load-more", _params, socket) do
    next_page = socket.assigns.page + 1
    posts = load_posts(next_page)

    {:noreply,
     socket
     |> assign(:page, next_page)
     |> stream(:posts, posts)}
  end

  defp load_posts(page) do
    offset = (page - 1) * @per_page
    MyApp.Social.list_posts(limit: @per_page, offset: offset)
  end
end
```

```heex
<div id="posts" phx-update="stream">
  <%= for {id, post} <- @streams.posts do %>
    <div id={id}><%= post.title %></div>
  <% end %>
</div>

<!-- Trigger para infinite scroll -->
<div id="infinite-scroll-marker" phx-hook="InfiniteScroll"></div>
```

### 6.5 Optimización: Actualizar Solo lo Necesario

```elixir
defmodule MyAppWeb.ProductLive do
  use MyAppWeb, :live_view

  def mount(%{"id" => id}, _session, socket) do
    if connected?(socket) do
      PubSub.subscribe(MyApp.PubSub, "product:#{id}")
    end

    product = MyApp.Catalog.get_product!(id)

    {:ok,
     socket
     |> assign(:product, product)
     |> assign(:stock, product.stock)
     |> assign(:price, product.price)}
  end

  # Solo actualizar stock (no re-renderizar todo el producto)
  def handle_info({:stock_updated, new_stock}, socket) do
    {:noreply, assign(socket, :stock, new_stock)}
  end

  # Solo actualizar precio
  def handle_info({:price_updated, new_price}, socket) do
    {:noreply, assign(socket, :price, new_price)}
  end
end
```

```heex
<Layouts.app flash={@flash}>
  <div class="max-w-4xl mx-auto">
    <h1 class="text-3xl font-bold"><%= @product.name %></h1>
    
    <!-- Solo esta parte se re-renderiza cuando cambia @price -->
    <p class="text-2xl font-bold text-green-600 mt-4">
      $<%= @price %>
    </p>
    
    <!-- Solo esta parte se re-renderiza cuando cambia @stock -->
    <p class="text-sm text-gray-600 mt-2">
      Stock disponible: <%= @stock %> unidades
    </p>
    
    <!-- Esto no se re-renderiza a menos que @product cambie -->
    <div class="mt-6">
      <p class="text-gray-800"><%= @product.description %></p>
    </div>
  </div>
</Layouts.app>
```

---

## Resumen de Mejores Prácticas

### Validaciones
✅ SIEMPRE validar en el servidor con `Ecto.Changeset`
✅ Usar `phx-change` para validaciones en tiempo real
✅ Usar `to_form/2` en LiveView, `<.form>` y `<.input>` en templates
✅ Nunca acceder changeset directamente en templates

### Testing
✅ Usar ExMachina para factories con relaciones FK
✅ Usar `Phoenix.LiveViewTest` para LiveViews
✅ Usar `element/2` y `has_element?/2` en lugar de HTML crudo
✅ Dar IDs únicos a elementos clave para testing
✅ Verificar asociaciones precargadas en tests

### PubSub Multi-Nodo
✅ Usar `Phoenix.PubSub.PG2` para distribuir mensajes
✅ Suscribirse en `mount/3` cuando `connected?(socket)`
✅ Hacer broadcast después de operaciones exitosas
✅ Configurar nombres de nodo y cookies para clustering

### RPC y Sincronización
✅ Usar `:rpc.call/4` para llamadas remotas
✅ Implementar GenServer para polling periódico
✅ Usar PubSub para notificar actualizaciones
✅ Manejar errores de conexión gracefully

### Refresco de Vistas
✅ Usar streams para colecciones grandes
✅ Usar assigns simples para datos pequeños
✅ Implementar JS Hooks para scroll infinito
✅ Actualizar solo lo necesario (granularidad)

---

## Recursos Adicionales

- [Phoenix LiveView Docs](https://hexdocs.pm/phoenix_live_view)
- [Ecto Changeset Docs](https://hexdocs.pm/ecto/Ecto.Changeset.html)
- [Phoenix PubSub Docs](https://hexdocs.pm/phoenix_pubsub)
- [Distributed Elixir](https://elixir-lang.org/getting-started/mix-otp/distributed-tasks.html)
# Guía Completa de Trabajo con Phoenix Framework

Esta guía cubre las mejores prácticas para trabajar con Phoenix, incluyendo validaciones, testing, LiveView con PubSub multi-nodo, y consumo de servicios RPC.

---

## Tabla de Contenidos

1. [Validaciones en HTML y LiveView](#validaciones)
2. [Testing con Datos Relacionados (FK)](#testing-datos-relacionados)
3. [Testing de HTML Estático vs LiveView](#testing-html)
4. [LiveView con PubSub en Múltiples Nodos](#liveview-pubsub-multinodo)
5. [Consumo de Servicios RPC/EPMD y Refresco de Vistas](#rpc-refresco-vistas)
6. [Patrones de Refresco de Vistas HTML](#patrones-refresco)

---

## 1. Validaciones en HTML y LiveView {#validaciones}

### 1.1 Validaciones del Lado del Servidor con Ecto.Changeset

**Regla fundamental**: TODAS las validaciones DEBEN ocurrir en el servidor mediante `Ecto.Changeset`.

```elixir
defmodule MyApp.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset

  schema "users" do
    field :email, :string
    field :name, :string
    field :age, :integer
    timestamps()
  end

  def changeset(user, attrs) do
    user
    |> cast(attrs, [:email, :name, :age])
    |> validate_required([:email, :name])
    |> validate_format(:email, ~r/@/)
    |> validate_length(:name, min: 2, max: 100)
    |> validate_number(:age, greater_than: 0, less_than: 150)
    |> unique_constraint(:email)
  end
end
```

### 1.2 Validaciones en Tiempo Real con LiveView

**Patrón correcto**: Usar `phx-change` para validaciones mientras el usuario escribe.

```elixir
defmodule MyAppWeb.UserLive.Form do
  use MyAppWeb, :live_view

  def mount(_params, _session, socket) do
    changeset = Accounts.change_user(%User{})
    
    {:ok,
     socket
     |> assign(:form, to_form(changeset))
     |> assign(:user, nil)}
  end

  # Validación en tiempo real mientras el usuario escribe
  def handle_event("validate", %{"user" => user_params}, socket) do
    changeset =
      %User{}
      |> Accounts.change_user(user_params)
      |> Map.put(:action, :validate)

    {:noreply, assign(socket, form: to_form(changeset))}
  end

  # Guardado final con validación completa
  def handle_event("save", %{"user" => user_params}, socket) do
    case Accounts.create_user(user_params) do
      {:ok, user} ->
        {:noreply,
         socket
         |> put_flash(:info, "Usuario creado exitosamente")
         |> push_navigate(to: ~p"/users/#{user}")}

      {:error, %Ecto.Changeset{} = changeset} ->
        {:noreply, assign(socket, form: to_form(changeset))}
    end
  end
end
```

### 1.3 Template HTML para Validaciones

**Reglas importantes**:
- SIEMPRE usar `to_form/2` en el LiveView
- SIEMPRE usar `<.form>` y `<.input>` en la template
- NUNCA acceder al changeset directamente en la template
- SIEMPRE dar un ID único al form

```heex
<Layouts.app flash={@flash}>
  <div class="mx-auto max-w-2xl">
    <h1 class="text-3xl font-bold mb-8">Nuevo Usuario</h1>

    <.form 
      for={@form} 
      id="user-form" 
      phx-change="validate" 
      phx-submit="save"
      class="space-y-6"
    >
      <.input 
        field={@form[:name]} 
        type="text" 
        label="Nombre completo"
        placeholder="Juan Pérez"
        required
      />

      <.input 
        field={@form[:email]} 
        type="email" 
        label="Email"
        placeholder="juan@ejemplo.com"
        required
      />

      <.input 
        field={@form[:age]} 
        type="number" 
        label="Edad"
        min="1"
        max="150"
      />

      <button 
        type="submit"
        class="w-full px-6 py-3 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition"
      >
        Guardar Usuario
      </button>
    </.form>
  </div>
</Layouts.app>
```

### 1.4 Validaciones Personalizadas

```elixir
defmodule MyApp.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset

  # ... schema ...

  def changeset(user, attrs) do
    user
    |> cast(attrs, [:email, :name, :birth_date])
    |> validate_required([:email, :name])
    |> validate_adult()
    |> validate_email_domain()
  end

  defp validate_adult(changeset) do
    validate_change(changeset, :birth_date, fn :birth_date, birth_date ->
      today = Date.utc_today()
      age = Date.diff(today, birth_date) / 365.25

      if age >= 18 do
        []
      else
        [birth_date: "debes ser mayor de 18 años"]
      end
    end)
  end

  defp validate_email_domain(changeset) do
    validate_change(changeset, :email, fn :email, email ->
      allowed_domains = ["empresa.com", "ejemplo.com"]
      [_name, domain] = String.split(email, "@")

      if domain in allowed_domains do
        []
      else
        [email: "el dominio debe ser #{Enum.join(allowed_domains, " o ")}"]
      end
    end)
  end
end
```

---

## 2. Testing con Datos Relacionados (FK) {#testing-datos-relacionados}

### 2.1 Configuración de ExMachina para Factories

Primero, agregar la dependencia en `mix.exs`:

```elixir
defp deps do
  [
    {:ex_machina, "~> 2.7", only: :test}
  ]
end
```

### 2.2 Crear Factory con Relaciones

```elixir
# test/support/factory.ex
defmodule MyApp.Factory do
  use ExMachina.Ecto, repo: MyApp.Repo

  def user_factory do
    %MyApp.Accounts.User{
      name: sequence(:name, &"Usuario #{&1}"),
      email: sequence(:email, &"user#{&1}@ejemplo.com"),
      age: 25
    }
  end

  def post_factory do
    %MyApp.Blog.Post{
      title: sequence(:title, &"Post #{&1}"),
      body: "Contenido del post",
      published: false,
      # Relación FK - crea automáticamente un user
      user: build(:user)
    }
  end

  def comment_factory do
    %MyApp.Blog.Comment{
      body: "Este es un comentario",
      # Relaciones FK múltiples
      user: build(:user),
      post: build(:post)
    }
  end

  def category_factory do
    %MyApp.Blog.Category{
      name: sequence(:name, &"Categoría #{&1}")
    }
  end

  # Factory con relación many_to_many
  def post_with_categories_factory do
    %MyApp.Blog.Post{
      title: "Post con categorías",
      body: "Contenido",
      user: build(:user),
      categories: build_list(3, :category)
    }
  end
end
```

### 2.3 Testing de Contextos con Datos Relacionados

```elixir
# test/my_app/blog_test.exs
defmodule MyApp.BlogTest do
  use MyApp.DataCase
  import MyApp.Factory

  alias MyApp.Blog

  describe "posts" do
    test "create_post/1 crea un post asociado a un usuario" do
      user = insert(:user)
      
      attrs = %{
        title: "Mi primer post",
        body: "Contenido del post",
        user_id: user.id
      }

      assert {:ok, post} = Blog.create_post(attrs)
      assert post.user_id == user.id
      assert post.title == "Mi primer post"
    end

    test "create_post/1 falla sin user_id válido" do
      attrs = %{
        title: "Post sin usuario",
        body: "Contenido"
      }

      assert {:error, changeset} = Blog.create_post(attrs)
      assert "can't be blank" in errors_on(changeset).user_id
    end

    test "list_posts_with_user/0 precarga el usuario" do
      user = insert(:user, name: "Juan")
      insert(:post, user: user, title: "Post de Juan")
      insert(:post, user: user, title: "Otro post de Juan")

      posts = Blog.list_posts_with_user()

      assert length(posts) == 2
      assert Enum.all?(posts, fn post -> 
        Ecto.assoc_loaded?(post.user) 
      end)
      assert Enum.all?(posts, fn post -> 
        post.user.name == "Juan" 
      end)
    end
  end

  describe "comments con relaciones encadenadas" do
    test "crear comentario requiere user y post válidos" do
      user = insert(:user)
      post = insert(:post)

      attrs = %{
        body: "Gran post!",
        user_id: user.id,
        post_id: post.id
      }

      assert {:ok, comment} = Blog.create_comment(attrs)
      assert comment.user_id == user.id
      assert comment.post_id == post.id
    end

    test "list_comments_with_associations/0 precarga todo" do
      # Crear datos relacionados
      user1 = insert(:user, name: "María")
      user2 = insert(:user, name: "Pedro")
      post = insert(:post, user: user1, title: "Post de María")
      
      insert(:comment, user: user2, post: post, body: "Comentario de Pedro")
      insert(:comment, user: user1, post: post, body: "Respuesta de María")

      comments = Blog.list_comments_with_associations()

      assert length(comments) == 2
      
      # Verificar que las asociaciones están precargadas
      Enum.each(comments, fn comment ->
        assert Ecto.assoc_loaded?(comment.user)
        assert Ecto.assoc_loaded?(comment.post)
        assert Ecto.assoc_loaded?(comment.post.user)
      end)
    end

    test "eliminar post elimina comentarios en cascada" do
      post = insert(:post)
      insert_list(3, :comment, post: post)

      assert Repo.aggregate(Blog.Comment, :count) == 3

      Blog.delete_post(post)

      assert Repo.aggregate(Blog.Comment, :count) == 0
    end
  end

  describe "relaciones many_to_many" do
    test "asociar categorías a un post" do
      post = insert(:post)
      categories = insert_list(3, :category)
      category_ids = Enum.map(categories, & &1.id)

      {:ok, updated_post} = Blog.update_post_categories(post, category_ids)

      post_with_categories = Blog.get_post_with_categories!(updated_post.id)
      assert length(post_with_categories.categories) == 3
    end
  end
end
```

### 2.4 Helpers para Testing

```elixir
# test/support/data_case.ex
defmodule MyApp.DataCase do
  use ExUnit.CaseTemplate

  using do
    quote do
      alias MyApp.Repo
      import Ecto
      import Ecto.Changeset
      import Ecto.Query
      import MyApp.DataCase
      import MyApp.Factory
    end
  end

  setup tags do
    MyApp.DataCase.setup_sandbox(tags)
    :ok
  end

  def setup_sandbox(tags) do
    pid = Ecto.Adapters.SQL.Sandbox.start_owner!(MyApp.Repo, shared: not tags[:async])
    on_exit(fn -> Ecto.Adapters.SQL.Sandbox.stop_owner(pid) end)
  end

  def errors_on(changeset) do
    Ecto.Changeset.traverse_errors(changeset, fn {message, opts} ->
      Regex.replace(~r"%{(\w+)}", message, fn _, key ->
        opts |> Keyword.get(String.to_existing_atom(key), key) |> to_string()
      end)
    end)
  end
end
```

---

## 3. Testing de HTML Estático vs LiveView {#testing-html}

### 3.1 Testing de Controladores Estáticos

```elixir
# test/my_app_web/controllers/page_controller_test.exs
defmodule MyAppWeb.PageControllerTest do
  use MyAppWeb.ConnCase
  import MyApp.Factory

  describe "GET /" do
    test "muestra la página de inicio", %{conn: conn} do
      conn = get(conn, ~p"/")
      
      assert html_response(conn, 200)
      assert html = html_response(conn, 200)
      assert html =~ "Bienvenido"
    end
  end

  describe "GET /about" do
    test "muestra la página sobre nosotros", %{conn: conn} do
      conn = get(conn, ~p"/about")
      
      assert html_response(conn, 200) =~ "Sobre Nosotros"
    end
  end
end
```

### 3.2 Testing de LiveView

**Reglas importantes**:
- Usar `Phoenix.LiveViewTest`
- Usar `element/2` y `has_element?/2` en lugar de probar HTML crudo
- Usar IDs únicos en elementos clave
- Usar `Floki` para selectores complejos

```elixir
# test/my_app_web/live/user_live_test.exs
defmodule MyAppWeb.UserLiveTest do
  use MyAppWeb.ConnCase
  import Phoenix.LiveViewTest
  import MyApp.Factory

  describe "Index" do
    test "muestra todos los usuarios", %{conn: conn} do
      users = insert_list(3, :user)

      {:ok, view, html} = live(conn, ~p"/users")

      assert html =~ "Listado de Usuarios"
      
      # Verificar que cada usuario aparece
      Enum.each(users, fn user ->
        assert has_element?(view, "#user-#{user.id}")
        assert html =~ user.name
      end)
    end

    test "permite crear un nuevo usuario", %{conn: conn} do
      {:ok, view, _html} = live(conn, ~p"/users")

      # Click en botón nuevo
      assert view |> element("a", "Nuevo Usuario") |> render_click()

      # Llenar formulario
      assert view
             |> form("#user-form", user: %{name: "", email: ""})
             |> render_change() =~ "can&#39;t be blank"

      # Submit válido
      assert view
             |> form("#user-form", user: %{
               name: "Juan Pérez",
               email: "juan@ejemplo.com",
               age: 30
             })
             |> render_submit()

      # Verificar flash y redirección
      assert_redirected(view, ~p"/users/#{user_id}")
      
      # Verificar que se creó en BD
      user = Repo.get_by!(User, email: "juan@ejemplo.com")
      assert user.name == "Juan Pérez"
    end

    test "permite eliminar un usuario", %{conn: conn} do
      user = insert(:user)

      {:ok, view, _html} = live(conn, ~p"/users")

      # Click en eliminar
      assert view
             |> element("#user-#{user.id} button[phx-click='delete']")
             |> render_click()

      # Verificar que no existe más
      refute has_element?(view, "#user-#{user.id}")
      assert Repo.get(User, user.id) == nil
    end
  end

  describe "Show" do
    test "muestra un usuario con sus posts", %{conn: conn} do
      user = insert(:user, name: "María")
      posts = insert_list(3, :post, user: user)

      {:ok, view, html} = live(conn, ~p"/users/#{user}")

      assert html =~ "María"
      
      Enum.each(posts, fn post ->
        assert has_element?(view, "#post-#{post.id}")
        assert html =~ post.title
      end)
    end
  end

  describe "Form validaciones en tiempo real" do
    test "valida email en tiempo real", %{conn: conn} do
      {:ok, view, _html} = live(conn, ~p"/users/new")

      # Email inválido
      assert view
             |> form("#user-form", user: %{email: "invalido"})
             |> render_change() =~ "tiene un formato inválido"

      # Email válido
      refute view
             |> form("#user-form", user: %{email: "valido@ejemplo.com"})
             |> render_change() =~ "tiene un formato inválido"
    end
  end
end
```

### 3.3 Testing con Floki para Selectores Complejos

```elixir
test "verifica estructura compleja del DOM", %{conn: conn} do
  user = insert(:user)
  {:ok, _view, html} = live(conn, ~p"/users/#{user}")

  {:ok, document} = Floki.parse_document(html)

  # Buscar elementos específicos
  assert [_element] = Floki.find(document, "#user-profile .user-name")
  assert [_element] = Floki.find(document, "button[phx-click='edit']")
  
  # Verificar atributos
  [button] = Floki.find(document, "#edit-button")
  assert Floki.attribute(button, "phx-click") == ["edit"]
  
  # Verificar texto
  assert Floki.find(document, ".user-email") 
         |> Floki.text() 
         |> String.contains?(user.email)
end
```

### 3.4 Testing de Streams en LiveView

```elixir
test "actualiza stream cuando se agrega item", %{conn: conn} do
  {:ok, view, _html} = live(conn, ~p"/messages")

  # Estado inicial vacío
  refute has_element?(view, "#messages [id^='messages-']")

  # Enviar mensaje
  view
  |> form("#message-form", message: %{content: "Hola mundo"})
  |> render_submit()

  # Verificar que aparece en el stream
  assert has_element?(view, "#messages [id^='messages-']")
  assert render(view) =~ "Hola mundo"
end
```

---

## 4. LiveView con PubSub en Múltiples Nodos {#liveview-pubsub-multinodo}

### 4.1 Configuración de PubSub para Multi-Nodo

```elixir
# config/config.exs
config :my_app, MyApp.PubSub,
  name: MyApp.PubSub,
  adapter: Phoenix.PubSub.PG2

# lib/my_app/application.ex
defmodule MyApp.Application do
  use Application

  def start(_type, _args) do
    children = [
      # PubSub para comunicación multi-nodo
      {Phoenix.PubSub, name: MyApp.PubSub},
      MyApp.Repo,
      MyAppWeb.Endpoint
    ]

    opts = [strategy: :one_for_one, name: MyApp.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

### 4.2 Configuración de Nodos (Clustering)

```elixir
# config/runtime.exs
import Config

if config_env() == :prod do
  # Configurar nombre del nodo
  node_name = System.get_env("NODE_NAME") || "myapp"
  node_host = System.get_env("NODE_HOST") || "localhost"
  
  # Formato: nombre@host
  :net_kernel.start([:"#{node_name}@#{node_host}"])
  
  # Cookie para autenticación entre nodos
  Node.set_cookie(:my_secret_cookie)
end
```

### 4.3 LiveView con PubSub - Patrón Completo

```elixir
defmodule MyAppWeb.ChatLive do
  use MyAppWeb, :live_view
  alias MyApp.Chat
  alias Phoenix.PubSub

  @topic "chat:messages"

  def mount(_params, _session, socket) do
    if connected?(socket) do
      # Suscribirse al topic cuando el socket está conectado
      PubSub.subscribe(MyApp.PubSub, @topic)
    end

    messages = Chat.list_recent_messages()

    {:ok,
     socket
     |> assign(:messages_empty?, messages == [])
     |> stream(:messages, messages)}
  end

  def handle_event("send_message", %{"message" => message_params}, socket) do
    case Chat.create_message(message_params) do
      {:ok, message} ->
        # Broadcast a TODOS los nodos suscritos
        PubSub.broadcast(
          MyApp.PubSub,
          @topic,
          {:new_message, message}
        )

        {:noreply,
         socket
         |> put_flash(:info, "Mensaje enviado")
         |> assign(:message_form, to_form(Chat.change_message(%Message{})))}

      {:error, changeset} ->
        {:noreply, assign(socket, message_form: to_form(changeset))}
    end
  end

  # Recibir broadcasts de CUALQUIER nodo
  def handle_info({:new_message, message}, socket) do
    {:noreply,
     socket
     |> assign(:messages_empty?, false)
     |> stream_insert(:messages, message, at: 0)}
  end

  # Manejar mensaje de usuario eliminado
  def handle_info({:message_deleted, message_id}, socket) do
    message = %{id: message_id}
    {:noreply, stream_delete(socket, :messages, message)}
  end
end
```

### 4.4 Contexto con PubSub Broadcasting

```elixir
defmodule MyApp.Chat do
  alias MyApp.Repo
  alias MyApp.Chat.Message
  alias Phoenix.PubSub

  @topic "chat:messages"

  def create_message(attrs) do
    %Message{}
    |> Message.changeset(attrs)
    |> Repo.insert()
    |> case do
      {:ok, message} = result ->
        # Broadcast automático después de crear
        broadcast({:new_message, message})
        result

      error ->
        error
    end
  end

  def delete_message(%Message{} = message) do
    Repo.delete(message)
    |> case do
      {:ok, deleted_message} = result ->
        broadcast({:message_deleted, deleted_message.id})
        result

      error ->
        error
    end
  end

  def update_message(%Message{} = message, attrs) do
    message
    |> Message.changeset(attrs)
    |> Repo.update()
    |> case do
      {:ok, updated_message} = result ->
        broadcast({:message_updated, updated_message})
        result

      error ->
        error
    end
  end

  # Función privada para broadcast
  defp broadcast(message) do
    PubSub.broadcast(MyApp.PubSub, @topic, message)
  end

  def subscribe do
    PubSub.subscribe(MyApp.PubSub, @topic)
  end
end
```

### 4.5 Testing de PubSub

```elixir
defmodule MyApp.ChatTest do
  use MyApp.DataCase
  import MyApp.Factory

  alias MyApp.Chat
  alias Phoenix.PubSub

  describe "PubSub broadcasting" do
    test "crear mensaje hace broadcast" do
      # Suscribirse al topic
      :ok = Chat.subscribe()

      user = insert(:user)
      attrs = %{content: "Hola!", user_id: user.id}

      {:ok, message} = Chat.create_message(attrs)

      # Verificar que recibimos el broadcast
      assert_receive {:new_message, ^message}
    end

    test "eliminar mensaje hace broadcast del ID" do
      :ok = Chat.subscribe()

      message = insert(:message)
      {:ok, _} = Chat.delete_message(message)

      assert_receive {:message_deleted, message_id}
      assert message_id == message.id
    end
  end
end
```

### 4.6 Conectar Nodos Manualmente (para desarrollo/testing)

```elixir
# En iex de nodo 1
iex --name node1@localhost -S mix phx.server

# En iex de nodo 2  
iex --name node2@localhost -S mix phx.server

# En node2, conectar a node1
Node.connect(:"node1@localhost")

# Verificar conexión
Node.list()
# => [:"node1@localhost"]

# Verificar PubSub funciona entre nodos
Phoenix.PubSub.subscribe(MyApp.PubSub, "test")
Phoenix.PubSub.broadcast(MyApp.PubSub, "test", {:mensaje, "hola desde node2"})
# En node1 deberías recibir el mensaje
```

---

## 5. Consumo de Servicios RPC/EPMD y Refresco de Vistas {#rpc-refresco-vistas}

### 5.1 Conceptos de RPC en Elixir

EPMD (Erlang Port Mapper Daemon) permite que nodos Erlang/Elixir se descubran y comuniquen.

### 5.2 Llamadas RPC Básicas

```elixir
defmodule MyApp.RemoteService do
  @doc """
  Llama a una función en un nodo remoto
  """
  def call_remote(node, module, function, args) do
    case :rpc.call(node, module, function, args) do
      {:badrpc, reason} ->
        {:error, reason}

      result ->
        {:ok, result}
    end
  end

  @doc """
  Obtiene datos de usuarios desde nodo remoto
  """
  def get_remote_users(remote_node) do
    call_remote(
      remote_node,
      MyApp.Accounts,
      :list_users,
      []
    )
  end
end
```

### 5.3 GenServer para Sincronización con Nodo Remoto

```elixir
defmodule MyApp.RemoteSync do
  use GenServer
  alias Phoenix.PubSub
  require Logger

  @sync_interval :timer.seconds(30)

  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def init(opts) do
    remote_node = Keyword.get(opts, :remote_node)
    
    # Programar primera sincronización
    schedule_sync()

    {:ok, %{remote_node: remote_node, last_sync: nil}}
  end

  def handle_info(:sync, state) do
    Logger.info("Sincronizando con nodo remoto: #{state.remote_node}")

    case sync_data(state.remote_node) do
      {:ok, data} ->
        # Broadcast de datos actualizados
        PubSub.broadcast(
          MyApp.PubSub,
          "remote:sync",
          {:data_updated, data}
        )

        schedule_sync()
        {:noreply, %{state | last_sync: DateTime.utc_now()}}

      {:error, reason} ->
        Logger.error("Error en sincronización: #{inspect(reason)}")
        schedule_sync()
        {:noreply, state}
    end
  end

  defp sync_data(remote_node) do
    case :rpc.call(remote_node, MyApp.DataService, :get_latest_data, []) do
      {:badrpc, reason} ->
        {:error, reason}

      data ->
        # Guardar datos localmente
        MyApp.Cache.put(:remote_data, data)
        {:ok, data}
    end
  end

  defp schedule_sync do
    Process.send_after(self(), :sync, @sync_interval)
  end
end
```

### 5.4 LiveView que Refresca con Datos RPC

```elixir
defmodule MyAppWeb.DashboardLive do
  use MyAppWeb, :live_view
  alias Phoenix.PubSub

  def mount(_params, _session, socket) do
    if connected?(socket) do
      # Suscribirse a actualizaciones remotas
      PubSub.subscribe(MyApp.PubSub, "remote:sync")
    end

    # Obtener datos iniciales (posiblemente del cache)
    remote_data = MyApp.Cache.get(:remote_data, [])

    {:ok,
     socket
     |> assign(:remote_data, remote_data)
     |> assign(:last_update, nil)
     |> assign(:sync_status, :idle)}
  end

  # Refrescar cuando llegan datos del nodo remoto
  def handle_info({:data_updated, data}, socket) do
    {:noreply,
     socket
     |> assign(:remote_data, data)
     |> assign(:last_update, DateTime.utc_now())
     |> put_flash(:info, "Datos actualizados desde servidor remoto")}
  end

  # Permitir refrescar manualmente
  def handle_event("refresh", _params, socket) do
    send(self(), :manual_refresh)
    {:noreply, assign(socket, :sync_status, :syncing)}
  end

  def handle_info(:manual_refresh, socket) do
    remote_node = Application.get_env(:my_app, :remote_node)

    case MyApp.RemoteService.get_remote_users(remote_node) do
      {:ok, users} ->
        {:noreply,
         socket
         |> assign(:remote_data, users)
         |> assign(:sync_status, :idle)
         |> put_flash(:info, "Datos actualizados")}

      {:error, reason} ->
        {:noreply,
         socket
         |> assign(:sync_status, :error)
         |> put_flash(:error, "Error al sincronizar: #{inspect(reason)}")}
    end
  end
end
```

### 5.5 Template para Dashboard con Datos Remotos

```heex
<Layouts.app flash={@flash}>
  <div class="max-w-7xl mx-auto">
    <div class="flex justify-between items-center mb-8">
      <h1 class="text-3xl font-bold">Dashboard - Datos Remotos</h1>
      
      <button
        phx-click="refresh"
        class="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700"
        disabled={@sync_status == :syncing}
      >
        <%= if @sync_status == :syncing do %>
          Sincronizando...
        <% else %>
          Refrescar
        <% end %>
      </button>
    </div>

    <%= if @last_update do %>
      <p class="text-sm text-gray-600 mb-4">
        Última actualización: <%= Calendar.strftime(@last_update, "%H:%M:%S") %>
      </p>
    <% end %>

    <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
      <%= for item <- @remote_data do %>
        <div class="bg-white p-6 rounded-lg shadow">
          <h3 class="font-bold text-lg mb-2"><%= item.name %></h3>
          <p class="text-gray-600"><%= item.description %></p>
        </div>
      <% end %>
    </div>
  </div>
</Layouts.app>
```

### 5.6 Configuración de Nodos Remotos

```elixir
# config/runtime.exs
import Config

if config_env() == :prod do
  # Nodo remoto para RPC
  config :my_app,
    remote_node: System.get_env("REMOTE_NODE") || :"remote@hostname"
end

# lib/my_app/application.ex
defmodule MyApp.Application do
  use Application

  def start(_type, _args) do
    children = [
      MyApp.Repo,
      {Phoenix.PubSub, name: MyApp.PubSub},
      MyAppWeb.Endpoint,
      # Sincronización con nodo remoto
      {MyApp.RemoteSync, remote_node: remote_node()}
    ]

    opts = [strategy: :one_for_one, name: MyApp.Supervisor]
    Supervisor.start_link(children, opts)
  end

  defp remote_node do
    Application.get_env(:my_app, :remote_node, :nonode@nohost)
  end
end
```

---

## 6. Patrones de Refresco de Vistas HTML {#patrones-refresco}

### 6.1 Refresco Automático con Streams

**Mejor práctica**: Usar streams para colecciones grandes.

```elixir
defmodule MyAppWeb.TimelineLive do
  use MyAppWeb, :live_view
  alias Phoenix.PubSub

  def mount(_params, _session, socket) do
    if connected?(socket) do
      PubSub.subscribe(MyApp.PubSub, "timeline:posts")
    end

    posts = MyApp.Social.list_timeline_posts()

    {:ok,
     socket
     |> assign(:posts_empty?, posts == [])
     |> stream(:posts, posts)}
  end

  # Nuevo post - insertar al inicio del stream
  def handle_info({:new_post, post}, socket) do
    {:noreply,
     socket
     |> assign(:posts_empty?, false)
     |> stream_insert(:posts, post, at: 0)}
  end

  # Post actualizado - reemplazar en stream
  def handle_info({:post_updated, post}, socket) do
    {:noreply, stream_insert(socket, :posts, post)}
  end

  # Post eliminado - remover del stream
  def handle_info({:post_deleted, post_id}, socket) do
    post = %{id: post_id}
    {:noreply, stream_delete(socket, :posts, post)}
  end
end
```

### 6.2 Template para Streams

```heex
<Layouts.app flash={@flash}>
  <div class="max-w-3xl mx-auto">
    <h1 class="text-3xl font-bold mb-8">Timeline</h1>

    <%= if @posts_empty? do %>
      <p class="text-gray-500 text-center py-12">
        No hay posts todavía
      </p>
    <% else %>
      <div id="posts" phx-update="stream" class="space-y-6">
        <%= for {id, post} <- @streams.posts do %>
          <div id={id} class="bg-white p-6 rounded-lg shadow">
            <div class="flex items-start justify-between">
              <div>
                <h3 class="font-bold text-lg"><%= post.title %></h3>
                <p class="text-sm text-gray-600 mt-1">
                  Por <%= post.user.name %> · 
                  <%= Calendar.strftime(post.inserted_at, "%d/%m/%Y %H:%M") %>
                </p>
              </div>
              
              <button 
                phx-click="delete_post" 
                phx-value-id={post.id}
                class="text-red-600 hover:text-red-800"
              >
                Eliminar
              </button>
            </div>
            
            <p class="mt-4 text-gray-800"><%= post.body %></p>
          </div>
        <% end %>
      </div>
    <% end %>
  </div>
</Layouts.app>
```

### 6.3 Refresco con Asignaciones Simples

Para datos pequeños, usar `assign/2`:

```elixir
defmodule MyAppWeb.CounterLive do
  use MyAppWeb, :live_view
  alias Phoenix.PubSub

  def mount(_params, _session, socket) do
    if connected?(socket) do
      PubSub.subscribe(MyApp.PubSub, "counter")
    end

    {:ok, assign(socket, :count, get_current_count())}
  end

  def handle_info({:count_updated, new_count}, socket) do
    {:noreply, assign(socket, :count, new_count)}
  end

  def handle_event("increment", _params, socket) do
    new_count = socket.assigns.count + 1
    
    PubSub.broadcast(MyApp.PubSub, "counter", {:count_updated, new_count})
    
    {:noreply, assign(socket, :count, new_count)}
  end

  defp get_current_count do
    MyApp.Counter.get_count()
  end
end
```

### 6.4 Refresco con JS Hooks para Scroll

```javascript
// assets/js/app.js
let Hooks = {}

Hooks.InfiniteScroll = {
  mounted() {
    this.pending = false
    
    this.observer = new IntersectionObserver(entries => {
      const target = entries[0]
      if (target.isIntersecting && !this.pending) {
        this.pending = true
        this.pushEvent("load-more", {}, () => {
          this.pending = false
        })
      }
    })
    
    this.observer.observe(this.el)
  },
  
  destroyed() {
    this.observer.disconnect()
  }
}

let liveSocket = new LiveSocket("/live", Socket, {
  params: {_csrf_token: csrfToken},
  hooks: Hooks
})
```

```elixir
# LiveView con scroll infinito
defmodule MyAppWeb.FeedLive do
  use MyAppWeb, :live_view

  @per_page 20

  def mount(_params, _session, socket) do
    {:ok,
     socket
     |> assign(:page, 1)
     |> stream(:posts, load_posts(1))}
  end

  def handle_event("load-more", _params, socket) do
    next_page = socket.assigns.page + 1
    posts = load_posts(next_page)

    {:noreply,
     socket
     |> assign(:page, next_page)
     |> stream(:posts, posts)}
  end

  defp load_posts(page) do
    offset = (page - 1) * @per_page
    MyApp.Social.list_posts(limit: @per_page, offset: offset)
  end
end
```

```heex
<div id="posts" phx-update="stream">
  <%= for {id, post} <- @streams.posts do %>
    <div id={id}><%= post.title %></div>
  <% end %>
</div>

<!-- Trigger para infinite scroll -->
<div id="infinite-scroll-marker" phx-hook="InfiniteScroll"></div>
```

### 6.5 Optimización: Actualizar Solo lo Necesario

```elixir
defmodule MyAppWeb.ProductLive do
  use MyAppWeb, :live_view

  def mount(%{"id" => id}, _session, socket) do
    if connected?(socket) do
      PubSub.subscribe(MyApp.PubSub, "product:#{id}")
    end

    product = MyApp.Catalog.get_product!(id)

    {:ok,
     socket
     |> assign(:product, product)
     |> assign(:stock, product.stock)
     |> assign(:price, product.price)}
  end

  # Solo actualizar stock (no re-renderizar todo el producto)
  def handle_info({:stock_updated, new_stock}, socket) do
    {:noreply, assign(socket, :stock, new_stock)}
  end

  # Solo actualizar precio
  def handle_info({:price_updated, new_price}, socket) do
    {:noreply, assign(socket, :price, new_price)}
  end
end
```

```heex
<Layouts.app flash={@flash}>
  <div class="max-w-4xl mx-auto">
    <h1 class="text-3xl font-bold"><%= @product.name %></h1>
    
    <!-- Solo esta parte se re-renderiza cuando cambia @price -->
    <p class="text-2xl font-bold text-green-600 mt-4">
      $<%= @price %>
    </p>
    
    <!-- Solo esta parte se re-renderiza cuando cambia @stock -->
    <p class="text-sm text-gray-600 mt-2">
      Stock disponible: <%= @stock %> unidades
    </p>
    
    <!-- Esto no se re-renderiza a menos que @product cambie -->
    <div class="mt-6">
      <p class="text-gray-800"><%= @product.description %></p>
    </div>
  </div>
</Layouts.app>
```

---

## Resumen de Mejores Prácticas

### Validaciones
✅ SIEMPRE validar en el servidor con `Ecto.Changeset`
✅ Usar `phx-change` para validaciones en tiempo real
✅ Usar `to_form/2` en LiveView, `<.form>` y `<.input>` en templates
✅ Nunca acceder changeset directamente en templates

### Testing
✅ Usar ExMachina para factories con relaciones FK
✅ Usar `Phoenix.LiveViewTest` para LiveViews
✅ Usar `element/2` y `has_element?/2` en lugar de HTML crudo
✅ Dar IDs únicos a elementos clave para testing
✅ Verificar asociaciones precargadas en tests

### PubSub Multi-Nodo
✅ Usar `Phoenix.PubSub.PG2` para distribuir mensajes
✅ Suscribirse en `mount/3` cuando `connected?(socket)`
✅ Hacer broadcast después de operaciones exitosas
✅ Configurar nombres de nodo y cookies para clustering

### RPC y Sincronización
✅ Usar `:rpc.call/4` para llamadas remotas
✅ Implementar GenServer para polling periódico
✅ Usar PubSub para notificar actualizaciones
✅ Manejar errores de conexión gracefully

### Refresco de Vistas
✅ Usar streams para colecciones grandes
✅ Usar assigns simples para datos pequeños
✅ Implementar JS Hooks para scroll infinito
✅ Actualizar solo lo necesario (granularidad)

---

## Recursos Adicionales

- [Phoenix LiveView Docs](https://hexdocs.pm/phoenix_live_view)
- [Ecto Changeset Docs](https://hexdocs.pm/ecto/Ecto.Changeset.html)
- [Phoenix PubSub Docs](https://hexdocs.pm/phoenix_pubsub)
- [Distributed Elixir](https://elixir-lang.org/getting-started/mix-otp/distributed-tasks.html)
# Guía Completa de Trabajo con Phoenix Framework

Esta guía cubre las mejores prácticas para trabajar con Phoenix, incluyendo validaciones, testing, LiveView con PubSub multi-nodo, y consumo de servicios RPC.

---

## Tabla de Contenidos

1. [Validaciones en HTML y LiveView](#validaciones)
2. [Testing con Datos Relacionados (FK)](#testing-datos-relacionados)
3. [Testing de HTML Estático vs LiveView](#testing-html)
4. [LiveView con PubSub en Múltiples Nodos](#liveview-pubsub-multinodo)
5. [Consumo de Servicios RPC/EPMD y Refresco de Vistas](#rpc-refresco-vistas)
6. [Patrones de Refresco de Vistas HTML](#patrones-refresco)

---

## 1. Validaciones en HTML y LiveView {#validaciones}

### 1.1 Validaciones del Lado del Servidor con Ecto.Changeset

**Regla fundamental**: TODAS las validaciones DEBEN ocurrir en el servidor mediante `Ecto.Changeset`.

```elixir
defmodule MyApp.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset

  schema "users" do
    field :email, :string
    field :name, :string
    field :age, :integer
    timestamps()
  end

  def changeset(user, attrs) do
    user
    |> cast(attrs, [:email, :name, :age])
    |> validate_required([:email, :name])
    |> validate_format(:email, ~r/@/)
    |> validate_length(:name, min: 2, max: 100)
    |> validate_number(:age, greater_than: 0, less_than: 150)
    |> unique_constraint(:email)
  end
end
```

### 1.2 Validaciones en Tiempo Real con LiveView

**Patrón correcto**: Usar `phx-change` para validaciones mientras el usuario escribe.

```elixir
defmodule MyAppWeb.UserLive.Form do
  use MyAppWeb, :live_view

  def mount(_params, _session, socket) do
    changeset = Accounts.change_user(%User{})
    
    {:ok,
     socket
     |> assign(:form, to_form(changeset))
     |> assign(:user, nil)}
  end

  # Validación en tiempo real mientras el usuario escribe
  def handle_event("validate", %{"user" => user_params}, socket) do
    changeset =
      %User{}
      |> Accounts.change_user(user_params)
      |> Map.put(:action, :validate)

    {:noreply, assign(socket, form: to_form(changeset))}
  end

  # Guardado final con validación completa
  def handle_event("save", %{"user" => user_params}, socket) do
    case Accounts.create_user(user_params) do
      {:ok, user} ->
        {:noreply,
         socket
         |> put_flash(:info, "Usuario creado exitosamente")
         |> push_navigate(to: ~p"/users/#{user}")}

      {:error, %Ecto.Changeset{} = changeset} ->
        {:noreply, assign(socket, form: to_form(changeset))}
    end
  end
end
```

### 1.3 Template HTML para Validaciones

**Reglas importantes**:
- SIEMPRE usar `to_form/2` en el LiveView
- SIEMPRE usar `<.form>` y `<.input>` en la template
- NUNCA acceder al changeset directamente en la template
- SIEMPRE dar un ID único al form

```heex
<Layouts.app flash={@flash}>
  <div class="mx-auto max-w-2xl">
    <h1 class="text-3xl font-bold mb-8">Nuevo Usuario</h1>

    <.form 
      for={@form} 
      id="user-form" 
      phx-change="validate" 
      phx-submit="save"
      class="space-y-6"
    >
      <.input 
        field={@form[:name]} 
        type="text" 
        label="Nombre completo"
        placeholder="Juan Pérez"
        required
      />

      <.input 
        field={@form[:email]} 
        type="email" 
        label="Email"
        placeholder="juan@ejemplo.com"
        required
      />

      <.input 
        field={@form[:age]} 
        type="number" 
        label="Edad"
        min="1"
        max="150"
      />

      <button 
        type="submit"
        class="w-full px-6 py-3 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition"
      >
        Guardar Usuario
      </button>
    </.form>
  </div>
</Layouts.app>
```

### 1.4 Validaciones Personalizadas

```elixir
defmodule MyApp.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset

  # ... schema ...

  def changeset(user, attrs) do
    user
    |> cast(attrs, [:email, :name, :birth_date])
    |> validate_required([:email, :name])
    |> validate_adult()
    |> validate_email_domain()
  end

  defp validate_adult(changeset) do
    validate_change(changeset, :birth_date, fn :birth_date, birth_date ->
      today = Date.utc_today()
      age = Date.diff(today, birth_date) / 365.25

      if age >= 18 do
        []
      else
        [birth_date: "debes ser mayor de 18 años"]
      end
    end)
  end

  defp validate_email_domain(changeset) do
    validate_change(changeset, :email, fn :email, email ->
      allowed_domains = ["empresa.com", "ejemplo.com"]
      [_name, domain] = String.split(email, "@")

      if domain in allowed_domains do
        []
      else
        [email: "el dominio debe ser #{Enum.join(allowed_domains, " o ")}"]
      end
    end)
  end
end
```

---

## 2. Testing con Datos Relacionados (FK) {#testing-datos-relacionados}

### 2.1 Configuración de ExMachina para Factories

Primero, agregar la dependencia en `mix.exs`:

```elixir
defp deps do
  [
    {:ex_machina, "~> 2.7", only: :test}
  ]
end
```

### 2.2 Crear Factory con Relaciones

```elixir
# test/support/factory.ex
defmodule MyApp.Factory do
  use ExMachina.Ecto, repo: MyApp.Repo

  def user_factory do
    %MyApp.Accounts.User{
      name: sequence(:name, &"Usuario #{&1}"),
      email: sequence(:email, &"user#{&1}@ejemplo.com"),
      age: 25
    }
  end

  def post_factory do
    %MyApp.Blog.Post{
      title: sequence(:title, &"Post #{&1}"),
      body: "Contenido del post",
      published: false,
      # Relación FK - crea automáticamente un user
      user: build(:user)
    }
  end

  def comment_factory do
    %MyApp.Blog.Comment{
      body: "Este es un comentario",
      # Relaciones FK múltiples
      user: build(:user),
      post: build(:post)
    }
  end

  def category_factory do
    %MyApp.Blog.Category{
      name: sequence(:name, &"Categoría #{&1}")
    }
  end

  # Factory con relación many_to_many
  def post_with_categories_factory do
    %MyApp.Blog.Post{
      title: "Post con categorías",
      body: "Contenido",
      user: build(:user),
      categories: build_list(3, :category)
    }
  end
end
```

### 2.3 Testing de Contextos con Datos Relacionados

```elixir
# test/my_app/blog_test.exs
defmodule MyApp.BlogTest do
  use MyApp.DataCase
  import MyApp.Factory

  alias MyApp.Blog

  describe "posts" do
    test "create_post/1 crea un post asociado a un usuario" do
      user = insert(:user)
      
      attrs = %{
        title: "Mi primer post",
        body: "Contenido del post",
        user_id: user.id
      }

      assert {:ok, post} = Blog.create_post(attrs)
      assert post.user_id == user.id
      assert post.title == "Mi primer post"
    end

    test "create_post/1 falla sin user_id válido" do
      attrs = %{
        title: "Post sin usuario",
        body: "Contenido"
      }

      assert {:error, changeset} = Blog.create_post(attrs)
      assert "can't be blank" in errors_on(changeset).user_id
    end

    test "list_posts_with_user/0 precarga el usuario" do
      user = insert(:user, name: "Juan")
      insert(:post, user: user, title: "Post de Juan")
      insert(:post, user: user, title: "Otro post de Juan")

      posts = Blog.list_posts_with_user()

      assert length(posts) == 2
      assert Enum.all?(posts, fn post -> 
        Ecto.assoc_loaded?(post.user) 
      end)
      assert Enum.all?(posts, fn post -> 
        post.user.name == "Juan" 
      end)
    end
  end

  describe "comments con relaciones encadenadas" do
    test "crear comentario requiere user y post válidos" do
      user = insert(:user)
      post = insert(:post)

      attrs = %{
        body: "Gran post!",
        user_id: user.id,
        post_id: post.id
      }

      assert {:ok, comment} = Blog.create_comment(attrs)
      assert comment.user_id == user.id
      assert comment.post_id == post.id
    end

    test "list_comments_with_associations/0 precarga todo" do
      # Crear datos relacionados
      user1 = insert(:user, name: "María")
      user2 = insert(:user, name: "Pedro")
      post = insert(:post, user: user1, title: "Post de María")
      
      insert(:comment, user: user2, post: post, body: "Comentario de Pedro")
      insert(:comment, user: user1, post: post, body: "Respuesta de María")

      comments = Blog.list_comments_with_associations()

      assert length(comments) == 2
      
      # Verificar que las asociaciones están precargadas
      Enum.each(comments, fn comment ->
        assert Ecto.assoc_loaded?(comment.user)
        assert Ecto.assoc_loaded?(comment.post)
        assert Ecto.assoc_loaded?(comment.post.user)
      end)
    end

    test "eliminar post elimina comentarios en cascada" do
      post = insert(:post)
      insert_list(3, :comment, post: post)

      assert Repo.aggregate(Blog.Comment, :count) == 3

      Blog.delete_post(post)

      assert Repo.aggregate(Blog.Comment, :count) == 0
    end
  end

  describe "relaciones many_to_many" do
    test "asociar categorías a un post" do
      post = insert(:post)
      categories = insert_list(3, :category)
      category_ids = Enum.map(categories, & &1.id)

      {:ok, updated_post} = Blog.update_post_categories(post, category_ids)

      post_with_categories = Blog.get_post_with_categories!(updated_post.id)
      assert length(post_with_categories.categories) == 3
    end
  end
end
```

### 2.4 Helpers para Testing

```elixir
# test/support/data_case.ex
defmodule MyApp.DataCase do
  use ExUnit.CaseTemplate

  using do
    quote do
      alias MyApp.Repo
      import Ecto
      import Ecto.Changeset
      import Ecto.Query
      import MyApp.DataCase
      import MyApp.Factory
    end
  end

  setup tags do
    MyApp.DataCase.setup_sandbox(tags)
    :ok
  end

  def setup_sandbox(tags) do
    pid = Ecto.Adapters.SQL.Sandbox.start_owner!(MyApp.Repo, shared: not tags[:async])
    on_exit(fn -> Ecto.Adapters.SQL.Sandbox.stop_owner(pid) end)
  end

  def errors_on(changeset) do
    Ecto.Changeset.traverse_errors(changeset, fn {message, opts} ->
      Regex.replace(~r"%{(\w+)}", message, fn _, key ->
        opts |> Keyword.get(String.to_existing_atom(key), key) |> to_string()
      end)
    end)
  end
end
```

---

## 3. Testing de HTML Estático vs LiveView {#testing-html}

### 3.1 Testing de Controladores Estáticos

```elixir
# test/my_app_web/controllers/page_controller_test.exs
defmodule MyAppWeb.PageControllerTest do
  use MyAppWeb.ConnCase
  import MyApp.Factory

  describe "GET /" do
    test "muestra la página de inicio", %{conn: conn} do
      conn = get(conn, ~p"/")
      
      assert html_response(conn, 200)
      assert html = html_response(conn, 200)
      assert html =~ "Bienvenido"
    end
  end

  describe "GET /about" do
    test "muestra la página sobre nosotros", %{conn: conn} do
      conn = get(conn, ~p"/about")
      
      assert html_response(conn, 200) =~ "Sobre Nosotros"
    end
  end
end
```

### 3.2 Testing de LiveView

**Reglas importantes**:
- Usar `Phoenix.LiveViewTest`
- Usar `element/2` y `has_element?/2` en lugar de probar HTML crudo
- Usar IDs únicos en elementos clave
- Usar `Floki` para selectores complejos

```elixir
# test/my_app_web/live/user_live_test.exs
defmodule MyAppWeb.UserLiveTest do
  use MyAppWeb.ConnCase
  import Phoenix.LiveViewTest
  import MyApp.Factory

  describe "Index" do
    test "muestra todos los usuarios", %{conn: conn} do
      users = insert_list(3, :user)

      {:ok, view, html} = live(conn, ~p"/users")

      assert html =~ "Listado de Usuarios"
      
      # Verificar que cada usuario aparece
      Enum.each(users, fn user ->
        assert has_element?(view, "#user-#{user.id}")
        assert html =~ user.name
      end)
    end

    test "permite crear un nuevo usuario", %{conn: conn} do
      {:ok, view, _html} = live(conn, ~p"/users")

      # Click en botón nuevo
      assert view |> element("a", "Nuevo Usuario") |> render_click()

      # Llenar formulario
      assert view
             |> form("#user-form", user: %{name: "", email: ""})
             |> render_change() =~ "can&#39;t be blank"

      # Submit válido
      assert view
             |> form("#user-form", user: %{
               name: "Juan Pérez",
               email: "juan@ejemplo.com",
               age: 30
             })
             |> render_submit()

      # Verificar flash y redirección
      assert_redirected(view, ~p"/users/#{user_id}")
      
      # Verificar que se creó en BD
      user = Repo.get_by!(User, email: "juan@ejemplo.com")
      assert user.name == "Juan Pérez"
    end

    test "permite eliminar un usuario", %{conn: conn} do
      user = insert(:user)

      {:ok, view, _html} = live(conn, ~p"/users")

      # Click en eliminar
      assert view
             |> element("#user-#{user.id} button[phx-click='delete']")
             |> render_click()

      # Verificar que no existe más
      refute has_element?(view, "#user-#{user.id}")
      assert Repo.get(User, user.id) == nil
    end
  end

  describe "Show" do
    test "muestra un usuario con sus posts", %{conn: conn} do
      user = insert(:user, name: "María")
      posts = insert_list(3, :post, user: user)

      {:ok, view, html} = live(conn, ~p"/users/#{user}")

      assert html =~ "María"
      
      Enum.each(posts, fn post ->
        assert has_element?(view, "#post-#{post.id}")
        assert html =~ post.title
      end)
    end
  end

  describe "Form validaciones en tiempo real" do
    test "valida email en tiempo real", %{conn: conn} do
      {:ok, view, _html} = live(conn, ~p"/users/new")

      # Email inválido
      assert view
             |> form("#user-form", user: %{email: "invalido"})
             |> render_change() =~ "tiene un formato inválido"

      # Email válido
      refute view
             |> form("#user-form", user: %{email: "valido@ejemplo.com"})
             |> render_change() =~ "tiene un formato inválido"
    end
  end
end
```

### 3.3 Testing con Floki para Selectores Complejos

```elixir
test "verifica estructura compleja del DOM", %{conn: conn} do
  user = insert(:user)
  {:ok, _view, html} = live(conn, ~p"/users/#{user}")

  {:ok, document} = Floki.parse_document(html)

  # Buscar elementos específicos
  assert [_element] = Floki.find(document, "#user-profile .user-name")
  assert [_element] = Floki.find(document, "button[phx-click='edit']")
  
  # Verificar atributos
  [button] = Floki.find(document, "#edit-button")
  assert Floki.attribute(button, "phx-click") == ["edit"]
  
  # Verificar texto
  assert Floki.find(document, ".user-email") 
         |> Floki.text() 
         |> String.contains?(user.email)
end
```

### 3.4 Testing de Streams en LiveView

```elixir
test "actualiza stream cuando se agrega item", %{conn: conn} do
  {:ok, view, _html} = live(conn, ~p"/messages")

  # Estado inicial vacío
  refute has_element?(view, "#messages [id^='messages-']")

  # Enviar mensaje
  view
  |> form("#message-form", message: %{content: "Hola mundo"})
  |> render_submit()

  # Verificar que aparece en el stream
  assert has_element?(view, "#messages [id^='messages-']")
  assert render(view) =~ "Hola mundo"
end
```

---

## 4. LiveView con PubSub en Múltiples Nodos {#liveview-pubsub-multinodo}

### 4.1 Configuración de PubSub para Multi-Nodo

```elixir
# config/config.exs
config :my_app, MyApp.PubSub,
  name: MyApp.PubSub,
  adapter: Phoenix.PubSub.PG2

# lib/my_app/application.ex
defmodule MyApp.Application do
  use Application

  def start(_type, _args) do
    children = [
      # PubSub para comunicación multi-nodo
      {Phoenix.PubSub, name: MyApp.PubSub},
      MyApp.Repo,
      MyAppWeb.Endpoint
    ]

    opts = [strategy: :one_for_one, name: MyApp.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

### 4.2 Configuración de Nodos (Clustering)

```elixir
# config/runtime.exs
import Config

if config_env() == :prod do
  # Configurar nombre del nodo
  node_name = System.get_env("NODE_NAME") || "myapp"
  node_host = System.get_env("NODE_HOST") || "localhost"
  
  # Formato: nombre@host
  :net_kernel.start([:"#{node_name}@#{node_host}"])
  
  # Cookie para autenticación entre nodos
  Node.set_cookie(:my_secret_cookie)
end
```

### 4.3 LiveView con PubSub - Patrón Completo

```elixir
defmodule MyAppWeb.ChatLive do
  use MyAppWeb, :live_view
  alias MyApp.Chat
  alias Phoenix.PubSub

  @topic "chat:messages"

  def mount(_params, _session, socket) do
    if connected?(socket) do
      # Suscribirse al topic cuando el socket está conectado
      PubSub.subscribe(MyApp.PubSub, @topic)
    end

    messages = Chat.list_recent_messages()

    {:ok,
     socket
     |> assign(:messages_empty?, messages == [])
     |> stream(:messages, messages)}
  end

  def handle_event("send_message", %{"message" => message_params}, socket) do
    case Chat.create_message(message_params) do
      {:ok, message} ->
        # Broadcast a TODOS los nodos suscritos
        PubSub.broadcast(
          MyApp.PubSub,
          @topic,
          {:new_message, message}
        )

        {:noreply,
         socket
         |> put_flash(:info, "Mensaje enviado")
         |> assign(:message_form, to_form(Chat.change_message(%Message{})))}

      {:error, changeset} ->
        {:noreply, assign(socket, message_form: to_form(changeset))}
    end
  end

  # Recibir broadcasts de CUALQUIER nodo
  def handle_info({:new_message, message}, socket) do
    {:noreply,
     socket
     |> assign(:messages_empty?, false)
     |> stream_insert(:messages, message, at: 0)}
  end

  # Manejar mensaje de usuario eliminado
  def handle_info({:message_deleted, message_id}, socket) do
    message = %{id: message_id}
    {:noreply, stream_delete(socket, :messages, message)}
  end
end
```

### 4.4 Contexto con PubSub Broadcasting

```elixir
defmodule MyApp.Chat do
  alias MyApp.Repo
  alias MyApp.Chat.Message
  alias Phoenix.PubSub

  @topic "chat:messages"

  def create_message(attrs) do
    %Message{}
    |> Message.changeset(attrs)
    |> Repo.insert()
    |> case do
      {:ok, message} = result ->
        # Broadcast automático después de crear
        broadcast({:new_message, message})
        result

      error ->
        error
    end
  end

  def delete_message(%Message{} = message) do
    Repo.delete(message)
    |> case do
      {:ok, deleted_message} = result ->
        broadcast({:message_deleted, deleted_message.id})
        result

      error ->
        error
    end
  end

  def update_message(%Message{} = message, attrs) do
    message
    |> Message.changeset(attrs)
    |> Repo.update()
    |> case do
      {:ok, updated_message} = result ->
        broadcast({:message_updated, updated_message})
        result

      error ->
        error
    end
  end

  # Función privada para broadcast
  defp broadcast(message) do
    PubSub.broadcast(MyApp.PubSub, @topic, message)
  end

  def subscribe do
    PubSub.subscribe(MyApp.PubSub, @topic)
  end
end
```

### 4.5 Testing de PubSub

```elixir
defmodule MyApp.ChatTest do
  use MyApp.DataCase
  import MyApp.Factory

  alias MyApp.Chat
  alias Phoenix.PubSub

  describe "PubSub broadcasting" do
    test "crear mensaje hace broadcast" do
      # Suscribirse al topic
      :ok = Chat.subscribe()

      user = insert(:user)
      attrs = %{content: "Hola!", user_id: user.id}

      {:ok, message} = Chat.create_message(attrs)

      # Verificar que recibimos el broadcast
      assert_receive {:new_message, ^message}
    end

    test "eliminar mensaje hace broadcast del ID" do
      :ok = Chat.subscribe()

      message = insert(:message)
      {:ok, _} = Chat.delete_message(message)

      assert_receive {:message_deleted, message_id}
      assert message_id == message.id
    end
  end
end
```

### 4.6 Conectar Nodos Manualmente (para desarrollo/testing)

```elixir
# En iex de nodo 1
iex --name node1@localhost -S mix phx.server

# En iex de nodo 2  
iex --name node2@localhost -S mix phx.server

# En node2, conectar a node1
Node.connect(:"node1@localhost")

# Verificar conexión
Node.list()
# => [:"node1@localhost"]

# Verificar PubSub funciona entre nodos
Phoenix.PubSub.subscribe(MyApp.PubSub, "test")
Phoenix.PubSub.broadcast(MyApp.PubSub, "test", {:mensaje, "hola desde node2"})
# En node1 deberías recibir el mensaje
```

---

## 5. Consumo de Servicios RPC/EPMD y Refresco de Vistas {#rpc-refresco-vistas}

### 5.1 Conceptos de RPC en Elixir

EPMD (Erlang Port Mapper Daemon) permite que nodos Erlang/Elixir se descubran y comuniquen.

### 5.2 Llamadas RPC Básicas

```elixir
defmodule MyApp.RemoteService do
  @doc """
  Llama a una función en un nodo remoto
  """
  def call_remote(node, module, function, args) do
    case :rpc.call(node, module, function, args) do
      {:badrpc, reason} ->
        {:error, reason}

      result ->
        {:ok, result}
    end
  end

  @doc """
  Obtiene datos de usuarios desde nodo remoto
  """
  def get_remote_users(remote_node) do
    call_remote(
      remote_node,
      MyApp.Accounts,
      :list_users,
      []
    )
  end
end
```

### 5.3 GenServer para Sincronización con Nodo Remoto

```elixir
defmodule MyApp.RemoteSync do
  use GenServer
  alias Phoenix.PubSub
  require Logger

  @sync_interval :timer.seconds(30)

  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def init(opts) do
    remote_node = Keyword.get(opts, :remote_node)
    
    # Programar primera sincronización
    schedule_sync()

    {:ok, %{remote_node: remote_node, last_sync: nil}}
  end

  def handle_info(:sync, state) do
    Logger.info("Sincronizando con nodo remoto: #{state.remote_node}")

    case sync_data(state.remote_node) do
      {:ok, data} ->
        # Broadcast de datos actualizados
        PubSub.broadcast(
          MyApp.PubSub,
          "remote:sync",
          {:data_updated, data}
        )

        schedule_sync()
        {:noreply, %{state | last_sync: DateTime.utc_now()}}

      {:error, reason} ->
        Logger.error("Error en sincronización: #{inspect(reason)}")
        schedule_sync()
        {:noreply, state}
    end
  end

  defp sync_data(remote_node) do
    case :rpc.call(remote_node, MyApp.DataService, :get_latest_data, []) do
      {:badrpc, reason} ->
        {:error, reason}

      data ->
        # Guardar datos localmente
        MyApp.Cache.put(:remote_data, data)
        {:ok, data}
    end
  end

  defp schedule_sync do
    Process.send_after(self(), :sync, @sync_interval)
  end
end
```

### 5.4 LiveView que Refresca con Datos RPC

```elixir
defmodule MyAppWeb.DashboardLive do
  use MyAppWeb, :live_view
  alias Phoenix.PubSub

  def mount(_params, _session, socket) do
    if connected?(socket) do
      # Suscribirse a actualizaciones remotas
      PubSub.subscribe(MyApp.PubSub, "remote:sync")
    end

    # Obtener datos iniciales (posiblemente del cache)
    remote_data = MyApp.Cache.get(:remote_data, [])

    {:ok,
     socket
     |> assign(:remote_data, remote_data)
     |> assign(:last_update, nil)
     |> assign(:sync_status, :idle)}
  end

  # Refrescar cuando llegan datos del nodo remoto
  def handle_info({:data_updated, data}, socket) do
    {:noreply,
     socket
     |> assign(:remote_data, data)
     |> assign(:last_update, DateTime.utc_now())
     |> put_flash(:info, "Datos actualizados desde servidor remoto")}
  end

  # Permitir refrescar manualmente
  def handle_event("refresh", _params, socket) do
    send(self(), :manual_refresh)
    {:noreply, assign(socket, :sync_status, :syncing)}
  end

  def handle_info(:manual_refresh, socket) do
    remote_node = Application.get_env(:my_app, :remote_node)

    case MyApp.RemoteService.get_remote_users(remote_node) do
      {:ok, users} ->
        {:noreply,
         socket
         |> assign(:remote_data, users)
         |> assign(:sync_status, :idle)
         |> put_flash(:info, "Datos actualizados")}

      {:error, reason} ->
        {:noreply,
         socket
         |> assign(:sync_status, :error)
         |> put_flash(:error, "Error al sincronizar: #{inspect(reason)}")}
    end
  end
end
```

### 5.5 Template para Dashboard con Datos Remotos

```heex
<Layouts.app flash={@flash}>
  <div class="max-w-7xl mx-auto">
    <div class="flex justify-between items-center mb-8">
      <h1 class="text-3xl font-bold">Dashboard - Datos Remotos</h1>
      
      <button
        phx-click="refresh"
        class="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700"
        disabled={@sync_status == :syncing}
      >
        <%= if @sync_status == :syncing do %>
          Sincronizando...
        <% else %>
          Refrescar
        <% end %>
      </button>
    </div>

    <%= if @last_update do %>
      <p class="text-sm text-gray-600 mb-4">
        Última actualización: <%= Calendar.strftime(@last_update, "%H:%M:%S") %>
      </p>
    <% end %>

    <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
      <%= for item <- @remote_data do %>
        <div class="bg-white p-6 rounded-lg shadow">
          <h3 class="font-bold text-lg mb-2"><%= item.name %></h3>
          <p class="text-gray-600"><%= item.description %></p>
        </div>
      <% end %>
    </div>
  </div>
</Layouts.app>
```

### 5.6 Configuración de Nodos Remotos

```elixir
# config/runtime.exs
import Config

if config_env() == :prod do
  # Nodo remoto para RPC
  config :my_app,
    remote_node: System.get_env("REMOTE_NODE") || :"remote@hostname"
end

# lib/my_app/application.ex
defmodule MyApp.Application do
  use Application

  def start(_type, _args) do
    children = [
      MyApp.Repo,
      {Phoenix.PubSub, name: MyApp.PubSub},
      MyAppWeb.Endpoint,
      # Sincronización con nodo remoto
      {MyApp.RemoteSync, remote_node: remote_node()}
    ]

    opts = [strategy: :one_for_one, name: MyApp.Supervisor]
    Supervisor.start_link(children, opts)
  end

  defp remote_node do
    Application.get_env(:my_app, :remote_node, :nonode@nohost)
  end
end
```

---

## 6. Patrones de Refresco de Vistas HTML {#patrones-refresco}

### 6.1 Refresco Automático con Streams

**Mejor práctica**: Usar streams para colecciones grandes.

```elixir
defmodule MyAppWeb.TimelineLive do
  use MyAppWeb, :live_view
  alias Phoenix.PubSub

  def mount(_params, _session, socket) do
    if connected?(socket) do
      PubSub.subscribe(MyApp.PubSub, "timeline:posts")
    end

    posts = MyApp.Social.list_timeline_posts()

    {:ok,
     socket
     |> assign(:posts_empty?, posts == [])
     |> stream(:posts, posts)}
  end

  # Nuevo post - insertar al inicio del stream
  def handle_info({:new_post, post}, socket) do
    {:noreply,
     socket
     |> assign(:posts_empty?, false)
     |> stream_insert(:posts, post, at: 0)}
  end

  # Post actualizado - reemplazar en stream
  def handle_info({:post_updated, post}, socket) do
    {:noreply, stream_insert(socket, :posts, post)}
  end

  # Post eliminado - remover del stream
  def handle_info({:post_deleted, post_id}, socket) do
    post = %{id: post_id}
    {:noreply, stream_delete(socket, :posts, post)}
  end
end
```

### 6.2 Template para Streams

```heex
<Layouts.app flash={@flash}>
  <div class="max-w-3xl mx-auto">
    <h1 class="text-3xl font-bold mb-8">Timeline</h1>

    <%= if @posts_empty? do %>
      <p class="text-gray-500 text-center py-12">
        No hay posts todavía
      </p>
    <% else %>
      <div id="posts" phx-update="stream" class="space-y-6">
        <%= for {id, post} <- @streams.posts do %>
          <div id={id} class="bg-white p-6 rounded-lg shadow">
            <div class="flex items-start justify-between">
              <div>
                <h3 class="font-bold text-lg"><%= post.title %></h3>
                <p class="text-sm text-gray-600 mt-1">
                  Por <%= post.user.name %> · 
                  <%= Calendar.strftime(post.inserted_at, "%d/%m/%Y %H:%M") %>
                </p>
              </div>
              
              <button 
                phx-click="delete_post" 
                phx-value-id={post.id}
                class="text-red-600 hover:text-red-800"
              >
                Eliminar
              </button>
            </div>
            
            <p class="mt-4 text-gray-800"><%= post.body %></p>
          </div>
        <% end %>
      </div>
    <% end %>
  </div>
</Layouts.app>
```

### 6.3 Refresco con Asignaciones Simples

Para datos pequeños, usar `assign/2`:

```elixir
defmodule MyAppWeb.CounterLive do
  use MyAppWeb, :live_view
  alias Phoenix.PubSub

  def mount(_params, _session, socket) do
    if connected?(socket) do
      PubSub.subscribe(MyApp.PubSub, "counter")
    end

    {:ok, assign(socket, :count, get_current_count())}
  end

  def handle_info({:count_updated, new_count}, socket) do
    {:noreply, assign(socket, :count, new_count)}
  end

  def handle_event("increment", _params, socket) do
    new_count = socket.assigns.count + 1
    
    PubSub.broadcast(MyApp.PubSub, "counter", {:count_updated, new_count})
    
    {:noreply, assign(socket, :count, new_count)}
  end

  defp get_current_count do
    MyApp.Counter.get_count()
  end
end
```

### 6.4 Refresco con JS Hooks para Scroll

```javascript
// assets/js/app.js
let Hooks = {}

Hooks.InfiniteScroll = {
  mounted() {
    this.pending = false
    
    this.observer = new IntersectionObserver(entries => {
      const target = entries[0]
      if (target.isIntersecting && !this.pending) {
        this.pending = true
        this.pushEvent("load-more", {}, () => {
          this.pending = false
        })
      }
    })
    
    this.observer.observe(this.el)
  },
  
  destroyed() {
    this.observer.disconnect()
  }
}

let liveSocket = new LiveSocket("/live", Socket, {
  params: {_csrf_token: csrfToken},
  hooks: Hooks
})
```

```elixir
# LiveView con scroll infinito
defmodule MyAppWeb.FeedLive do
  use MyAppWeb, :live_view

  @per_page 20

  def mount(_params, _session, socket) do
    {:ok,
     socket
     |> assign(:page, 1)
     |> stream(:posts, load_posts(1))}
  end

  def handle_event("load-more", _params, socket) do
    next_page = socket.assigns.page + 1
    posts = load_posts(next_page)

    {:noreply,
     socket
     |> assign(:page, next_page)
     |> stream(:posts, posts)}
  end

  defp load_posts(page) do
    offset = (page - 1) * @per_page
    MyApp.Social.list_posts(limit: @per_page, offset: offset)
  end
end
```

```heex
<div id="posts" phx-update="stream">
  <%= for {id, post} <- @streams.posts do %>
    <div id={id}><%= post.title %></div>
  <% end %>
</div>

<!-- Trigger para infinite scroll -->
<div id="infinite-scroll-marker" phx-hook="InfiniteScroll"></div>
```

### 6.5 Optimización: Actualizar Solo lo Necesario

```elixir
defmodule MyAppWeb.ProductLive do
  use MyAppWeb, :live_view

  def mount(%{"id" => id}, _session, socket) do
    if connected?(socket) do
      PubSub.subscribe(MyApp.PubSub, "product:#{id}")
    end

    product = MyApp.Catalog.get_product!(id)

    {:ok,
     socket
     |> assign(:product, product)
     |> assign(:stock, product.stock)
     |> assign(:price, product.price)}
  end

  # Solo actualizar stock (no re-renderizar todo el producto)
  def handle_info({:stock_updated, new_stock}, socket) do
    {:noreply, assign(socket, :stock, new_stock)}
  end

  # Solo actualizar precio
  def handle_info({:price_updated, new_price}, socket) do
    {:noreply, assign(socket, :price, new_price)}
  end
end
```

```heex
<Layouts.app flash={@flash}>
  <div class="max-w-4xl mx-auto">
    <h1 class="text-3xl font-bold"><%= @product.name %></h1>
    
    <!-- Solo esta parte se re-renderiza cuando cambia @price -->
    <p class="text-2xl font-bold text-green-600 mt-4">
      $<%= @price %>
    </p>
    
    <!-- Solo esta parte se re-renderiza cuando cambia @stock -->
    <p class="text-sm text-gray-600 mt-2">
      Stock disponible: <%= @stock %> unidades
    </p>
    
    <!-- Esto no se re-renderiza a menos que @product cambie -->
    <div class="mt-6">
      <p class="text-gray-800"><%= @product.description %></p>
    </div>
  </div>
</Layouts.app>
```

---

## Resumen de Mejores Prácticas

### Validaciones
✅ SIEMPRE validar en el servidor con `Ecto.Changeset`
✅ Usar `phx-change` para validaciones en tiempo real
✅ Usar `to_form/2` en LiveView, `<.form>` y `<.input>` en templates
✅ Nunca acceder changeset directamente en templates

### Testing
✅ Usar ExMachina para factories con relaciones FK
✅ Usar `Phoenix.LiveViewTest` para LiveViews
✅ Usar `element/2` y `has_element?/2` en lugar de HTML crudo
✅ Dar IDs únicos a elementos clave para testing
✅ Verificar asociaciones precargadas en tests

### PubSub Multi-Nodo
✅ Usar `Phoenix.PubSub.PG2` para distribuir mensajes
✅ Suscribirse en `mount/3` cuando `connected?(socket)`
✅ Hacer broadcast después de operaciones exitosas
✅ Configurar nombres de nodo y cookies para clustering

### RPC y Sincronización
✅ Usar `:rpc.call/4` para llamadas remotas
✅ Implementar GenServer para polling periódico
✅ Usar PubSub para notificar actualizaciones
✅ Manejar errores de conexión gracefully

### Refresco de Vistas
✅ Usar streams para colecciones grandes
✅ Usar assigns simples para datos pequeños
✅ Implementar JS Hooks para scroll infinito
✅ Actualizar solo lo necesario (granularidad)

---

## Recursos Adicionales

- [Phoenix LiveView Docs](https://hexdocs.pm/phoenix_live_view)
- [Ecto Changeset Docs](https://hexdocs.pm/ecto/Ecto.Changeset.html)
- [Phoenix PubSub Docs](https://hexdocs.pm/phoenix_pubsub)
- [Distributed Elixir](https://elixir-lang.org/getting-started/mix-otp/distributed-tasks.html)
