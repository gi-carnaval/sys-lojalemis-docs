# Task Queue: Rotas REST e Fluxo de Sincronização de Pedidos

[Voltar ao índice](../task-queue-bootstrap.md)

## Rotas REST

As rotas são registradas em `rest_api_init` por:

`wordpress/wp-content/themes/siteorigin-corp/task-queue/api/module.php`

Todas as rotas atuais usam `lemis_task_queue_can_read_batch()`, que exige:

```php
current_user_can('manage_options')
```

Portanto, o uso atual é administrativo.

## Progresso do batch

```http
GET /wp-json/lemis/v1/job-batches/{id}
```

Resposta de sucesso:

```json
{
  "success": true,
  "data": {
    "id": 123,
    "type": "sync_bling_orders_dry_run",
    "status": "processing",
    "total_jobs": 10,
    "completed_jobs": 4,
    "failed_jobs": 1,
    "pending_jobs": 4,
    "processing_jobs": 1,
    "finished_jobs": 5,
    "progress": 50,
    "created_at": "2026-09-15 10:00:00",
    "finished_at": null
  }
}
```

## Resumo do batch

```http
GET /wp-json/lemis/v1/job-batches/{id}/summary
```

Resposta de sucesso no fluxo de pedidos:

```json
{
  "success": true,
  "data": {
    "batch_id": 123,
    "total": 10,
    "mode": "dry_run",
    "is_dry_run": true,
    "created": 0,
    "would_create": 7,
    "existing": 2,
    "ignored": 1,
    "failed": 0,
    "sales": []
  }
}
```

## Início da sincronização de pedidos

```http
POST /wp-json/lemis/v1/sync-pedidos
Content-Type: application/json
X-WP-Nonce: {nonce}

{
  "fetch_limit": 10,
  "mode": "dry_run"
}
```

Resposta de sucesso:

```json
{
  "success": true,
  "data": {
    "success": true,
    "batch_id": 123,
    "total_jobs_planned": 10,
    "jobs_created": 10,
    "errors": [],
    "status": "processing"
  }
}
```

## Fluxo atual de sincronização

Entrada:

- A página enfileira a sincronização via JavaScript em `task-queue/assets/js/sync-page.js`.
- O JavaScript chama `POST /wp-json/lemis/v1/sync-pedidos`.
- Depois faz polling em `GET /wp-json/lemis/v1/job-batches/{id}`.
- Ao concluir, consulta `GET /wp-json/lemis/v1/job-batches/{id}/summary`.

Função que cria o batch:

`lemis_sync_get_orders_id_to_sync()` em `wordpress/wp-content/themes/siteorigin-corp/functions.php`

Tipos usados:

- Batch `sync_bling_orders_dry_run` com jobs `sync_bling_order_dry_run`
- Batch `sync_bling_orders_live` com jobs `sync_bling_order`

Payload por job:

```php
array(
	'order_id' => $order_id,
	'fetch_limit' => $options['fetch_limit'],
)
```

Handlers:

- `lemis_task_queue_handle_sync_bling_order()`
- `lemis_task_queue_handle_sync_bling_order_dry_run()`

Ambos chamam `lemis_sync_bling_order($order_id, $options)`.

## Contrato do resumo de pedidos

`task_queue_get_batch_summary()` interpreta o `result` dos jobs como resultado de sincronização de pedidos.

Chaves relevantes no `result`:

- `success`: quando `false`, conta como falha.
- `existing_venda_id`: conta como venda já existente.
- `created_venda_id`: conta como venda criada em modo real.
- `decision`: quando igual a `would_create_sale`, conta como venda que seria criada no `dry_run`.
- `pedido_id`: usado no resumo de venda.
- `platform`: plataforma do pedido.
- `save_preview`: contém `meta` e `produtos` usados para montar o resumo.
- `reasons`: motivos de decisão ou ignorar.

Formato esperado de `save_preview`:

```php
array(
	'meta' => array(
		'cliente' => 'Nome do cliente',
		'pagamento' => 100.00,
		'subtotal' => 90.00,
		'frete' => 10.00,
		'taxas' => 0.00,
		'plataforma' => 'bling',
	),
	'produtos' => array(
		array(
			'qtd' => 1,
			'codigo' => 'SKU-1',
			'nome' => 'Produto',
			'valor_un' => 90.00,
		),
	),
)
```

Para fluxos que não sejam de pedidos/vendas, não force esse contrato. Prefira criar uma função de resumo específica.
