# Task Queue: Worker, Retries e Recuperação

[Voltar ao índice](task-queue-bootstrap.md)

## Worker

Arquivo:

`wordpress/wp-content/themes/siteorigin-corp/task-queue/worker/worker.php`

O worker:

- carrega `wp-load.php`
- carrega `task-queue/bootstrap.php`
- roda em loop infinito
- recupera jobs presos periodicamente
- reserva o próximo job `pending` disponível
- executa `lemis_task_queue_process_job()`
- dorme por `LEMIS_TASK_QUEUE_WORKER_SLEEP_SECONDS` quando não há jobs
- registra mensagens com `error_log`

Execução típica dentro do container:

```bash
docker compose exec wordpress php /var/www/html/wp-content/themes/siteorigin-corp/task-queue/worker/worker.php
```

Enquanto o worker não estiver rodando, os jobs podem ser criados normalmente, mas ficarão em `pending`.

## Ciclo de vida de um job

O worker processa jobs em:

```php
lemis_task_queue_process_job(array $job): string
```

Ciclo de execução:

1. O worker reserva um job `pending` disponível.
2. O status muda para `processing`.
3. O campo `attempts` é incrementado.
4. O payload JSON é decodificado.
5. O handler é localizado pelo `type`.
6. O handler é executado.
7. Em sucesso, o job vira `completed` e o retorno é salvo em `result`.
8. Em `LemisTaskQueueRetryableException`, o job volta para `pending` se ainda houver tentativas disponíveis.
9. Em qualquer outra exceção, o job vira `failed`.
10. Se o job pertence a um batch, os contadores do batch são atualizados em transação.
11. Quando `completed_jobs + failed_jobs >= total_jobs`, o batch vira `completed`.

Se o handler não existir, o job falha imediatamente.

## Retries

Retry temporário:

```php
lemis_task_queue_throw_retryable_exception(
	'API externa indisponível.',
	array('http_code' => 503)
);
```

Comportamento:

- `attempts` é incrementado quando o job é reservado.
- O retry padrão agenda o job para ficar disponível novamente em 10 segundos.
- O job só tenta novamente enquanto `attempts < LEMIS_TASK_QUEUE_WORKER_MAX_ATTEMPTS`.
- Ao atingir o limite, a exceção retryable também resulta em `failed`.

Use retry apenas para falhas transitórias, como timeout externo, indisponibilidade temporária de API ou limite temporário do fornecedor.

## Falhas definitivas

Qualquer exceção que não seja `LemisTaskQueueRetryableException` marca o job como `failed`.

Exemplos de falha definitiva:

- payload inválido
- regra de negócio que impede o processamento
- handler inexistente
- resposta externa válida, mas incompatível com o que o fluxo precisa

## Recuperação de jobs presos

O worker chama periodicamente:

```php
lemis_task_queue_recover_stale_jobs()
```

Um job é considerado preso quando:

- está em `processing`
- tem `started_at` preenchido
- `started_at` é mais antigo que `LEMIS_TASK_QUEUE_PROCESSING_TIMEOUT_SECONDS`

Comportamento:

- Se `attempts < LEMIS_TASK_QUEUE_WORKER_MAX_ATTEMPTS`, o job volta para `pending`.
- Se `attempts >= LEMIS_TASK_QUEUE_WORKER_MAX_ATTEMPTS`, o job vira `failed`.
- Quando um job preso com `batch_id` vira `failed`, o contador do batch é atualizado em transação.
- Se essa falha completar o batch, o hook `after_batch_finish` é executado.

Isso protege a fila contra queda do worker no meio de um processamento.
