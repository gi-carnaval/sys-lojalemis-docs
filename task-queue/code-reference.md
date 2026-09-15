# Task Queue: Referências de Código

[Voltar ao índice](task-queue-bootstrap.md)

## Arquivos principais

- `wordpress/wp-content/themes/siteorigin-corp/task-queue/bootstrap.php`
- `wordpress/wp-content/themes/siteorigin-corp/task-queue/config/module.php`
- `wordpress/wp-content/themes/siteorigin-corp/task-queue/services/module.php`
- `wordpress/wp-content/themes/siteorigin-corp/task-queue/services/jobs.php`
- `wordpress/wp-content/themes/siteorigin-corp/task-queue/services/batches.php`
- `wordpress/wp-content/themes/siteorigin-corp/task-queue/api/module.php`
- `wordpress/wp-content/themes/siteorigin-corp/task-queue/api/batches.php`
- `wordpress/wp-content/themes/siteorigin-corp/task-queue/api/sync.php`
- `wordpress/wp-content/themes/siteorigin-corp/task-queue/domain/exceptions.php`
- `wordpress/wp-content/themes/siteorigin-corp/task-queue/assets/enqueue.php`
- `wordpress/wp-content/themes/siteorigin-corp/task-queue/infrastructure/database/table-names.php`
- `wordpress/wp-content/themes/siteorigin-corp/task-queue/infrastructure/database/schema-map.php`
- `wordpress/wp-content/themes/siteorigin-corp/task-queue/infrastructure/database/schema-installer.php`
- `wordpress/wp-content/themes/siteorigin-corp/task-queue/infrastructure/handlers/handler-registry.php`
- `wordpress/wp-content/themes/siteorigin-corp/task-queue/infrastructure/handlers/batch-hook-registry.php`
- `wordpress/wp-content/themes/siteorigin-corp/task-queue/infrastructure/handlers/handlers.php`
- `wordpress/wp-content/themes/siteorigin-corp/task-queue/worker/process-job.php`
- `wordpress/wp-content/themes/siteorigin-corp/task-queue/worker/worker.php`
- `wordpress/wp-content/themes/siteorigin-corp/task-queue/assets/js/batch-progress.js`
- `wordpress/wp-content/themes/siteorigin-corp/task-queue/assets/js/sync-page-test.js`
- `wordpress/wp-content/themes/siteorigin-corp/task-queue/assets/js/sync-page.js`
- `wordpress/wp-content/themes/siteorigin-corp/functions.php`
- `wordpress/wp-content/themes/siteorigin-corp/page-sincronizar.php`
- `wordpress/wp-content/themes/siteorigin-corp/page-sincronizar-teste.php`

## Verificação recomendada

Depois de alterar código do módulo:

```bash
docker compose exec wordpress php -l /var/www/html/wp-content/themes/siteorigin-corp/task-queue/bootstrap.php
docker compose exec wordpress php -l /var/www/html/wp-content/themes/siteorigin-corp/task-queue/assets/enqueue.php
docker compose exec wordpress php -l /var/www/html/wp-content/themes/siteorigin-corp/task-queue/services/jobs.php
docker compose exec wordpress php -l /var/www/html/wp-content/themes/siteorigin-corp/task-queue/services/batches.php
docker compose exec wordpress php -l /var/www/html/wp-content/themes/siteorigin-corp/task-queue/infrastructure/handlers/handlers.php
```

Para validar o fluxo completo:

1. Suba o ambiente com `docker compose up -d --build`.
2. Inicie o worker em um terminal.
3. Abra a tela que dispara o fluxo.
4. Crie um batch.
5. Acompanhe o progresso pela UI ou por `GET /wp-json/lemis/v1/job-batches/{id}`.
6. Confira o resumo final.
7. Verifique logs do container WordPress se houver falhas.
