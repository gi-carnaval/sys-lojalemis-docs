# Task Queue: Bootstrap, Configuração e Banco de Dados

[Voltar ao índice](../task-queue-bootstrap.md)

## Bootstrap

O arquivo `bootstrap.php` não contém regra de negócio. Ele carrega as dependências e registra hooks do WordPress.

Arquivos carregados:

- `config/module.php`: constantes do módulo
- `infrastructure/database/table-names.php`: nomes das tabelas com prefixo do WordPress
- `infrastructure/database/schema-map.php`: definição das colunas e índices
- `infrastructure/database/schema-installer.php`: instalação/atualização do schema
- `domain/exceptions.php`: exceção retryable
- `services/jobs.php`: criação, reserva, conclusão, falha, retry e recuperação de jobs
- `services/batches.php`: criação, início, progresso, resumo e finalização de batches
- `services/module.php`: bootstrap interno e instalação condicional do schema
- `api/batches.php`: rotas REST genéricas de acompanhamento
- `api/sync.php`: rota REST atual de sincronização de pedidos
- `api/module.php`: registro das rotas no `rest_api_init`
- `infrastructure/handlers/batch-hook-registry.php`: registry de hooks de batch
- `infrastructure/handlers/handlers.php`: handlers atuais
- `infrastructure/handlers/handler-registry.php`: registry de handlers de job
- `worker/process-job.php`: execução de um job reservado

Hooks registrados:

```php
add_action('after_setup_theme', 'lojalemis_task_queue_bootstrap', 7);
add_action('after_switch_theme', 'lojalemis_task_queue_install_schema');
add_action('admin_init', 'lojalemis_task_queue_maybe_install_schema');
```

Na prática:

- `after_setup_theme` monta o container simples de serviços do módulo.
- `after_switch_theme` instala as tabelas quando o tema é ativado.
- `admin_init` compara a opção `LEMIS_TASK_QUEUE_SCHEMA_OPTION` com `LEMIS_TASK_QUEUE_MODULE_VERSION` e reinstala o schema quando necessário.

## Configuração

As constantes ficam em `task-queue/config/module.php`.

Constantes principais:

- `LEMIS_TASK_QUEUE_MODULE_VERSION`: versão do schema/módulo. Incrementar quando uma mudança de schema precisar ser aplicada.
- `LEMIS_TASK_QUEUE_MODULE_PATH`: caminho local do módulo.
- `LEMIS_TASK_QUEUE_MODULE_URL`: URL pública dos assets do módulo.
- `LEMIS_TASK_QUEUE_SCHEMA_OPTION`: nome da option que guarda a versão instalada.
- `LEMIS_TASK_QUEUE_WORKER_SLEEP_SECONDS`: tempo que o worker espera quando não há jobs disponíveis. Hoje: `2`.
- `LEMIS_TASK_QUEUE_WORKER_MAX_ATTEMPTS`: número máximo de tentativas por job. Hoje: `3`.
- `LEMIS_TASK_QUEUE_PROCESSING_TIMEOUT_SECONDS`: tempo máximo para um job ficar em `processing` antes de ser considerado preso. Hoje: `600`.
- `LEMIS_TASK_QUEUE_RECOVERY_INTERVAL_SECONDS`: intervalo entre varreduras de recuperação no worker. Hoje: `10`.

## Banco de dados

As tabelas são retornadas por `lemis_task_queue_table_names()`:

- `{wp_prefix}lemis_jobs`
- `{wp_prefix}lemis_job_batches`

### Tabela de jobs

Cada linha representa uma unidade de trabalho.

Campos mais importantes:

- `id`: identificador do job.
- `type`: tipo do job usado para encontrar o handler no registry.
- `status`: estado atual do job.
- `payload`: JSON com os dados de entrada.
- `result`: JSON com o retorno do handler.
- `attempts`: quantidade de vezes que o job já foi reservado pelo worker.
- `available_at`: data/hora a partir da qual o job pode ser reservado.
- `started_at`: data/hora em que a tentativa atual começou.
- `finished_at`: data/hora de conclusão ou falha.
- `error_message`: mensagem de erro da última falha ou retry.
- `batch_id`: batch ao qual o job pertence.
- `created_at` e `updated_at`: timestamps operacionais.

Índices relevantes:

- `idx_queue (status, available_at)`: usado pelo worker para buscar o próximo job disponível.
- `idx_type_status (type, status)`: consultas por tipo/status.
- `idx_batch_status (batch_id, status)`: contagem de progresso por batch.

### Tabela de batches

Cada linha representa uma execução agrupada.

Campos mais importantes:

- `id`: identificador do batch.
- `type`: tipo do batch usado para encontrar hooks no registry.
- `status`: `pending`, `processing` ou `completed`.
- `total_jobs`: quantidade total esperada.
- `completed_jobs`: jobs concluídos com sucesso.
- `failed_jobs`: jobs concluídos com falha definitiva.
- `created_at`: data/hora de criação.
- `finished_at`: data/hora de finalização.
