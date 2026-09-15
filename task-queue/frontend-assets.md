# Task Queue: Assets e Progresso no Frontend

[Voltar ao índice](task-queue-bootstrap.md)

## Objetivo

O frontend do módulo foi separado em duas camadas:

- `batch-progress.js`: componente genérico para iniciar batches, fazer polling de progresso e buscar resumo.
- `sync-page-test.js`: adaptador da tela de simulação de sincronização de pedidos.

Essa separação permite reaproveitar o acompanhamento de batch em novas telas sem copiar a lógica de `fetch`, polling, estados de botão e atualização básica da UI.

## Helper PHP de enqueue

Arquivo:

`wordpress/wp-content/themes/siteorigin-corp/task-queue/assets/enqueue.php`

Uso:

```php
lemis_task_queue_enqueue(
	array(
		'script' => 'sync-page-test',
	)
);
```

Comportamento:

- Valida se `script` foi informado.
- Verifica se o script existe em `lemis_task_queue_get_frontend_scripts()`.
- Registra os assets compartilhados com `lemis_task_queue_register_frontend_assets()`.
- Enfileira o script solicitado com suas dependências.
- Injeta `window.lemisTaskQueue` antes do script da página.
- Retorna `true` quando enfileirou e `false` quando a entrada é inválida.

Configuração global injetada:

```js
window.lemisTaskQueue = {
  restUrl: rest_url('lemis/v1/'),
  nonce: wp_create_nonce('wp_rest')
};
```

## Scripts registrados

Script compartilhado:

- handle: `lemis-batch-progress`
- arquivo: `assets/js/batch-progress.js`
- expõe: `window.createBatchProgress`

Script de página registrado hoje:

- nome lógico: `sync-page-test`
- handle: `lemis-sync-page`
- arquivo: `assets/js/sync-page-test.js`
- dependência: `lemis-batch-progress`

Para adicionar um novo script de página, inclua uma entrada em `lemis_task_queue_get_frontend_scripts()`:

```php
'minha-page' => array(
	'handle' => 'lemis-minha-page',
	'file' => 'minha-page.js',
	'dependencies' => array(
		'lemis-batch-progress',
	),
),
```

## `createBatchProgress()`

`batch-progress.js` expõe:

```js
window.createBatchProgress(options)
```

Opções principais:

- `restUrl`: base da API REST, normalmente `window.lemisTaskQueue.restUrl`.
- `nonce`: nonce REST, normalmente `window.lemisTaskQueue.nonce`.
- `api.start`: endpoint relativo para iniciar o batch.
- `api.progress`: endpoint relativo ou função que recebe `batchId`.
- `api.summary`: endpoint relativo ou função que recebe `batchId`; opcional.
- `elements`: mapa de IDs dos elementos que serão atualizados.
- `polling.interval`: intervalo normal de polling em milissegundos.
- `polling.retryInterval`: intervalo após erro temporário de polling.
- `statusBaseClass`: classe base aplicada ao elemento de status.
- `loadingText`: texto do botão enquanto a operação inicia/roda.
- `buttonText`: texto do botão ao liberar novamente.

Callbacks:

- `getStatusMessage(data)`: customiza o texto do status.
- `getStatusClass(data)`: customiza a classe visual do status.
- `onStarted(result, state)`: executado depois que a API retorna `batch_id`.
- `onProgress(data, state)`: executado a cada atualização de progresso.
- `onCompleted(summary, data, state)`: executado quando o batch termina.
- `onSummaryError(error, state)`: executado se o resumo falhar após conclusão.
- `onError(error, state)`: executado se a inicialização falhar.

Métodos retornados:

- `start(payload)`: chama o endpoint de início e começa o polling.
- `poll(batchId)`: consulta progresso até o batch terminar.
- `getProgress(batchId)`: consulta progresso uma vez.
- `getSummary(batchId)`: consulta resumo uma vez quando `api.summary` existe.
- `renderProgress(data)`: atualiza estado interno e DOM com dados já carregados.
- `getState()`: retorna cópia do estado atual.
- `reset()`: limpa o estado interno.

## Contrato de endpoints

O endpoint de início precisa retornar em `data`:

```json
{
  "batch_id": 123,
  "jobs_created": 10,
  "status": "processing"
}
```

O endpoint de progresso deve retornar o formato de `task_queue_get_batch_progress()`:

```json
{
  "status": "processing",
  "total_jobs": 10,
  "completed_jobs": 4,
  "failed_jobs": 1,
  "pending_jobs": 4,
  "processing_jobs": 1,
  "progress": 50
}
```

O endpoint de resumo é opcional. Quando configurado, ele é chamado depois que o batch chega a `completed` ou `failed`.

## Contrato de elementos

`elements` aceita os seguintes nomes:

- `panel`: painel que recebe a classe `is-visible`.
- `status`: texto e classe do status.
- `batchId`: exibe `Batch #{id}`.
- `progressBar`: recebe `style.width` com o percentual.
- `progressText`: exibe `{progress}% concluído`.
- `completed`: contador de concluídos.
- `completedCard`: contador alternativo de concluídos.
- `failed`: contador de falhas.
- `pending`: contador em fila.
- `processing`: contador em processamento.
- `total`: total de jobs.
- `startButton`: botão que é desabilitado durante o processamento.

Todos os elementos são opcionais. Quando um ID não existe na página, o componente apenas ignora aquele ponto de renderização.

## `sync-page-test.js`

`sync-page-test.js` adapta `createBatchProgress()` para a página `page-sincronizar-teste.php`.

Responsabilidades:

- Validar se `window.lemisTaskQueue` e `window.createBatchProgress` existem.
- Ligar o formulário `#form_sync` ao endpoint `sync-pedidos`.
- Ler `#fetch_limit` e `#key_mode`.
- Enviar `{ fetch_limit, mode }` para iniciar a fila.
- Renderizar mensagens específicas da sincronização.
- Renderizar o resumo de vendas retornado pelo `summary_callback` de pedidos.

IDs esperados pela tela:

- `form_sync`
- `fetch_limit`
- `key_mode`
- `sync-progress-panel`
- `sync-status`
- `sync-batch-id`
- `sync-progress-bar`
- `sync-progress-text`
- `sync-finished-summary`
- `sync-total-summary`
- `sync-completed-card`
- `sync-failed`
- `sync-pending`
- `sync-processing`
- `sync-start`
- `sync-summary`
- `summary-created`
- `summary-created-label`
- `summary-existing`
- `summary-ignored`
- `summary-failed`
- `sync-sales-title`
- `sync-sales-list`

Alguns elementos de resumo podem estar ausentes ou comentados; o script verifica a existência antes de escrever.
