# RPC Básico en Elixir

Llamadas a funciones en nodos remotos.

## Llamada Básica

```elixir
:rpc.call(node, module, function, args)
```

## Wrapper Seguro

```elixir
defmodule MyApp.RemoteService do
  def call_remote(node, module, function, args) do
    case :rpc.call(node, module, function, args) do
      {:badrpc, reason} -> {:error, reason}
      result -> {:ok, result}
    end
  end
end
```

## Uso

```elixir
# Obtener datos de nodo remoto
{:ok, users} = RemoteService.call_remote(
  :"app@remote.host",
  MyApp.Accounts,
  :list_users,
  []
)
```

## GenServer para Sincronización

```elixir
defmodule MyApp.RemoteSync do
  use GenServer

  def init(opts) do
    schedule_sync()
    {:ok, %{remote_node: opts[:remote_node]}}
  end

  def handle_info(:sync, state) do
    case :rpc.call(state.remote_node, MyApp.Data, :get_latest, []) do
      {:badrpc, _} -> :error
      data ->
        Phoenix.PubSub.broadcast(MyApp.PubSub, "remote:sync", {:data, data})
    end

    schedule_sync()
    {:noreply, state}
  end

  defp schedule_sync do
    Process.send_after(self(), :sync, :timer.seconds(30))
  end
end
```
