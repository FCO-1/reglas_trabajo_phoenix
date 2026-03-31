# Testing de Notificaciones

Guía para escribir tests completos del sistema de notificaciones.

---

## Tabla de Contenidos

1. [Testing del Contexto](#testing-contexto)
2. [Testing de LiveView](#testing-liveview)
3. [Testing de PubSub](#testing-pubsub)
4. [Testing con Presence](#testing-presence)

---

## 1. Testing del Contexto {#testing-contexto}

```elixir
# test/my_app/notifications_test.exs
defmodule MyApp.NotificationsTest do
  use MyApp.DataCase
  import MyApp.Factory

  alias MyApp.Notifications

  describe "create_notification/1" do
    test "crea notificación con datos válidos" do
      user = insert(:user)
      actor = insert(:user)

      attrs = %{
        type: "mention",
        title: "Te mencionaron",
        body: "Juan te mencionó",
        user_id: user.id,
        actor_id: actor.id
      }

      assert {:ok, notification} = Notifications.create_notification(attrs)
      assert notification.type == "mention"
      assert notification.user_id == user.id
      assert notification.actor_id == actor.id
      assert notification.read == false
    end

    test "falla con datos inválidos" do
      assert {:error, changeset} = Notifications.create_notification(%{})
      assert "can't be blank" in errors_on(changeset).type
      assert "can't be blank" in errors_on(changeset).title
      assert "can't be blank" in errors_on(changeset).user_id
    end

    test "valida tipo de notificación" do
      user = insert(:user)

      attrs = %{
        type: "invalid_type",
        title: "Test",
        user_id: user.id
      }

      assert {:error, changeset} = Notifications.create_notification(attrs)
      assert "is invalid" in errors_on(changeset).type
    end
  end

  describe "list_notifications/2" do
    test "lista notificaciones del usuario" do
      user1 = insert(:user)
      user2 = insert(:user)
      
      notif1 = insert(:notification, user: user1)
      notif2 = insert(:notification, user: user1)
      _notif3 = insert(:notification, user: user2)

      notifications = Notifications.list_notifications(user1.id)

      assert length(notifications) == 2
      assert notif1.id in Enum.map(notifications, & &1.id)
      assert notif2.id in Enum.map(notifications, & &1.id)
    end

    test "filtra solo no leídas" do
      user = insert(:user)
      
      unread = insert(:notification, user: user, read: false)
      _read = insert(:notification, user: user, read: true)

      notifications = Notifications.list_notifications(user.id, unread_only: true)

      assert length(notifications) == 1
      assert hd(notifications).id == unread.id
    end

    test "respeta límite de resultados" do
      user = insert(:user)
      insert_list(10, :notification, user: user)

      notifications = Notifications.list_notifications(user.id, limit: 5)

      assert length(notifications) == 5
    end
  end

  describe "mark_as_read/2" do
    test "marca notificación como leída" do
      user = insert(:user)
      notification = insert(:notification, user: user, read: false)

      assert {:ok, updated} = Notifications.mark_as_read(notification.id, user.id)
      assert updated.read == true
      assert updated.read_at != nil
    end

    test "no permite marcar notificación de otro usuario" do
      user1 = insert(:user)
      user2 = insert(:user)
      notification = insert(:notification, user: user1)

      assert {:error, :not_found} = Notifications.mark_as_read(notification.id, user2.id)
    end
  end

  describe "mark_all_as_read/1" do
    test "marca todas las notificaciones como leídas" do
      user = insert(:user)
      insert_list(3, :notification, user: user, read: false)

      assert {:ok, 3} = Notifications.mark_all_as_read(user.id)
      
      notifications = Notifications.list_notifications(user.id, unread_only: true)
      assert notifications == []
    end
  end

  describe "count_unread/1" do
    test "cuenta notificaciones no leídas" do
      user = insert(:user)
      insert_list(3, :notification, user: user, read: false)
      insert_list(2, :notification, user: user, read: true)

      assert Notifications.count_unread(user.id) == 3
    end
  end
end
```

---

## 2. Testing de LiveView {#testing-liveview}

```elixir
# test/my_app_web/live/notifications_live_test.exs
defmodule MyAppWeb.NotificationsLiveTest do
  use MyAppWeb.ConnCase
  import Phoenix.LiveViewTest
  import MyApp.Factory

  describe "Index" do
    setup :register_and_log_in_user

    test "muestra notificaciones del usuario", %{conn: conn, user: user} do
      notification = insert(:notification, user: user, title: "Test Notification")

      {:ok, _view, html} = live(conn, ~p"/notifications")

      assert html =~ "Notificaciones"
      assert html =~ "Test Notification"
    end

    test "muestra contador de no leídas", %{conn: conn, user: user} do
      insert_list(3, :notification, user: user, read: false)

      {:ok, _view, html} = live(conn, ~p"/notifications")

      assert html =~ "3"
    end

    test "permite marcar como leída", %{conn: conn, user: user} do
      notification = insert(:notification, user: user, read: false)

      {:ok, view, _html} = live(conn, ~p"/notifications")

      assert view
             |> element("#notification-#{notification.id} button", "Marcar leída")
             |> render_click()

      updated = Repo.get!(Notification, notification.id)
      assert updated.read == true
    end

    test "permite marcar todas como leídas", %{conn: conn, user: user} do
      insert_list(3, :notification, user: user, read: false)

      {:ok, view, _html} = live(conn, ~p"/notifications")

      assert view
             |> element("button", "Marcar todas como leídas")
             |> render_click()

      assert Notifications.count_unread(user.id) == 0
    end

    test "no muestra notificaciones de otros usuarios", %{conn: conn, user: user} do
      other_user = insert(:user)
      _other_notification = insert(:notification, user: other_user, title: "Other Notification")

      {:ok, _view, html} = live(conn, ~p"/notifications")

      refute html =~ "Other Notification"
    end
  end
end
```

---

## 3. Testing de PubSub {#testing-pubsub}

```elixir
# test/my_app/notifications_test.exs (continuación)
describe "broadcasting" do
  test "hace broadcast al crear notificación" do
    user = insert(:user)
    
    # Suscribirse antes de crear
    :ok = Notifications.subscribe(user.id)

    attrs = %{
      type: "mention",
      title: "Test",
      user_id: user.id
    }

    {:ok, notification} = Notifications.create_notification(attrs)

    # Verificar que recibimos el broadcast
    assert_receive {:new_notification, ^notification}
  end

  test "hace broadcast al marcar como leída" do
    user = insert(:user)
    notification = insert(:notification, user: user, read: false)
    
    :ok = Notifications.subscribe(user.id)

    {:ok, _} = Notifications.mark_as_read(notification.id, user.id)

    assert_receive {:notification_read, notification_id}
    assert notification_id == notification.id
  end

  test "hace broadcast al marcar todas como leídas" do
    user = insert(:user)
    insert_list(3, :notification, user: user, read: false)
    
    :ok = Notifications.subscribe(user.id)

    {:ok, _} = Notifications.mark_all_as_read(user.id)

    assert_receive :all_notifications_read
  end
end
```

---

## 4. Testing con Presence {#testing-presence}

```elixir
# test/my_app/notifications_test.exs (continuación)
describe "notificaciones con presence" do
  test "envía broadcast solo si usuario está online" do
    user = insert(:user)
    
    # Usuario offline
    attrs = %{
      type: "mention",
      title: "Test",
      user_id: user.id
    }

    # Suscribirse pero el usuario no está tracked
    :ok = Notifications.subscribe(user.id)

    {:ok, _notification} = Notifications.create_and_notify(attrs)

    # No debe recibir broadcast porque no está online
    refute_receive {:new_notification, _}
  end

  test "carga notificaciones pendientes al conectarse" do
    user = insert(:user)
    
    # Crear notificaciones mientras está offline
    insert_list(3, :notification, user: user, read: false)

    # Simular conexión y carga de pendientes
    pending = Notifications.load_pending_notifications(user.id)

    assert length(pending) == 3
    assert Enum.all?(pending, &(!&1.read))
  end
end
```

### Factory para Tests

```elixir
# test/support/factory.ex
defmodule MyApp.Factory do
  use ExMachina.Ecto, repo: MyApp.Repo

  def user_factory do
    %MyApp.Accounts.User{
      name: sequence(:name, &"User #{&1}"),
      email: sequence(:email, &"user#{&1}@example.com")
    }
  end

  def notification_factory do
    %MyApp.Notifications.Notification{
      type: "mention",
      title: sequence(:title, &"Notification #{&1}"),
      body: "Test notification body",
      read: false,
      data: %{},
      user: build(:user),
      actor: build(:user)
    }
  end
end
```

### Helper para Tests de LiveView

```elixir
# test/support/conn_case.ex
defmodule MyAppWeb.ConnCase do
  use ExUnit.CaseTemplate

  using do
    quote do
      import MyAppWeb.ConnCase
      import MyApp.Factory

      def register_and_log_in_user(%{conn: conn}) do
        user = insert(:user)
        %{conn: log_in_user(conn, user), user: user}
      end

      defp log_in_user(conn, user) do
        token = MyApp.Accounts.generate_user_session_token(user)

        conn
        |> Phoenix.ConnTest.init_test_session(%{})
        |> Plug.Conn.put_session(:user_token, token)
      end
    end
  end
end
```

---

## Mejores Prácticas

✅ **Testear contexto completo** - Todas las funciones públicas  
✅ **Testear broadcasts** - Verificar PubSub funciona  
✅ **Testear UI** - Interacciones en LiveView  
✅ **Usar factories** - Datos consistentes en tests  
✅ **Testear edge cases** - Notificaciones de otros usuarios, etc

❌ **No testear implementación** - Testear comportamiento  
❌ **No tests flaky** - Usar `assert_receive` con timeout adecuado  
❌ **No olvidar limpiar** - Sandbox de Ecto limpia automáticamente

---

## Recursos

- Anterior: [Notificaciones con Presence](03_notificaciones_presence.md)
- [ExUnit Docs](https://hexdocs.pm/ex_unit)
- [Phoenix LiveViewTest](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveViewTest.html)
