# Testing de Formularios LiveView

Guía práctica para testing de formularios en Phoenix LiveView.

---

## Setup Básico

```elixir
# test/my_app_web/live/product_live_test.exs
defmodule MyAppWeb.ProductLiveTest do
  use MyAppWeb.ConnCase
  import Phoenix.LiveViewTest
  import MyApp.Factory

  describe "Formulario de Producto" do
    test "renderiza el formulario", %{conn: conn} do
      {:ok, view, html} = live(conn, ~p"/products/new")

      assert html =~ "Nuevo Producto"
      assert has_element?(view, "#product-form")
    end
  end
end
```

---

## Testing de Validaciones en Tiempo Real

```elixir
test "valida en tiempo real con phx-change", %{conn: conn} do
  {:ok, view, _html} = live(conn, ~p"/products/new")

  # Campo vacío muestra error
  assert view
         |> form("#product-form", product: %{name: ""})
         |> render_change() =~ "can&#39;t be blank"

  # Campo válido no muestra error
  refute view
         |> form("#product-form", product: %{name: "Widget"})
         |> render_change() =~ "can&#39;t be blank"
end

test "valida formato de email", %{conn: conn} do
  {:ok, view, _html} = live(conn, ~p"/users/new")

  # Email inválido
  html = view
         |> form("#user-form", user: %{email: "invalido"})
         |> render_change()

  assert html =~ "must have the @ sign"

  # Email válido
  html = view
         |> form("#user-form", user: %{email: "test@ejemplo.com"})
         |> render_change()

  refute html =~ "must have the @ sign"
end
```

---

## Testing de Submit

```elixir
test "crea producto con datos válidos", %{conn: conn} do
  {:ok, view, _html} = live(conn, ~p"/products/new")

  assert view
         |> form("#product-form", product: %{
           name: "Widget",
           price: 29.99,
           sku: "WDG-001"
         })
         |> render_submit()

  # Verificar redirección
  assert_redirected(view, ~p"/products/#{product_id}")

  # Verificar que se creó en BD
  product = Repo.get_by!(Product, sku: "WDG-001")
  assert product.name == "Widget"
  assert Decimal.equal?(product.price, Decimal.new("29.99"))
end

test "muestra errores con datos inválidos", %{conn: conn} do
  {:ok, view, _html} = live(conn, ~p"/products/new")

  html = view
         |> form("#product-form", product: %{
           name: "",
           price: -10
         })
         |> render_submit()

  assert html =~ "can&#39;t be blank"
  assert html =~ "must be greater than"
end
```

---

## Testing de Formularios de Edición

```elixir
test "edita producto existente", %{conn: conn} do
  product = insert(:product, name: "Old Name", price: 10.00)

  {:ok, view, html} = live(conn, ~p"/products/#{product}/edit")

  # Verificar que muestra datos actuales
  assert html =~ "Old Name"
  assert html =~ "10.00"

  # Actualizar
  assert view
         |> form("#product-form", product: %{
           name: "New Name",
           price: 15.00
         })
         |> render_submit()

  assert_redirected(view, ~p"/products/#{product}")

  # Verificar actualización en BD
  updated = Repo.get!(Product, product.id)
  assert updated.name == "New Name"
  assert Decimal.equal?(updated.price, Decimal.new("15.00"))
end
```

---

## Testing de Formularios con Relaciones

```elixir
test "crea post con categorías", %{conn: conn} do
  category1 = insert(:category, name: "Tech")
  category2 = insert(:category, name: "News")

  {:ok, view, _html} = live(conn, ~p"/posts/new")

  assert view
         |> form("#post-form", post: %{
           title: "Mi Post",
           body: "Contenido",
           category_ids: [category1.id, category2.id]
         })
         |> render_submit()

  # Verificar relaciones
  post = Repo.get_by!(Post, title: "Mi Post") |> Repo.preload(:categories)
  assert length(post.categories) == 2
  assert Enum.any?(post.categories, &(&1.name == "Tech"))
end
```

---

## Testing de Formularios Anidados

```elixir
test "crea orden con items", %{conn: conn} do
  product1 = insert(:product)
  product2 = insert(:product)

  {:ok, view, _html} = live(conn, ~p"/orders/new")

  assert view
         |> form("#order-form", order: %{
           items: %{
             "0" => %{product_id: product1.id, quantity: 2},
             "1" => %{product_id: product2.id, quantity: 1}
           }
         })
         |> render_submit()

  order = Repo.one!(Order) |> Repo.preload(:items)
  assert length(order.items) == 2
end
```

---

## Testing con Floki

```elixir
test "verifica estructura del formulario", %{conn: conn} do
  {:ok, _view, html} = live(conn, ~p"/products/new")

  {:ok, document} = Floki.parse_document(html)

  # Verificar campos específicos
  assert [_input] = Floki.find(document, "input[name='product[name]']")
  assert [_input] = Floki.find(document, "input[name='product[price]']")
  assert [_button] = Floki.find(document, "button[type='submit']")

  # Verificar atributos
  [name_input] = Floki.find(document, "input[name='product[name]']")
  assert Floki.attribute(name_input, "required") == ["required"]
end
```

---

## Testing de Uploads

```elixir
test "sube archivo correctamente", %{conn: conn} do
  {:ok, view, _html} = live(conn, ~p"/products/new")

  # Crear archivo temporal
  image = %Plug.Upload{
    path: "test/support/fixtures/product.jpg",
    filename: "product.jpg"
  }

  view
  |> form("#product-form", product: %{
    name: "Product",
    image: image
  })
  |> render_submit()

  product = Repo.get_by!(Product, name: "Product")
  assert product.image_url =~ "product.jpg"
end
```

---

## Testing de Errores de Validación Personalizados

```elixir
test "muestra error de validación personalizada", %{conn: conn} do
  {:ok, view, _html} = live(conn, ~p"/products/new")

  existing_product = insert(:product, sku: "ABC-123")

  html = view
         |> form("#product-form", product: %{
           name: "Nuevo",
           sku: "ABC-123"  # SKU duplicado
         })
         |> render_submit()

  assert html =~ "has already been taken"
end
```

---

## Mejores Prácticas

### ✅ Hacer

```elixir
# Usar IDs únicos en formularios
test "usa ID único", %{conn: conn} do
  {:ok, view, _html} = live(conn, ~p"/products/new")
  assert has_element?(view, "#product-form")
end

# Verificar con has_element? en lugar de HTML crudo
assert has_element?(view, "button[type='submit']")

# Verificar cambios en BD
product = Repo.get_by!(Product, sku: "ABC")
assert product.name == "Expected"
```

### ❌ No Hacer

```elixir
# No confiar solo en HTML
refute html =~ "error"  # Puede fallar con cambios de texto

# No omitir verificación de BD
# Siempre verificar que los datos se guardaron correctamente

# No usar render_change sin verificar resultado
form(view, "#form", data: %{...}) |> render_change()  # ❌
```

---

## Testing con LiveView Helpers

```elixir
test "usa helpers de LiveView", %{conn: conn} do
  {:ok, view, _html} = live(conn, ~p"/products/new")

  # element/3 para interacciones específicas
  view
  |> element("#add-item-button")
  |> render_click()

  # form/3 para formularios
  view
  |> form("#product-form", product: %{name: "Test"})
  |> render_submit()

  # has_element?/2 para verificaciones
  assert has_element?(view, ".success-message")
end
```

---

## Recursos

- [Phoenix.LiveViewTest Docs](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveViewTest.html)
- [Floki Docs](https://hexdocs.pm/floki)
