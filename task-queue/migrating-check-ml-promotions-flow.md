# Migracao do fluxo de promocoes Mercado Livre

[Voltar ao indice](task-queue-bootstrap.md)

Este guia descreve a migracao do processamento de promocoes do Mercado Livre para o fluxo declarativo do `task-queue` com `lemis_task_queue_register_flow()`.

Ele tambem documenta o contrato real observado no banco de dados dos jobs. Um job de promocoes nao recebe apenas a loja: ele recebe a loja e um chunk de produtos ja normalizados em `produtos`.

O objetivo e tirar a pagina `page-promocoes-mercado-livre.php` do processamento por reload com `?processar_promos=1` e passar a:

- iniciar um batch pela rota generica de flows;
- acompanhar progresso por `job-batches/{id}`;
- manter o worker `promotions` processando jobs `check_ml_promotions_chunk`;
- centralizar `batch_type`, `handler`, callbacks e labels em `task-queue/config/flows.php`.

## Estado anterior

Antes da migracao, a pagina chamava `processarLotePromocoes()` dentro da propria request HTTP:

```php
if (isset($_GET['processar_promos'])) {
	$res = processarLotePromocoes($accessToken, 200, $loja);
	// reload ate finalizar
}
```

Tambem existia um iniciador direto em `functions.php`:

```php
function lemis_start_ml_promotions_check(string $loja): array
{
	// busca produtos, cria chunks e chama lemis_task_queue_create_batch_jobs()
}
```

Esse modelo espalhava a definicao do fluxo:

- `batch_type` em `batch-hook-registry.php`;
- `handler` em `handler-registry.php`;
- montagem dos chunks fora do modulo de flows;
- pagina PHP responsavel por iniciar e acompanhar o processamento.

## Estado atual trazido do servidor

O arquivo `page-promocoes-mercado-livre.php` trazido do servidor ja nao usa mais `processar_promos`, mas ainda usa endpoints locais da propria pagina:

- `?iniciar_fila=1`: valida a loja e chama `lemis_start_ml_promotions_check($loja)`;
- `?progresso_batch={id}`: consulta diretamente a tabela `wp_lemis_job_batches` para retornar `status`, `total_jobs`, `completed_jobs`, `failed_jobs` e `progress`;
- `?reset_promos=1`: remove os caches `cache_promocoes_{loja}.json` e `cache_progresso_{loja}.json`, inicia nova fila e redireciona com `resetado=1`.

Esse estado ja cria jobs persistidos, mas a pagina ainda controla o start e o polling por endpoints customizados. A migracao para `lemis_task_queue_register_flow()` remove essa responsabilidade da pagina e reaproveita a rota generica de flows e as rotas genericas de batch.

## Estado alvo

O fluxo passa a ser registrado em `task-queue/config/flows.php`, dentro de `lemis_task_queue_register_default_flows()`:

```php
lemis_task_queue_register_flow(
	array(
		'slug' => 'check-ml-promotions',
		'route' => 'check-ml-promotions',
		'batch_type' => 'check_ml_promotions',
		'handler' => 'check_ml_promotions_chunk',
		'item_key' => 'produtos',
		'prepare_callback' => 'lemis_check_ml_promotions_prepare_flow',
		'job_callback' => 'lemis_task_queue_handle_check_ml_promotions_chunk',
		'summary_callback' => 'lemis_task_queue_summarize_ml_promotions',
		'labels' => array(
			'loading' => 'Verificando promocoes...',
			'button' => 'Verificar promocoes ML',
			'processing' => 'Verificando promocoes ML',
			'completed' => 'Verificacao concluida',
			'completed_with_errors' => 'Verificacao concluida com falhas',
			'failed' => 'Verificacao falhou',
		),
	)
);
```

A pagina inicia o fluxo com:

```http
POST /wp-json/lemis/v1/task-queue/flows/check-ml-promotions/start
```

Payload:

```json
{
  "loja": "store"
}
```

Esse e o payload inicial enviado pela interface para iniciar o fluxo. Ele nao e o payload final salvo em cada job.

O JavaScript generico `task-queue-flow-page.js` passa a consultar automaticamente:

- `GET /wp-json/lemis/v1/job-batches/{batch_id}`;
- `GET /wp-json/lemis/v1/job-batches/{batch_id}/summary`.

## Payload real do job

O payload persistido no banco para cada job tem este formato:

```json
{
  "loja": "one",
  "produtos": [
    {
      "id": "MLB4758346979",
      "price": 532.9,
      "preco_sys": 532.9,
      "preco_por": 484.9,
      "title": "Vaso Decorativo Branco Patina De Metal 40x26cm Lucatti Branco",
      "sku": "123329",
      "thumbnail": "http://http2.mlstatic.com/D_820981-MLB113035738925_062026-I.jpg"
    },
    {
      "id": "MLB4758372005",
      "price": 125.9,
      "preco_sys": 118.9,
      "preco_por": 108.3,
      "title": "Mini Anjo Decorativo Cristal Murano Quartzo Labone 8x8cm Quartzo Perola",
      "sku": "126408",
      "thumbnail": "http://http2.mlstatic.com/D_810452-MLB111914295044_062026-I.jpg"
    }
  ]
}
```

No banco, o JSON pode aparecer com barras escapadas em URLs e caracteres Unicode escapados, por exemplo `http:\/\/...` e `\u00e9`. Isso e normal para JSON serializado e nao muda o contrato do handler.

Campos esperados em cada produto:

- `id`: ID do anuncio no Mercado Livre, por exemplo `MLB4758346979`;
- `price`: preco atual do anuncio no Mercado Livre;
- `preco_sys`: preco de referencia do sistema;
- `preco_por`: preco minimo/promocional usado para validar elegibilidade;
- `title`: titulo do anuncio;
- `sku`: SKU interno usado para rastreio;
- `thumbnail`: imagem do anuncio.

Esse payload e recebido por:

```php
lemis_task_queue_handle_check_ml_promotions_chunk(array $payload)
```

Portanto, a diferenca importante e:

- payload inicial da tela ou rota generica: `{"loja":"one"}`;
- payload persistido em cada job: `{"loja":"one","produtos":[...]}`.

Quem transforma o payload inicial no payload do job e o preparador do fluxo. No modelo alvo, essa responsabilidade fica em `lemis_check_ml_promotions_prepare_flow()`.

## Backend

O `prepare_callback` e responsavel por transformar a requisicao inicial em jobs.

Contrato de entrada:

- `loja`: deve ser `store` ou `one`.

Contrato de saida:

```php
return array(
	'success' => true,
	'items' => array_chunk($ativos, 50),
	'job_payload' => array(
		'loja' => $loja,
	),
	'item_key' => 'produtos',
);
```

Com `item_key => produtos`, cada item de `items` vira `$payload['produtos']` no job. O payload final recebido pelo handler continua sendo:

```php
array(
	'loja' => 'store',
	'produtos' => array(
		// ate 50 produtos normalizados
	),
)
```

Isso explica o payload observado no banco: `job_payload` fornece `loja`, e cada item retornado em `items` entra como `produtos`.

O handler nao deve depender da request original nem de estado da pagina. Ele recebe tudo que precisa no payload do job, exceto o token do Mercado Livre, que e obtido internamente a partir de `loja`.

Isso preserva o contrato do worker `promotions`, que continua consumindo o job type `check_ml_promotions_chunk`.

## Handler e resumo

O handler `lemis_task_queue_handle_check_ml_promotions_chunk()` deve retornar os campos que o summary usa:

```php
return array(
	'success' => true,
	'produtos_processados' => count($produtos),
	'promos_elegiveis' => $elegiveisNoChunk,
	'respostas_invalidas' => $errosNoChunk,
);
```

O summary `lemis_task_queue_summarize_ml_promotions()` soma:

- `produtos_processados`;
- `promos_elegiveis`;
- `respostas_invalidas`.

Se o handler retornar `produtos` em vez de `produtos_processados`, o resumo ficara com total de produtos zerado.

## Limpeza dos registries legados

Depois que o flow esta registrado, `lemis_task_queue_get_handler()` e `lemis_task_queue_get_batch_definition()` resolvem callbacks pela definicao do flow.

Por isso, remova dos fallbacks legados:

```php
// handler-registry.php
'check_ml_promotions_chunk' => 'lemis_task_queue_handle_check_ml_promotions_chunk',
```

```php
// batch-hook-registry.php
'check_ml_promotions' => array(
	'before_batch_start' => null,
	'after_batch_finish' => null,
	'summary_callback' => 'lemis_task_queue_summarize_ml_promotions',
),
```

Mantenha os fallbacks apenas para fluxos ainda nao migrados.

## Frontend PHP

### Estado atual da pagina

No arquivo trazido do servidor, a pagina inicia a fila com JavaScript chamando a propria URL com `iniciar_fila=1`. A resposta deve conter o `batch_id` retornado por `lemis_start_ml_promotions_check($loja)`.

Depois disso, a pagina consulta `progresso_batch={batch_id}` na propria URL para atualizar:

- status textual;
- barra de progresso;
- contador de jobs concluidos;
- total de jobs;
- falhas.

Esse modelo funciona, mas deixa a pagina acoplada ao formato interno da tabela `lemis_job_batches`.

### Estado alvo com frontend generico

Em `page-promocoes-mercado-livre.php`, registre o frontend do flow antes de `get_header()`:

```php
if (function_exists('lemis_task_queue_enqueue_flow')) {
	lemis_task_queue_enqueue_flow(
		'check-ml-promotions',
		array(
			'form_id' => 'check_ml_promotions_form',
			'id_prefix' => 'check-ml-promotions',
			'status_base_class' => 'check-ml-promotions-status',
		)
	);
}
```

Substitua o link antigo:

```php
<a href="?mercado=store&processar_promos=1">Iniciar processamento de Promocoes</a>
```

por um formulario:

```php
<form id="check_ml_promotions_form">
	<input type="hidden" name="loja" value="<?php echo esc_attr($loja); ?>">
	<button type="submit" id="check-ml-promotions-start">
		Iniciar processamento de Promocoes
	</button>
</form>
```

Renderize o painel de progresso com o mesmo `id_prefix`:

```php
if (function_exists('lemis_task_queue_render_progress_panel')) {
	lemis_task_queue_render_progress_panel(
		array(
			'id_prefix' => 'check-ml-promotions',
			'title' => 'Acompanhamento das promocoes',
			'initial_status' => 'Aguardando processamento',
			'class_prefix' => 'check-ml-promotions',
		)
	);
}
```

O script generico procura estes IDs:

- `check-ml-promotions-progress-panel`;
- `check-ml-promotions-status`;
- `check-ml-promotions-batch-id`;
- `check-ml-promotions-progress-bar`;
- `check-ml-promotions-progress-text`;
- `check-ml-promotions-pending`;
- `check-ml-promotions-processing`;
- `check-ml-promotions-completed-card`;
- `check-ml-promotions-failed`;
- `check-ml-promotions-start`.

Se algum ID for alterado, informe o mapeamento em `elements` na chamada de `lemis_task_queue_enqueue_flow()`.

Ao migrar para o frontend generico, remova os endpoints locais `iniciar_fila` e `progresso_batch` da pagina apenas depois de confirmar que:

- o submit do formulario chama `/wp-json/lemis/v1/task-queue/flows/check-ml-promotions/start`;
- a resposta retorna `batch_id`;
- o polling generico consulta `/wp-json/lemis/v1/job-batches/{batch_id}`;
- o worker `promotions` processa os jobs `check_ml_promotions_chunk`.

## Atualizacao da listagem

O processamento grava o cache em:

```text
logsMercado/cache_promocoes_{loja}.json
```

A listagem da pagina consome esse cache. Depois que o batch terminar, a pagina pode ser recarregada para exibir os novos itens elegiveis.

Nesta migracao, o acompanhamento de progresso nao precisa de JavaScript especifico para promocoes. O formulario inicia o batch e o painel mostra a evolucao dos jobs. Se for necessario atualizar a tabela automaticamente ao concluir, adicione um callback especifico para `check-ml-promotions` em `task-queue-flow-page.js`.

## Worker

O profile `promotions` continua valido:

```php
'promotions' => array(
	'types' => array(
		'check_ml_promotions_chunk',
	),
),
```

A migracao muda como o batch e criado, mas nao muda o tipo de job consumido pelo worker.

## Verificacao

Rode lint nos arquivos alterados:

```bash
docker compose exec wordpress php -l /var/www/html/wp-content/themes/siteorigin-corp/task-queue/config/flows.php
docker compose exec wordpress php -l /var/www/html/wp-content/themes/siteorigin-corp/task-queue/infrastructure/handlers/handlers.php
docker compose exec wordpress php -l /var/www/html/wp-content/themes/siteorigin-corp/page-promocoes-mercado-livre.php
```

Teste funcional:

1. Abra a pagina com `?mercado=store`.
2. Clique em `Iniciar processamento de Promocoes`.
3. Confirme que a resposta cria um `batch_id`.
4. Confirme no banco que os jobs foram criados com `type = check_ml_promotions_chunk`.
5. Rode o worker `promotions`.
6. Acompanhe o painel ate `completed`.
7. Consulte `/wp-json/lemis/v1/job-batches/{batch_id}/summary`.
8. Recarregue a pagina e confirme que a listagem reflete o cache atualizado.
