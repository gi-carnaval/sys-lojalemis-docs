# Task Queue: Guia Para Criar Um Novo Fluxo

[Voltar ao índice](task-queue-bootstrap.md)

Para fluxos novos, prefira registrar tudo com `lemis_task_queue_register_flow()` em `task-queue/config/flows.php`. Os registries manuais de handler e batch continuam como fallback para fluxos legados.

## 1. Defina a unidade de trabalho

Escolha o menor item que pode ser processado de forma independente.

Bons exemplos:

- um pedido
- um produto
- uma imagem
- um cliente
- uma página de relatório

Evite criar um job que processa muitos itens internamente, porque isso reduz a utilidade do progresso, do retry e da recuperação.

## 2. Crie o handler

Adicione a função em `task-queue/infrastructure/handlers/handlers.php` ou em um arquivo carregado antes do registry ser usado.

```php
function lemis_task_queue_handle_sync_product(array $payload): array
{
	$product_id = (int) ($payload['product_id'] ?? 0);

	if ($product_id <= 0) {
		throw new RuntimeException('Produto inválido.');
	}

	lemis_sync_product($product_id);

	return array(
		'success' => true,
		'product_id' => $product_id,
	);
}
```

## 3. Registre o flow

Atualize `task-queue/config/flows.php`:

```php
lemis_task_queue_register_flow(
	array(
		'slug' => 'sync-products',
		'route' => 'sync-products',
		'batch_type' => 'sync_products',
		'handler' => 'sync_product',
		'item_key' => 'product_id',
		'prepare_callback' => 'lemis_sync_products_prepare_flow',
		'job_callback' => 'lemis_task_queue_handle_sync_product',
		'summary_callback' => 'lemis_task_queue_summarize_sync_products',
	)
);
```

O `handler` e o `batch_type` passam a ser descobertos pelos registries a partir do flow. So adicione entradas manuais em `handler-registry.php` ou `batch-hook-registry.php` quando estiver mantendo um fluxo legado que ainda nao usa `lemis_task_queue_register_flow()`.

## 4. Crie o prepare callback

O `prepare_callback` recebe a requisicao inicial e retorna os itens que viram jobs:

```php
function lemis_sync_products_prepare_flow(WP_REST_Request $request): array
{
	$product_ids = array_map('absint', (array) $request->get_param('product_ids'));
	$product_ids = array_values(array_filter($product_ids));

	return array(
		'success' => true,
		'items' => $product_ids,
		'item_key' => 'product_id',
	);
}
```

Se precisar enviar dados comuns para todos os jobs, retorne `job_payload`:

```php
return array(
	'success' => true,
	'items' => $product_ids,
	'job_payload' => array('mode' => 'live'),
	'item_key' => 'product_id',
);
```

## 5. Crie o summary callback quando necessario

Se o resumo generico for suficiente, use `summary_callback => null`. Um `summary_callback` recebe o batch e os jobs:

```php
function lemis_task_queue_summarize_sync_products(
	array $batch,
	array $jobs
): array {
	return array(
		'batch_id' => (int) $batch['id'],
		'type' => $batch['type'],
		'total' => count($jobs),
		'products' => array(),
	);
}
```

## 6. Inicie o flow pela rota generica

A rota generica usa o `slug` do flow:

```text
POST /wp-json/lemis/v1/task-queue/flows/sync-products/start
```

Ela chama `lemis_task_queue_start_flow()`, executa o `prepare_callback`, cria o batch e retorna `batch_id`, `jobs_created` e `status`.

Use uma funcao iniciadora propria apenas quando precisar preservar um contrato legado:

```php
function lemis_start_product_sync(array $product_ids): array
{
	$batch_id = task_queue_create_batch('sync_products', 0);
	$jobs_created = 0;
	$errors = array();

	foreach ($product_ids as $product_id) {
		try {
			lemis_task_queue_create_job(
				'sync_product',
				array('product_id' => (int) $product_id),
				$batch_id
			);

			$jobs_created++;
		} catch (Throwable $error) {
			$errors[] = array(
				'product_id' => $product_id,
				'error' => $error->getMessage(),
			);
		}
	}

	task_queue_set_batch_total_jobs($batch_id, $jobs_created);

	if ($jobs_created === 0) {
		task_queue_complete_pending_batch($batch_id);
	} else {
		task_queue_start_batch($batch_id);
	}

	return array(
		'success' => true,
		'batch_id' => $batch_id,
		'jobs_created' => $jobs_created,
		'errors' => $errors,
		'status' => $jobs_created === 0 ? 'completed' : 'processing',
	);
}
```

## 7. Exponha progresso para a interface

Uma tela pode reutilizar as rotas genéricas:

- `GET /wp-json/lemis/v1/job-batches/{id}`
- `GET /wp-json/lemis/v1/job-batches/{id}/summary`

Se o resumo genérico não servir, registre um `summary_callback` para o `batch_type`.

## 8. Reutilize o frontend de progresso

Para páginas que precisam iniciar um batch e acompanhar progresso, prefira `lemis_task_queue_enqueue_flow()` e `lemis_task_queue_render_progress_panel()`.

```php
lemis_task_queue_enqueue_flow(
	'sync-products',
	array(
		'form_id' => 'sync_products_form',
		'id_prefix' => 'sync-products',
	)
);
```

```php
<form id="sync_products_form">
	<input type="hidden" name="product_ids[]" value="123">
	<button type="submit" id="sync-products-start">
		Sincronizar produtos
	</button>
</form>

<?php
lemis_task_queue_render_progress_panel(
	array(
		'id_prefix' => 'sync-products',
		'title' => 'Acompanhamento dos produtos',
	)
);
?>
```

Para telas com comportamento especifico, ainda e possivel criar um script de pagina pequeno que use `batch-progress.js`.

Registre o script em `task-queue/assets/enqueue.php`:

```php
'sync_products_page' => array(
	'handle' => 'lemis-sync-products-page',
	'file' => 'sync-products-page.js',
	'dependencies' => array(
		'lemis-batch-progress',
	),
),
```

Enfileire na página:

```php
lemis_task_queue_enqueue(
	array(
		'script' => 'sync_products_page',
	)
);
```

No JavaScript da página:

```js
var batchProgress = window.createBatchProgress({
  restUrl: window.lemisTaskQueue.restUrl,
  nonce: window.lemisTaskQueue.nonce,
  api: {
    start: 'sync-products',
    progress: function (batchId) {
      return 'job-batches/' + batchId;
    },
    summary: function (batchId) {
      return 'job-batches/' + batchId + '/summary';
    }
  },
  elements: {
    panel: 'sync-products-progress-panel',
    status: 'sync-products-status',
    progressBar: 'sync-products-progress-bar',
    progressText: 'sync-products-progress-text',
    startButton: 'sync-products-start'
  }
});
```

O endpoint `api.start` precisa retornar pelo menos `batch_id`, `jobs_created` e `status`.

## Boas práticas

- Use IDs e opções simples no payload.
- Inclua no payload tudo que o handler precisa para rodar sem depender da requisição original.
- Mantenha o handler idempotente quando possível.
- Evite efeitos colaterais antes de validações importantes.
- Registre no `result` as informações que alguém precisará para auditar o job.
- Use `LemisTaskQueueRetryableException` para indisponibilidade temporária de API, timeout externo ou limite transitório.
- Use falha definitiva para payload inválido, regra de negócio inválida ou handler inexistente.
- Crie batches com `total_jobs = 0` quando a criação dos jobs puder falhar parcialmente.
- Chame `task_queue_start_batch()` só depois de criar os jobs e ajustar `total_jobs`.
- Registre `summary_callback` quando o domínio precisar de campos próprios no resumo.
- Use `lemis_task_queue_enqueue()` para páginas que precisam dos assets frontend do módulo.
- Garanta que o worker esteja rodando no ambiente em que a fila precisa andar.

## Limitações conhecidas

- O registry de handlers é manual.
- O registry de hooks de batch é manual.
- As permissões REST atuais são administrativas.
- O resumo específico de pedidos existe apenas nos batch types de sincronização; outros batch types usam fallback genérico.
- Não existe painel genérico para listar todos os batches ou jobs.
- O worker roda em loop infinito e precisa ser gerenciado pelo ambiente.
- Jobs sem worker ativo permanecem em `pending`.
- Um handler que retorna dados não serializáveis pode gerar um `result` de erro de serialização.

## Checklist

Antes de considerar um novo fluxo pronto:

- O `task-queue/bootstrap.php` é carregado pelo tema.
- O schema foi instalado no banco local.
- Existe um `batch_type` registrado em `batch-hook-registry.php`.
- O `batch_type` tem `summary_callback` quando precisa de resumo próprio.
- Existe um `job_type` registrado em `handler-registry.php`.
- O handler é `callable`.
- O payload contém somente dados serializáveis.
- O handler valida o payload.
- O handler retorna um array serializável.
- A função iniciadora cria batch, cria jobs, ajusta `total_jobs` e inicia/finaliza o batch.
- O endpoint ou tela retorna `batch_id`.
- A UI consulta a rota de progresso ou usa `createBatchProgress()`.
- O script de página foi registrado em `lemis_task_queue_get_frontend_scripts()` quando usa assets do módulo.
- O worker está rodando.
- Arquivos PHP alterados passaram por `php -l`.
