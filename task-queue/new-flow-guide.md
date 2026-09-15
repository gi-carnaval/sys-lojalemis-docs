# Task Queue: Guia Para Criar Um Novo Fluxo

[Voltar ao índice](../task-queue-bootstrap.md)

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

## 3. Registre o job type

Atualize `task-queue/infrastructure/handlers/handler-registry.php`:

```php
'sync_product' => 'lemis_task_queue_handle_sync_product',
```

## 4. Registre o batch type

Atualize `task-queue/infrastructure/handlers/batch-hook-registry.php`:

```php
'sync_products' => array(
	'before_batch_start' => null,
	'after_batch_finish' => 'lemis_sync_products_after_batch_finish',
	'summary_callback' => 'lemis_task_queue_summarize_sync_products',
),
```

Se não precisar de finalização, use `after_batch_finish => null`.

Se o resumo genérico for suficiente, omita `summary_callback` ou use `summary_callback => null`.

Um `summary_callback` recebe o batch e os jobs:

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

## 5. Crie uma função iniciadora

Essa função pode ser chamada por uma rota REST, página admin ou ação interna.

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

## 6. Exponha progresso para a interface

Uma tela pode reutilizar as rotas genéricas:

- `GET /wp-json/lemis/v1/job-batches/{id}`
- `GET /wp-json/lemis/v1/job-batches/{id}/summary`

Se o resumo genérico não servir, registre um `summary_callback` para o `batch_type`.

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
- A UI consulta a rota de progresso.
- O worker está rodando.
- Arquivos PHP alterados passaram por `php -l`.
