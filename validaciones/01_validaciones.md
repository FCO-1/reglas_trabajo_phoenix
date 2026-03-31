# Validaciones en Phoenix y LiveView

Guía completa sobre validaciones del lado del servidor y en tiempo real con Phoenix Framework.

---

## Tabla de Contenidos

1. [Validaciones del Lado del Servidor](#validaciones-servidor)
2. [Validaciones en Tiempo Real con LiveView](#validaciones-tiempo-real)
3. [Templates HTML para Validaciones](#templates-html)
4. [Validaciones Personalizadas](#validaciones-personalizadas)
5. [Mejores Prácticas](#mejores-practicas)

---

## 1. Validaciones del Lado del Servidor {#validaciones-servidor}

### Regla Fundamental

**TODAS las validaciones DEBEN ocurrir en el servidor mediante `Ecto.Changeset`.**

Las validaciones del cliente (HTML5, JavaScript) son solo para mejorar la UX, pero **nunca** reemplazan las validaciones del servidor.

### Ejemplo Básico

```elixir
defmodule MyApp.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset

  schema "users" do
    field :email, :string
    field :name, :string
    field :age, :integer
    field :phone, :string
    timestamps()
  end

  def changeset(user, attrs) do
    user
    |> cast(attrs, [:email, :name, :age, :phone])
    |> validate_required([:email, :name])
    |> validate_format(:email, ~r/^[^\s]+@[^\s]+$/, message: "debe ser un email válido")
    |> validate_length(:name, min: 2, max: 100)
    |> validate_number(:age, greater_than: 0, less_than: 150)
    |> validate_format(:phone, ~r/^\d{3}-\d{3}-\d{4}$/, message: "formato debe ser XXX-XXX-XXXX")
    |> unique_constraint(:email)
  end
end
```

### Validaciones Disponibles en Ecto

```elixir
# Validación de presencia
|> validate_required([:field1, :field2])

# Validación de formato (regex)
|> validate_format(:email, ~r/@/)

# Validación de longitud
|> validate_length(:name, min: 2, max: 100)
|> validate_length(:description, min: 10)
|> validate_length(:password, is: 8)

# Validación de números
|> validate_number(:age, greater_than: 0)
|> validate_number(:price, greater_than_or_equal_to: 0)
|> validate_number(:discount, less_than: 100)
|> validate_number(:quantity, equal_to: 1)

# Validación de inclusión/exclusión
|> validate_inclusion(:status, ["active", "inactive", "pending"])
|> validate_exclusion(:username, ["admin", "root", "system"])

# Validación de cambios
|> validate_change(:field, fn :field, value ->
  # Retornar [] si es válido, o [{:field, "mensaje"}] si es inválido
end)

# Validación de confirmación (útil para passwords)
|> validate_confirmation(:password, message: "las contraseñas no coinciden")

# Validación de aceptación (términos y condiciones)
|> validate_acceptance(:terms_of_service)

# Restricciones de base de datos
|> unique_constraint(:email)
|> foreign_key_constraint(:user_id)
```

---

## 2. Validaciones en Tiempo Real con LiveView {#validaciones-tiempo-real}

### Patrón Correcto: phx-change

```elixir
defmodule MyAppWeb.UserLive.Form do
  use MyAppWeb, :live_view
  alias MyApp.Accounts

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

---

## 3. Templates HTML para Validaciones {#templates-html}

### Reglas Importantes

✅ **SIEMPRE** usar `to_form/2` en el LiveView
✅ **SIEMPRE** usar `<.form>` y `<.input>` en la template
✅ **NUNCA** acceder al changeset directamente en la template
✅ **SIEMPRE** dar un ID único al form

### Template Completa

```heex
<Layouts.app flash={@flash}>
  <div class="mx-auto max-w-2xl px-4 py-8">
    <h1 class="text-3xl font-bold mb-8 text-gray-900">Nuevo Usuario</h1>

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

      <.input 
        field={@form[:phone]} 
        type="tel" 
        label="Teléfono"
        placeholder="555-123-4567"
      />

      <button 
        type="submit"
        class="w-full px-6 py-3 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition font-medium"
      >
        Guardar Usuario
      </button>
    </.form>
  </div>
</Layouts.app>
```

---

## 4. Validaciones Personalizadas {#validaciones-personalizadas}

### Validación de Edad Adulta

```elixir
defmodule MyApp.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset

  schema "users" do
    field :email, :string
    field :name, :string
    field :birth_date, :date
    timestamps()
  end

  def changeset(user, attrs) do
    user
    |> cast(attrs, [:email, :name, :birth_date])
    |> validate_required([:email, :name, :birth_date])
    |> validate_adult()
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
end
```

### Validación de Dominio de Email

```elixir
defp validate_email_domain(changeset) do
  validate_change(changeset, :email, fn :email, email ->
    allowed_domains = ["empresa.com", "ejemplo.com"]
    
    case String.split(email, "@") do
      [_name, domain] when domain in allowed_domains ->
        []
      
      [_name, domain] ->
        [email: "el dominio #{domain} no está permitido"]
      
      _ ->
        [email: "formato de email inválido"]
    end
  end)
end
```

### Validación de Contraseña Fuerte

```elixir
defp validate_strong_password(changeset) do
  validate_change(changeset, :password, fn :password, password ->
    errors = []
    
    errors = if String.length(password) < 8 do
      ["debe tener al menos 8 caracteres" | errors]
    else
      errors
    end
    
    errors = if not String.match?(password, ~r/[A-Z]/) do
      ["debe contener al menos una mayúscula" | errors]
    else
      errors
    end
    
    errors = if not String.match?(password, ~r/[a-z]/) do
      ["debe contener al menos una minúscula" | errors]
    else
      errors
    end
    
    errors = if not String.match?(password, ~r/[0-9]/) do
      ["debe contener al menos un número" | errors]
    else
      errors
    end
    
    if Enum.empty?(errors) do
      []
    else
      [password: Enum.join(errors, ", ")]
    end
  end)
end
```

### Validación Condicional

```elixir
def changeset(user, attrs) do
  user
  |> cast(attrs, [:type, :company_name, :tax_id])
  |> validate_required([:type])
  |> validate_company_fields()
end

defp validate_company_fields(changeset) do
  type = get_field(changeset, :type)
  
  if type == "business" do
    changeset
    |> validate_required([:company_name, :tax_id])
    |> validate_length(:tax_id, is: 9)
  else
    changeset
  end
end
```

---

## 5. Mejores Prácticas {#mejores-practicas}

### ✅ Hacer

1. **Validar en el servidor SIEMPRE**
```elixir
# Contexto
def create_user(attrs) do
  %User{}
  |> User.changeset(attrs)
  |> Repo.insert()
end
```

2. **Usar `to_form/2` en LiveView**
```elixir
def mount(_params, _session, socket) do
  {:ok, assign(socket, :form, to_form(changeset))}
end
```

3. **Acceder campos con `@form[:field]`**
```heex
<.input field={@form[:email]} type="email" />
```

4. **Dar IDs únicos a formularios**
```heex
<.form for={@form} id="user-form" phx-submit="save">
```

5. **Usar mensajes de error descriptivos**
```elixir
|> validate_format(:email, ~r/@/, message: "debe ser un email válido")
```

### ❌ No Hacer

1. **No confiar en validaciones del cliente**
```html
<!-- INCORRECTO: Solo HTML5 validation -->
<input type="email" required>
```

2. **No acceder al changeset en templates**
```heex
<!-- INCORRECTO -->
<%= @changeset.errors[:email] %>

<!-- CORRECTO -->
<.input field={@form[:email]} />
```

3. **No olvidar `phx-change` para validación en tiempo real**
```heex
<!-- INCORRECTO: Sin phx-change -->
<.form for={@form} phx-submit="save">

<!-- CORRECTO: Con phx-change -->
<.form for={@form} phx-change="validate" phx-submit="save">
```

4. **No usar `validate_number/3` con `:allow_nil`**
```elixir
# INCORRECTO: validate_number no soporta :allow_nil
|> validate_number(:age, greater_than: 0, allow_nil: true)

# CORRECTO: La validación solo corre si el campo tiene valor
|> validate_number(:age, greater_than: 0)
```

---

## Recursos Adicionales

- [Ecto.Changeset Documentation](https://hexdocs.pm/ecto/Ecto.Changeset.html)
- [Phoenix.Component.form/1](https://hexdocs.pm/phoenix_live_view/Phoenix.Component.html#form/1)
- [Phoenix LiveView Forms Guide](https://hexdocs.pm/phoenix_live_view/form-bindings.html)

# Validaciones en Phoenix y LiveView

Guía completa sobre validaciones del lado del servidor y en tiempo real con Phoenix Framework.

---

## Tabla de Contenidos

1. [Validaciones del Lado del Servidor](#validaciones-servidor)
2. [Validaciones en Tiempo Real con LiveView](#validaciones-tiempo-real)
3. [Templates HTML para Validaciones](#templates-html)
4. [Validaciones Personalizadas](#validaciones-personalizadas)
5. [Mejores Prácticas](#mejores-practicas)

---

## 1. Validaciones del Lado del Servidor {#validaciones-servidor}

### Regla Fundamental

**TODAS las validaciones DEBEN ocurrir en el servidor mediante `Ecto.Changeset`.**

Las validaciones del cliente (HTML5, JavaScript) son solo para mejorar la UX, pero **nunca** reemplazan las validaciones del servidor.

### Ejemplo Básico

```elixir
defmodule MyApp.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset

  schema "users" do
    field :email, :string
    field :name, :string
    field :age, :integer
    field :phone, :string
    timestamps()
  end

  def changeset(user, attrs) do
    user
    |> cast(attrs, [:email, :name, :age, :phone])
    |> validate_required([:email, :name])
    |> validate_format(:email, ~r/^[^\s]+@[^\s]+$/, message: "debe ser un email válido")
    |> validate_length(:name, min: 2, max: 100)
    |> validate_number(:age, greater_than: 0, less_than: 150)
    |> validate_format(:phone, ~r/^\d{3}-\d{3}-\d{4}$/, message: "formato debe ser XXX-XXX-XXXX")
    |> unique_constraint(:email)
  end
end
```

### Validaciones Disponibles en Ecto

```elixir
# Validación de presencia
|> validate_required([:field1, :field2])

# Validación de formato (regex)
|> validate_format(:email, ~r/@/)

# Validación de longitud
|> validate_length(:name, min: 2, max: 100)
|> validate_length(:description, min: 10)
|> validate_length(:password, is: 8)

# Validación de números
|> validate_number(:age, greater_than: 0)
|> validate_number(:price, greater_than_or_equal_to: 0)
|> validate_number(:discount, less_than: 100)
|> validate_number(:quantity, equal_to: 1)

# Validación de inclusión/exclusión
|> validate_inclusion(:status, ["active", "inactive", "pending"])
|> validate_exclusion(:username, ["admin", "root", "system"])

# Validación de cambios
|> validate_change(:field, fn :field, value ->
  # Retornar [] si es válido, o [{:field, "mensaje"}] si es inválido
end)

# Validación de confirmación (útil para passwords)
|> validate_confirmation(:password, message: "las contraseñas no coinciden")

# Validación de aceptación (términos y condiciones)
|> validate_acceptance(:terms_of_service)

# Restricciones de base de datos
|> unique_constraint(:email)
|> foreign_key_constraint(:user_id)
```

---

## 2. Validaciones en Tiempo Real con LiveView {#validaciones-tiempo-real}

### Patrón Correcto: phx-change

```elixir
defmodule MyAppWeb.UserLive.Form do
  use MyAppWeb, :live_view
  alias MyApp.Accounts

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

---

## 3. Templates HTML para Validaciones {#templates-html}

### Reglas Importantes

✅ **SIEMPRE** usar `to_form/2` en el LiveView
✅ **SIEMPRE** usar `<.form>` y `<.input>` en la template
✅ **NUNCA** acceder al changeset directamente en la template
✅ **SIEMPRE** dar un ID único al form

### Template Completa

```heex
<Layouts.app flash={@flash}>
  <div class="mx-auto max-w-2xl px-4 py-8">
    <h1 class="text-3xl font-bold mb-8 text-gray-900">Nuevo Usuario</h1>

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

      <.input 
        field={@form[:phone]} 
        type="tel" 
        label="Teléfono"
        placeholder="555-123-4567"
      />

      <button 
        type="submit"
        class="w-full px-6 py-3 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition font-medium"
      >
        Guardar Usuario
      </button>
    </.form>
  </div>
</Layouts.app>
```

---

## 4. Validaciones Personalizadas {#validaciones-personalizadas}

### Validación de Edad Adulta

```elixir
defmodule MyApp.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset

  schema "users" do
    field :email, :string
    field :name, :string
    field :birth_date, :date
    timestamps()
  end

  def changeset(user, attrs) do
    user
    |> cast(attrs, [:email, :name, :birth_date])
    |> validate_required([:email, :name, :birth_date])
    |> validate_adult()
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
end
```

### Validación de Dominio de Email

```elixir
defp validate_email_domain(changeset) do
  validate_change(changeset, :email, fn :email, email ->
    allowed_domains = ["empresa.com", "ejemplo.com"]
    
    case String.split(email, "@") do
      [_name, domain] when domain in allowed_domains ->
        []
      
      [_name, domain] ->
        [email: "el dominio #{domain} no está permitido"]
      
      _ ->
        [email: "formato de email inválido"]
    end
  end)
end
```

### Validación de Contraseña Fuerte

```elixir
defp validate_strong_password(changeset) do
  validate_change(changeset, :password, fn :password, password ->
    errors = []
    
    errors = if String.length(password) < 8 do
      ["debe tener al menos 8 caracteres" | errors]
    else
      errors
    end
    
    errors = if not String.match?(password, ~r/[A-Z]/) do
      ["debe contener al menos una mayúscula" | errors]
    else
      errors
    end
    
    errors = if not String.match?(password, ~r/[a-z]/) do
      ["debe contener al menos una minúscula" | errors]
    else
      errors
    end
    
    errors = if not String.match?(password, ~r/[0-9]/) do
      ["debe contener al menos un número" | errors]
    else
      errors
    end
    
    if Enum.empty?(errors) do
      []
    else
      [password: Enum.join(errors, ", ")]
    end
  end)
end
```

### Validación Condicional

```elixir
def changeset(user, attrs) do
  user
  |> cast(attrs, [:type, :company_name, :tax_id])
  |> validate_required([:type])
  |> validate_company_fields()
end

defp validate_company_fields(changeset) do
  type = get_field(changeset, :type)
  
  if type == "business" do
    changeset
    |> validate_required([:company_name, :tax_id])
    |> validate_length(:tax_id, is: 9)
  else
    changeset
  end
end
```

---

## 5. Mejores Prácticas {#mejores-practicas}

### ✅ Hacer

1. **Validar en el servidor SIEMPRE**
```elixir
# Contexto
def create_user(attrs) do
  %User{}
  |> User.changeset(attrs)
  |> Repo.insert()
end
```

2. **Usar `to_form/2` en LiveView**
```elixir
def mount(_params, _session, socket) do
  {:ok, assign(socket, :form, to_form(changeset))}
end
```

3. **Acceder campos con `@form[:field]`**
```heex
<.input field={@form[:email]} type="email" />
```

4. **Dar IDs únicos a formularios**
```heex
<.form for={@form} id="user-form" phx-submit="save">
```

5. **Usar mensajes de error descriptivos**
```elixir
|> validate_format(:email, ~r/@/, message: "debe ser un email válido")
```

### ❌ No Hacer

1. **No confiar en validaciones del cliente**
```html
<!-- INCORRECTO: Solo HTML5 validation -->
<input type="email" required>
```

2. **No acceder al changeset en templates**
```heex
<!-- INCORRECTO -->
<%= @changeset.errors[:email] %>

<!-- CORRECTO -->
<.input field={@form[:email]} />
```

3. **No olvidar `phx-change` para validación en tiempo real**
```heex
<!-- INCORRECTO: Sin phx-change -->
<.form for={@form} phx-submit="save">

<!-- CORRECTO: Con phx-change -->
<.form for={@form} phx-change="validate" phx-submit="save">
```

4. **No usar `validate_number/3` con `:allow_nil`**
```elixir
# INCORRECTO: validate_number no soporta :allow_nil
|> validate_number(:age, greater_than: 0, allow_nil: true)

# CORRECTO: La validación solo corre si el campo tiene valor
|> validate_number(:age, greater_than: 0)
```

---

## Recursos Adicionales

- [Ecto.Changeset Documentation](https://hexdocs.pm/ecto/Ecto.Changeset.html)
- [Phoenix.Component.form/1](https://hexdocs.pm/phoenix_live_view/Phoenix.Component.html#form/1)
- [Phoenix LiveView Forms Guide](https://hexdocs.pm/phoenix_live_view/form-bindings.html)
