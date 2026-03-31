# Guía Completa de Trabajo con Phoenix Framework

Documentación completa sobre mejores prácticas para trabajar con Phoenix, LiveView, PubSub, Presence, RPC y más.

---

## 📚 Índice de Contenidos

### 1. Validaciones
- [Validaciones en Phoenix y LiveView](validaciones/01_validaciones.md)
  - Validaciones del lado del servidor con Ecto.Changeset
  - Validaciones en tiempo real con `phx-change`
  - Templates HTML con `to_form/2`, `<.form>` y `<.input>`
  - Validaciones personalizadas

### 2. Testing
- [Testing con Datos Relacionados (FK)](testing/01_testing_datos_relacionados.md)
  - Configuración de ExMachina
  - Factories con relaciones FK
  - Testing de contextos con datos relacionados
  - Helpers para testing

- [Testing de Formularios LiveView](testing/02_testing_liveview_forms.md)
  - Setup básico de tests
  - Testing de validaciones en tiempo real
  - Testing de submit y errores
  - Testing con Floki

### 3. Phoenix Presence
- [Configuración de Presence](presence/01_presence_setup.md)
  - Instalación y configuración
  - Tracking básico de usuarios
  - Presence en múltiples nodos
  - Mejores prácticas

### 4. Recuperación de Estado
- [Recuperación de Estado en LiveView](recuperacion_estado/01_recuperacion_estado.md)
  - El problema de la recarga
  - Soluciones con localStorage y JS Hooks
  - Guardar estado en assigns
  - Temporary assigns

### 5. Notificaciones Push
- [Notificaciones Push en Tiempo Real](notificaciones/01_notificaciones_push.md)
  - Schema de notificaciones
  - Contexto con broadcasting
  - LiveView con notificaciones
  - Notificaciones con Presence

### 6. PubSub Multi-Nodo
- [PubSub en Múltiples Nodos](pubsub/01_pubsub_multinodo.md)
  - Configuración de PubSub.PG2
  - Configurar nodos para clustering
  - Broadcasting entre nodos
  - Testing multi-nodo

### 7. RPC y Servicios Remotos
- [RPC Básico en Elixir](rpc/01_rpc_basico.md)
  - Llamadas RPC básicas
  - Wrapper seguro para RPC
  - GenServer para sincronización
  - Consumo de servicios remotos

---

## 🚀 Inicio Rápido

### Validaciones en LiveView

```elixir
def handle_event("validate", %{"user" => params}, socket) do
  changeset = User.changeset(%User{}, params)
  {:noreply, assign(socket, form: to_form(changeset))}
end
```

### Presence para Tracking de Usuarios

```elixir
if connected?(socket) do
  Presence.track(self(), "room:lobby", user.id, %{
    username: user.name,
    online_at: System.system_time(:second)
  })
end
```

### Notificaciones en Tiempo Real

```elixir
def create_notification(attrs) do
  with {:ok, notif} <- Repo.insert(changeset) do
    PubSub.broadcast(MyApp.PubSub, "notifications:#{notif.user_id}", 
      {:new_notification, notif})
    {:ok, notif}
  end
end
```

---

## 📖 Mejores Prácticas Generales

### Validaciones
✅ SIEMPRE validar en el servidor con `Ecto.Changeset`  
✅ Usar `phx-change` para validaciones en tiempo real  
✅ Usar `to_form/2` en LiveView, `<.form>` y `<.input>` en templates  
❌ Nunca acceder al changeset directamente en templates

### Testing
✅ Usar ExMachina para factories con relaciones FK  
✅ Usar `Phoenix.LiveViewTest` para LiveViews  
✅ Usar `element/2` y `has_element?/2` en lugar de HTML crudo  
✅ Dar IDs únicos a elementos clave para testing

### Presence
✅ Track solo cuando `connected?(socket)`  
✅ Usar `sync_diff` para actualizaciones  
✅ Mantener metadata mínima  
❌ No trackear en mount sin verificar conexión

### PubSub Multi-Nodo
✅ Usar `Phoenix.PubSub.PG2` para distribuir mensajes  
✅ Suscribirse en `mount/3` cuando `connected?(socket)`  
✅ Hacer broadcast después de operaciones exitosas  
✅ Configurar nombres de nodo y cookies para clustering

---

## 🔧 Configuración de Proyecto

### Dependencias Comunes

```elixir
# mix.exs
defp deps do
  [
    {:phoenix, "~> 1.7.0"},
    {:phoenix_live_view, "~> 0.20.0"},
    {:ex_machina, "~> 2.7", only: :test},
    {:floki, ">= 0.30.0", only: :test}
  ]
end
```

### Generar Presence

```bash
mix phx.gen.presence
```

### Configurar Nodos (Producción)

```elixir
# config/runtime.exs
if config_env() == :prod do
  node_name = System.get_env("NODE_NAME") || "app"
  node_host = System.get_env("POD_IP") || "127.0.0.1"
  
  :net_kernel.start([:"#{node_name}@#{node_host}"])
  Node.set_cookie(:my_app_cookie)
end
```

---

## 📚 Recursos Adicionales

- [Phoenix LiveView Docs](https://hexdocs.pm/phoenix_live_view)
- [Ecto Changeset Docs](https://hexdocs.pm/ecto/Ecto.Changeset.html)
- [Phoenix PubSub Docs](https://hexdocs.pm/phoenix_pubsub)
- [Phoenix Presence Docs](https://hexdocs.pm/phoenix/Phoenix.Presence.html)
- [Distributed Elixir](https://elixir-lang.org/getting-started/mix-otp/distributed-tasks.html)

---

## 🤝 Contribuciones

Esta documentación cubre los patrones más comunes y mejores prácticas para trabajar con Phoenix Framework.

**Temas cubiertos:**
- ✅ Validaciones en LiveView
- ✅ Testing (datos relacionados + formularios)
- ✅ Phoenix Presence (tracking de usuarios)
- ✅ Recuperación de estado
- ✅ Notificaciones push en tiempo real
- ✅ PubSub multi-nodo
- ✅ RPC y servicios remotos

---

**Última actualización:** 2024

