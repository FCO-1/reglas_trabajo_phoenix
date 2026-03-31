# Testing con Datos Relacionados (FK)

Guía completa sobre testing en Phoenix con ExMachina y datos relacionados mediante Foreign Keys.

---

## Tabla de Contenidos

1. [Configuración de ExMachina](#configuracion)
2. [Factories con Relaciones](#factories)
3. [Testing de Contextos](#testing-contextos)
4. [Helpers para Testing](#helpers)
5. [Mejores Prácticas](#mejores-practicas)

---

## 1. Configuración de ExMachina {#configuracion}

### Instalación

Agregar la dependencia en `mix.exs`:

```elixir
defp deps do
  [
    {:ex_machina, "~> 2.7", only: :test}
  ]
end
```

Luego ejecutar:

```bash
mix deps.get
```

### Configuración en test/support/factory.ex

```elixir
defmodule MyApp.Factory do
  use ExMachina.Ecto, repo: MyApp.Repo

  # Factories básicas aquí
end
```

### Agregar al test/support/data_case.ex

```elixir
defmodule MyApp.DataCase do
  use ExUnit.CaseTemplate

  using do
    quote do
      alias MyApp.Repo
      import Ecto
      import Ecto.Changeset
      import Ecto.Query
      import MyApp.DataCase
      import MyApp.Factory  # Importar factory
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

## 2. Factories con Relaciones {#factories}

### Factory Simple

```elixir
def user_factory do
  %MyApp.Accounts.User{
    name: sequence(:name, &"Usuario #{&1}"),
    email: sequence(:email, &"user#{&1}@ejemplo.com"),
    age: 25
  }
end
```

### Factory con Relación belongs_to

```elixir
def post_factory do
  %MyApp.Blog.Post{
    title: sequence(:title, &"Post #{&1}"),
    body: "Contenido del post",
    published: false,
    # Relación FK - crea automáticamente un user
    user: build(:user)
  }
end
```

### Factory con Múltiples Relaciones FK

```elixir
def comment_factory do
  %MyApp.Blog.Comment{
    body: "Este es un comentario",
    # Relaciones FK múltiples
    user: build(:user),
    post: build(:post)
  }
end
```

### Factory con has_many

```elixir
def user_with_posts_factory do
  %MyApp.Accounts.User{
    name: "Usuario con posts",
    email: sequence(:email, &"user#{&1}@ejemplo.com"),
    # Esto NO crea los posts automáticamente
    # Los posts se deben crear manualmente
  }
end

# Uso:
# user = insert(:user_with_posts)
# insert_list(3, :post, user: user)
```

### Factory con many_to_many

```elixir
def category_factory do
  %MyApp.Blog.Category{
    name: sequence(:name, &"Categoría #{&1}")
  }
end

def post_with_categories_factory do
  %MyApp.Blog.Post{
    title: "Post con categorías",
    body: "Contenido",
    user: build(:user),
    categories: build_list(3, :category)
  }
end
```

### Traits (Variaciones de Factory)

```elixir
def user_factory do
  %MyApp.Accounts.User{
    name: sequence(:name, &"Usuario #{&1}"),
    email: sequence(:email, &"user#{&1}@ejemplo.com"),
    role: "user",
    active: true
  }
end

# Trait para admin
def admin_factory do
  struct!(
    user_factory(),
    %{role: "admin"}
  )
end

# Trait para usuario inactivo
def inactive_user_factory do
  struct!(
    user_factory(),
    %{active: false}
  )
end

# Uso:
# admin = insert(:admin)
# inactive = insert(:inactive_user)
```

---

## 3. Testing de Contextos {#testing-contextos}

### Test Básico con Relación FK

```elixir
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
  end
end
```

### Test con Preload de Asociaciones

```elixir
test "list_posts_with_user/0 precarga el usuario" do
  user = insert(:user, name: "Juan")
  insert(:post, user: user, title: "Post de Juan")
  insert(:post, user: user, title: "Otro post de Juan")

  posts = Blog.list_posts_with_user()

  assert length(posts) == 2
  
  # Verificar que las asociaciones están precargadas
  assert Enum.all?(posts, fn post -> 
    Ecto.assoc_loaded?(post.user) 
  end)
  
  assert Enum.all?(posts, fn post -> 
    post.user.name == "Juan" 
  end)
end
```

### Test con Relaciones Encadenadas

```elixir
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
end
```

### Test de Cascada (on_delete)

```elixir
test "eliminar post elimina comentarios en cascada" do
  post = insert(:post)
  insert_list(3, :comment, post: post)

  assert Repo.aggregate(Blog.Comment, :count) == 3

  Blog.delete_post(post)

  assert Repo.aggregate(Blog.Comment, :count) == 0
end
```

### Test de Relaciones many_to_many

```elixir
describe "relaciones many_to_many" do
  test "asociar categorías a un post" do
    post = insert(:post)
    categories = insert_list(3, :category)
    category_ids = Enum.map(categories, & &1.id)

    {:ok, updated_post} = Blog.update_post_categories(post, category_ids)

    post_with_categories = Blog.get_post_with_categories!(updated_post.id)
    assert length(post_with_categories.categories) == 3
  end

  test "eliminar asociación no elimina la categoría" do
    post = insert(:post_with_categories)
    category = List.first(post.categories)

    # Remover todas las categorías del post
    {:ok, updated_post} = Blog.update_post_categories(post, [])

    # La categoría sigue existiendo
    assert Repo.get(Blog.Category, category.id) != nil
    
    # Pero el post ya no tiene categorías
    post_reloaded = Blog.get_post_with_categories!(updated_post.id)
    assert post_reloaded.categories == []
  end
end
```

### Test con insert_list

```elixir
test "crear múltiples registros relacionados" do
  user = insert(:user)
  posts = insert_list(5, :post, user: user)

  assert length(posts) == 5
  assert Enum.all?(posts, fn post -> post.user_id == user.id end)
end
```

### Test con params_for (sin insertar en BD)

```elixir
test "generar parámetros sin insertar" do
  # params_for genera un mapa con los attrs, pero NO inserta
  user_params = params_for(:user)
  
  assert user_params[:name]
  assert user_params[:email]
  
  # No existe en BD
  assert Repo.get_by(User, email: user_params[:email]) == nil
end
```

### Test con build (construir sin insertar)

```elixir
test "construir struct sin insertar en BD" do
  # build crea el struct pero NO inserta
  user = build(:user)
  
  assert user.name
  assert user.email
  assert user.id == nil  # No tiene ID porque no está en BD
  
  # No existe en BD
  assert Repo.all(User) == []
end
```

---

## 4. Helpers para Testing {#helpers}

### Helper para Verificar Asociaciones Precargadas

```elixir
defmodule MyApp.DataCase do
  # ... código existente ...

  def assert_preloaded(struct, association) do
    assert Ecto.assoc_loaded?(Map.get(struct, association)),
           "Expected #{association} to be preloaded"
  end

  def refute_preloaded(struct, association) do
    refute Ecto.assoc_loaded?(Map.get(struct, association)),
           "Expected #{association} NOT to be preloaded"
  end
end

# Uso:
test "verifica preload" do
  post = Blog.get_post_with_user!(post.id)
  assert_preloaded(post, :user)
end
```

### Helper para Contar Asociaciones

```elixir
def assert_association_count(struct, association, count) do
  assoc = Map.get(struct, association)
  assert Ecto.assoc_loaded?(assoc), "#{association} not preloaded"
  assert length(assoc) == count,
         "Expected #{count} #{association}, got #{length(assoc)}"
end

# Uso:
test "post tiene 3 comentarios" do
  post = Blog.get_post_with_comments!(post_id)
  assert_association_count(post, :comments, 3)
end
```

### Helper para Crear Datos de Prueba Complejos

```elixir
def create_blog_scenario do
  user = insert(:user, name: "Blogger")
  categories = insert_list(3, :category)
  
  posts = 
    Enum.map(1..5, fn i ->
      post = insert(:post, user: user, title: "Post #{i}")
      
      # Asociar categorías
      Blog.update_post_categories(post, Enum.map(categories, & &1.id))
      
      # Agregar comentarios
      insert_list(2, :comment, post: post)
      
      post
    end)
  
  %{user: user, categories: categories, posts: posts}
end

# Uso:
test "escenario completo de blog" do
  %{user: user, posts: posts} = create_blog_scenario()
  
  assert length(posts) == 5
  assert Enum.all?(posts, fn post -> post.user_id == user.id end)
end
```

---

## 5. Mejores Prácticas {#mejores-practicas}

### ✅ Hacer

1. **Usar `insert` para crear datos en BD**
```elixir
user = insert(:user)
```

2. **Usar `build` para structs sin persistir**
```elixir
user = build(:user)
```

3. **Usar `params_for` para maps de atributos**
```elixir
attrs = params_for(:user)
```

4. **Verificar asociaciones precargadas**
```elixir
assert Ecto.assoc_loaded?(post.user)
```

5. **Usar `insert_list` para múltiples registros**
```elixir
users = insert_list(5, :user)
```

6. **Probar errores de FK inválidas**
```elixir
assert {:error, changeset} = create_post(%{user_id: -1})
assert "does not exist" in errors_on(changeset).user_id
```

### ❌ No Hacer

1. **No insertar datos innecesarios**
```elixir
# INCORRECTO: Insertar cuando solo necesitas attrs
user = insert(:user)
attrs = %{name: user.name, email: user.email}

# CORRECTO: Usar params_for
attrs = params_for(:user)
```

2. **No olvidar limpiar datos entre tests**
```elixir
# DataCase.setup_sandbox ya lo hace automáticamente
# con Ecto.Adapters.SQL.Sandbox
```

3. **No asumir el orden de inserción**
```elixir
# INCORRECTO: Asumir IDs específicos
assert user.id == 1

# CORRECTO: Comparar structs o verificar existencia
assert user.id != nil
```

4. **No crear relaciones circulares**
```elixir
# INCORRECTO: Post crea User que crea Post...
def post_factory do
  %Post{
    user: build(:user_with_posts)  # ❌ Círculo infinito
  }
end
```

### Patrones Comunes

#### Setup con Datos Compartidos

```elixir
describe "posts" do
  setup do
    user = insert(:user)
    post = insert(:post, user: user)
    
    %{user: user, post: post}
  end
  
  test "puede eliminar su propio post", %{user: user, post: post} do
    assert {:ok, _} = Blog.delete_post(post, user)
  end
end
```

#### Testing de Queries Complejas

```elixir
test "buscar posts por múltiples criterios" do
  user = insert(:user)
  category = insert(:category, name: "Tech")
  
  tech_post = insert(:post, user: user, published: true)
  Blog.update_post_categories(tech_post, [category.id])
  
  draft_post = insert(:post, user: user, published: false)
  
  # Buscar solo posts publicados de categoría Tech
  results = Blog.search_posts(%{
    published: true,
    category_id: category.id
  })
  
  assert length(results) == 1
  assert List.first(results).id == tech_post.id
end
```

---

## Recursos Adicionales

- [ExMachina Documentation](https://hexdocs.pm/ex_machina)
- [Ecto Associations Guide](https://hexdocs.pm/ecto/associations.html)
- [ExUnit Documentation](https://hexdocs.pm/ex_unit/ExUnit.html)

# Testing con Datos Relacionados (FK)

Guía completa sobre testing en Phoenix con ExMachina y datos relacionados mediante Foreign Keys.

---

## Tabla de Contenidos

1. [Configuración de ExMachina](#configuracion)
2. [Factories con Relaciones](#factories)
3. [Testing de Contextos](#testing-contextos)
4. [Helpers para Testing](#helpers)
5. [Mejores Prácticas](#mejores-practicas)

---

## 1. Configuración de ExMachina {#configuracion}

### Instalación

Agregar la dependencia en `mix.exs`:

```elixir
defp deps do
  [
    {:ex_machina, "~> 2.7", only: :test}
  ]
end
```

Luego ejecutar:

```bash
mix deps.get
```

### Configuración en test/support/factory.ex

```elixir
defmodule MyApp.Factory do
  use ExMachina.Ecto, repo: MyApp.Repo

  # Factories básicas aquí
end
```

### Agregar al test/support/data_case.ex

```elixir
defmodule MyApp.DataCase do
  use ExUnit.CaseTemplate

  using do
    quote do
      alias MyApp.Repo
      import Ecto
      import Ecto.Changeset
      import Ecto.Query
      import MyApp.DataCase
      import MyApp.Factory  # Importar factory
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

## 2. Factories con Relaciones {#factories}

### Factory Simple

```elixir
def user_factory do
  %MyApp.Accounts.User{
    name: sequence(:name, &"Usuario #{&1}"),
    email: sequence(:email, &"user#{&1}@ejemplo.com"),
    age: 25
  }
end
```

### Factory con Relación belongs_to

```elixir
def post_factory do
  %MyApp.Blog.Post{
    title: sequence(:title, &"Post #{&1}"),
    body: "Contenido del post",
    published: false,
    # Relación FK - crea automáticamente un user
    user: build(:user)
  }
end
```

### Factory con Múltiples Relaciones FK

```elixir
def comment_factory do
  %MyApp.Blog.Comment{
    body: "Este es un comentario",
    # Relaciones FK múltiples
    user: build(:user),
    post: build(:post)
  }
end
```

### Factory con has_many

```elixir
def user_with_posts_factory do
  %MyApp.Accounts.User{
    name: "Usuario con posts",
    email: sequence(:email, &"user#{&1}@ejemplo.com"),
    # Esto NO crea los posts automáticamente
    # Los posts se deben crear manualmente
  }
end

# Uso:
# user = insert(:user_with_posts)
# insert_list(3, :post, user: user)
```

### Factory con many_to_many

```elixir
def category_factory do
  %MyApp.Blog.Category{
    name: sequence(:name, &"Categoría #{&1}")
  }
end

def post_with_categories_factory do
  %MyApp.Blog.Post{
    title: "Post con categorías",
    body: "Contenido",
    user: build(:user),
    categories: build_list(3, :category)
  }
end
```

### Traits (Variaciones de Factory)

```elixir
def user_factory do
  %MyApp.Accounts.User{
    name: sequence(:name, &"Usuario #{&1}"),
    email: sequence(:email, &"user#{&1}@ejemplo.com"),
    role: "user",
    active: true
  }
end

# Trait para admin
def admin_factory do
  struct!(
    user_factory(),
    %{role: "admin"}
  )
end

# Trait para usuario inactivo
def inactive_user_factory do
  struct!(
    user_factory(),
    %{active: false}
  )
end

# Uso:
# admin = insert(:admin)
# inactive = insert(:inactive_user)
```

---

## 3. Testing de Contextos {#testing-contextos}

### Test Básico con Relación FK

```elixir
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
  end
end
```

### Test con Preload de Asociaciones

```elixir
test "list_posts_with_user/0 precarga el usuario" do
  user = insert(:user, name: "Juan")
  insert(:post, user: user, title: "Post de Juan")
  insert(:post, user: user, title: "Otro post de Juan")

  posts = Blog.list_posts_with_user()

  assert length(posts) == 2
  
  # Verificar que las asociaciones están precargadas
  assert Enum.all?(posts, fn post -> 
    Ecto.assoc_loaded?(post.user) 
  end)
  
  assert Enum.all?(posts, fn post -> 
    post.user.name == "Juan" 
  end)
end
```

### Test con Relaciones Encadenadas

```elixir
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
end
```

### Test de Cascada (on_delete)

```elixir
test "eliminar post elimina comentarios en cascada" do
  post = insert(:post)
  insert_list(3, :comment, post: post)

  assert Repo.aggregate(Blog.Comment, :count) == 3

  Blog.delete_post(post)

  assert Repo.aggregate(Blog.Comment, :count) == 0
end
```

### Test de Relaciones many_to_many

```elixir
describe "relaciones many_to_many" do
  test "asociar categorías a un post" do
    post = insert(:post)
    categories = insert_list(3, :category)
    category_ids = Enum.map(categories, & &1.id)

    {:ok, updated_post} = Blog.update_post_categories(post, category_ids)

    post_with_categories = Blog.get_post_with_categories!(updated_post.id)
    assert length(post_with_categories.categories) == 3
  end

  test "eliminar asociación no elimina la categoría" do
    post = insert(:post_with_categories)
    category = List.first(post.categories)

    # Remover todas las categorías del post
    {:ok, updated_post} = Blog.update_post_categories(post, [])

    # La categoría sigue existiendo
    assert Repo.get(Blog.Category, category.id) != nil
    
    # Pero el post ya no tiene categorías
    post_reloaded = Blog.get_post_with_categories!(updated_post.id)
    assert post_reloaded.categories == []
  end
end
```

### Test con insert_list

```elixir
test "crear múltiples registros relacionados" do
  user = insert(:user)
  posts = insert_list(5, :post, user: user)

  assert length(posts) == 5
  assert Enum.all?(posts, fn post -> post.user_id == user.id end)
end
```

### Test con params_for (sin insertar en BD)

```elixir
test "generar parámetros sin insertar" do
  # params_for genera un mapa con los attrs, pero NO inserta
  user_params = params_for(:user)
  
  assert user_params[:name]
  assert user_params[:email]
  
  # No existe en BD
  assert Repo.get_by(User, email: user_params[:email]) == nil
end
```

### Test con build (construir sin insertar)

```elixir
test "construir struct sin insertar en BD" do
  # build crea el struct pero NO inserta
  user = build(:user)
  
  assert user.name
  assert user.email
  assert user.id == nil  # No tiene ID porque no está en BD
  
  # No existe en BD
  assert Repo.all(User) == []
end
```

---

## 4. Helpers para Testing {#helpers}

### Helper para Verificar Asociaciones Precargadas

```elixir
defmodule MyApp.DataCase do
  # ... código existente ...

  def assert_preloaded(struct, association) do
    assert Ecto.assoc_loaded?(Map.get(struct, association)),
           "Expected #{association} to be preloaded"
  end

  def refute_preloaded(struct, association) do
    refute Ecto.assoc_loaded?(Map.get(struct, association)),
           "Expected #{association} NOT to be preloaded"
  end
end

# Uso:
test "verifica preload" do
  post = Blog.get_post_with_user!(post.id)
  assert_preloaded(post, :user)
end
```

### Helper para Contar Asociaciones

```elixir
def assert_association_count(struct, association, count) do
  assoc = Map.get(struct, association)
  assert Ecto.assoc_loaded?(assoc), "#{association} not preloaded"
  assert length(assoc) == count,
         "Expected #{count} #{association}, got #{length(assoc)}"
end

# Uso:
test "post tiene 3 comentarios" do
  post = Blog.get_post_with_comments!(post_id)
  assert_association_count(post, :comments, 3)
end
```

### Helper para Crear Datos de Prueba Complejos

```elixir
def create_blog_scenario do
  user = insert(:user, name: "Blogger")
  categories = insert_list(3, :category)
  
  posts = 
    Enum.map(1..5, fn i ->
      post = insert(:post, user: user, title: "Post #{i}")
      
      # Asociar categorías
      Blog.update_post_categories(post, Enum.map(categories, & &1.id))
      
      # Agregar comentarios
      insert_list(2, :comment, post: post)
      
      post
    end)
  
  %{user: user, categories: categories, posts: posts}
end

# Uso:
test "escenario completo de blog" do
  %{user: user, posts: posts} = create_blog_scenario()
  
  assert length(posts) == 5
  assert Enum.all?(posts, fn post -> post.user_id == user.id end)
end
```

---

## 5. Mejores Prácticas {#mejores-practicas}

### ✅ Hacer

1. **Usar `insert` para crear datos en BD**
```elixir
user = insert(:user)
```

2. **Usar `build` para structs sin persistir**
```elixir
user = build(:user)
```

3. **Usar `params_for` para maps de atributos**
```elixir
attrs = params_for(:user)
```

4. **Verificar asociaciones precargadas**
```elixir
assert Ecto.assoc_loaded?(post.user)
```

5. **Usar `insert_list` para múltiples registros**
```elixir
users = insert_list(5, :user)
```

6. **Probar errores de FK inválidas**
```elixir
assert {:error, changeset} = create_post(%{user_id: -1})
assert "does not exist" in errors_on(changeset).user_id
```

### ❌ No Hacer

1. **No insertar datos innecesarios**
```elixir
# INCORRECTO: Insertar cuando solo necesitas attrs
user = insert(:user)
attrs = %{name: user.name, email: user.email}

# CORRECTO: Usar params_for
attrs = params_for(:user)
```

2. **No olvidar limpiar datos entre tests**
```elixir
# DataCase.setup_sandbox ya lo hace automáticamente
# con Ecto.Adapters.SQL.Sandbox
```

3. **No asumir el orden de inserción**
```elixir
# INCORRECTO: Asumir IDs específicos
assert user.id == 1

# CORRECTO: Comparar structs o verificar existencia
assert user.id != nil
```

4. **No crear relaciones circulares**
```elixir
# INCORRECTO: Post crea User que crea Post...
def post_factory do
  %Post{
    user: build(:user_with_posts)  # ❌ Círculo infinito
  }
end
```

### Patrones Comunes

#### Setup con Datos Compartidos

```elixir
describe "posts" do
  setup do
    user = insert(:user)
    post = insert(:post, user: user)
    
    %{user: user, post: post}
  end
  
  test "puede eliminar su propio post", %{user: user, post: post} do
    assert {:ok, _} = Blog.delete_post(post, user)
  end
end
```

#### Testing de Queries Complejas

```elixir
test "buscar posts por múltiples criterios" do
  user = insert(:user)
  category = insert(:category, name: "Tech")
  
  tech_post = insert(:post, user: user, published: true)
  Blog.update_post_categories(tech_post, [category.id])
  
  draft_post = insert(:post, user: user, published: false)
  
  # Buscar solo posts publicados de categoría Tech
  results = Blog.search_posts(%{
    published: true,
    category_id: category.id
  })
  
  assert length(results) == 1
  assert List.first(results).id == tech_post.id
end
```

---

## Recursos Adicionales

- [ExMachina Documentation](https://hexdocs.pm/ex_machina)
- [Ecto Associations Guide](https://hexdocs.pm/ecto/associations.html)
- [ExUnit Documentation](https://hexdocs.pm/ex_unit/ExUnit.html)
