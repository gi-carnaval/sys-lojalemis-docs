# Task Queue: API Pública e Contratos

[Voltar ao índice](../task-queue-bootstrap.md)

## Criar e iniciar batches

Estas são as funções que um novo fluxo normalmente usa:

- `task_queue_create_batch(string $type, int $total_jobs): int`
- `task_queue_set_batch_total_jobs(int $batch_id, int $total_jobs): bool`
- `task_queue_start_batch(int $batch_id): bool`
- `task_queue_complete_pending_batch(int $batch_id): bool`
- `lemis_task_queue_create_job(string $type, array $payload = array(), ?int $batch_id = null): int`

Exemplo:

```php
$batch_id = task_queue_create_batch('meu_batch_type', 0);

foreach ($items as $item) {
	lemis_task_queue_create_job(
		'meu_job_type',
		array('item_id' => (int) $item['id']),
		$batch_id
	);
}

task_queue_set_batch_total_jobs($batch_id, count($items));

if (count($items) === 0) {
	task_queue_complete_pending_batch($batch_id);
} else {
	task_queue_start_batch($batch_id);
}
```

O padrão atual cria o batch com `total_jobs = 0`, tenta criar todos os jobs e depois ajusta o total para a quantidade realmente criada. Isso evita que o batch fique esperando jobs que falharam na criação.

## Consultar andamento

```php
$progress = task_queue_get_batch_progress($batch_id);
```

Retorno:

```php
array(
	'id' => 123,
	'type' => 'sync_bling_orders_dry_run',
	'status' => 'processing',
	'total_jobs' => 10,
	'completed_jobs' => 4,
	'failed_jobs' => 1,
	'pending_jobs' => 4,
	'processing_jobs' => 1,
	'finished_jobs' => 5,
	'progress' => 50,
	'created_at' => '2026-09-15 10:00:00',
	'finished_at' => null,
)
```

Observações:

- `progress` é calculado por `(completed_jobs + failed_jobs) / total_jobs`.
- Batch sem jobs tem progresso `100`.
- `pending_jobs` e `processing_jobs` vêm da tabela de jobs.

## Consultar resumo

```php
$summary = task_queue_get_batch_summary($batch_id);
```

O resumo atual é orientado ao domínio de sincronização de pedidos. Ele lê o `result` JSON dos jobs e agrupa vendas criadas, vendas que seriam criadas em `dry_run`, vendas existentes, ignoradas e falhas.

Para outros domínios, use `task_queue_get_batch_progress()` para acompanhamento genérico e crie um resumo próprio se os dados não tiverem a mesma semântica de pedidos/vendas.

## Registry de handlers

Arquivo:

`wordpress/wp-content/themes/siteorigin-corp/task-queue/infrastructure/handlers/handler-registry.php`

Função:

```php
function lemis_task_queue_get_handler(string $type): ?callable
```

Exemplo atual:

```php
$handlers = array(
	'sync_bling_order' => 'lemis_task_queue_handle_sync_bling_order',
	'sync_bling_order_dry_run' => 'lemis_task_queue_handle_sync_bling_order_dry_run',
);
```

Para adicionar um tipo novo:

```php
$handlers = array(
	'sync_bling_order' => 'lemis_task_queue_handle_sync_bling_order',
	'sync_bling_order_dry_run' => 'lemis_task_queue_handle_sync_bling_order_dry_run',
	'meu_job_type' => 'lemis_task_queue_handle_meu_job',
);
```

O handler precisa ser `callable`. Se o tipo não existir ou apontar para uma função inexistente, o worker marca o job como `failed` com a mensagem `Handler nao encontrado`.

## Registry de hooks de batch

Arquivo:

`wordpress/wp-content/themes/siteorigin-corp/task-queue/infrastructure/handlers/batch-hook-registry.php`

Função:

```php
function lemis_task_queue_get_batch_definition(string $batch_type): ?array
```

Exemplo atual:

```php
$definitions = array(
	'sync_bling_orders_dry_run' => array(
		'before_batch_start' => null,
		'after_batch_finish' => 'lemis_sync_pedidos_after_batch_finish',
	),
	'sync_bling_orders_live' => array(
		'before_batch_start' => null,
		'after_batch_finish' => 'lemis_sync_pedidos_after_batch_finish',
	),
);
```

Hooks disponíveis:

- `before_batch_start`: executado por `task_queue_start_batch()` antes de mudar o batch de `pending` para `processing`.
- `after_batch_finish`: executado quando o batch chega a `completed`.

O hook `after_batch_finish` roda dentro de um `try/catch`; se falhar, o erro é enviado ao `error_log`, mas o batch continua finalizado.

## Contrato do payload

O `payload` é salvo como JSON na tabela de jobs.

Regras práticas:

- Use apenas dados serializáveis.
- Prefira IDs, strings, números, booleans e arrays simples.
- Inclua tudo que o handler precisa para executar sem depender da requisição original.
- Não use closures, objetos complexos, recursos PHP ou estado implícito.

## Contrato do handler

Um handler recebe o payload decodificado e deve retornar um array serializável em JSON.

Exemplo:

```php
function lemis_task_queue_handle_meu_job(array $payload): array
{
	$item_id = (int) ($payload['item_id'] ?? 0);

	if ($item_id <= 0) {
		throw new RuntimeException('Item inválido.');
	}

	$result = meu_servico_processar_item($item_id);

	return array(
		'success' => true,
		'item_id' => $item_id,
		'processed_id' => $result['id'] ?? null,
	);
}
```

Regras práticas:

- Valide o payload no início do handler.
- Processe uma única unidade de trabalho por job.
- Retorne arrays simples, com strings, números, booleans, `null` e arrays aninhados simples.
- Inclua dados de negócio úteis para auditoria e resumo.
- Lance `LemisTaskQueueRetryableException` apenas para falhas temporárias.
- Lance `RuntimeException` ou outra exceção para falhas definitivas.
