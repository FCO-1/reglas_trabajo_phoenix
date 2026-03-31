# Recuperación de Estado en LiveView

Guía para manejar la recuperación de estado cuando LiveView se recarga.

## El Problema

Cuando un usuario está inactivo por mucho tiempo, LiveView puede desconectarse y reconectarse, perdiendo el estado del formulario.

## Solución 1: localStorage con JS Hook

```javascript
// assets/js/app.js
let Hooks = {}

Hooks.FormPersist = {
  mounted() {
    const formId = this.el.id
    const saved = localStorage.getItem(`form_${formId}`)
    
    if (saved) {
      const data = JSON.parse(saved)
      Object.keys(data).forEach(key => {
        const input = this.el.querySelector(`[name="${key}"]`)
        if (input) input.value = data[key]
      })
    }

    this.el.addEventListener('input', (e) => {
      const formData = new FormData(this.el)
      const data = Object.fromEntries(formData)
      localStorage.setItem(`form_${formId}`, JSON.stringify(data))
    })
  }
}

let liveSocket = new LiveSocket("/live", Socket, {
  params: {_csrf_token: csrfToken},
  hooks: Hooks
})
```

## Uso en Template

```heex
<.form
  for={@form}
  id="product-form"
  phx-hook="FormPersist"
  phx-submit="save"
>
  <.input field={@form[:name]} />
  <.input field={@form[:price]} />
</.form>
```

## Solución 2: Guardar en Assigns

```elixir
def handle_event("validate", %{"product" => params}, socket) do
  changeset = Product.changeset(%Product{}, params)
  
  {:noreply,
   socket
   |> assign(:form, to_form(changeset))
   |> assign(:form_data, params)}  # Guardar params
end

def mount(_params, _session, socket) do
  # Recuperar de session o asignar vacío
  form_data = get_session(socket, :form_data) || %{}
  
  {:ok, assign(socket, :form_data, form_data)}
end
```

## Solución 3: Temporary Assigns

```elixir
def mount(_params, _session, socket) do
  {:ok,
   socket
   |> assign(:form, to_form(%{}))
   |> assign(:temp_data, nil)
   |> put_temporary_assigns(temp_data: nil)}
end
```
