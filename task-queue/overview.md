# Task Queue: Visão Geral

[Voltar ao índice](../task-queue-bootstrap.md)

## Objetivo

O `task-queue` foi criado para tirar processamento pesado da requisição HTTP. Em vez de uma página ou rota REST processar muitos itens diretamente, ela cria um lote de jobs, inicia o batch e deixa um worker PHP consumir a fila em segundo plano.

O ponto de entrada do módulo é:

`wordpress/wp-content/themes/siteorigin-corp/task-queue/bootstrap.php`

O uso principal hoje é a sincronização de pedidos do Bling, disparada por:

- `wordpress/wp-content/themes/siteorigin-corp/page-sincronizar.php`
- `wordpress/wp-content/themes/siteorigin-corp/page-sincronizar-teste.php`
- `POST /wp-json/lemis/v1/sync-pedidos`

## Quando usar

Use o `task-queue` quando um fluxo:

- processa vários itens independentes, como pedidos, produtos, clientes ou arquivos
- pode demorar mais do que uma requisição HTTP deveria esperar
- precisa mostrar progresso para uma tela administrativa
- precisa registrar sucesso, falha, retry e resumo final por item
- pode ser reexecutado por unidade de trabalho sem depender do estado da requisição original

Não use o módulo para ações pequenas e síncronas que precisam responder imediatamente ao usuário.

## Modelo mental

Um fluxo com fila normalmente tem três partes:

- Batch: representa a execução completa, por exemplo "sincronizar 50 pedidos".
- Job: representa uma unidade independente dentro do batch, por exemplo "sincronizar o pedido 123".
- Worker: processo PHP em segundo plano que reserva jobs pendentes e executa o handler de cada tipo.

Fluxo normal:

1. Uma página, rota REST ou ação interna identifica os itens a processar.
2. O código cria um batch.
3. O código cria um job para cada item.
4. O batch é iniciado.
5. O worker reserva cada job `pending`.
6. O handler do tipo do job executa o trabalho.
7. O job vira `completed`, `failed` ou volta para `pending` em caso de retry.
8. O batch acumula contadores de sucesso e falha.
9. Quando todos os jobs terminam, o batch vira `completed`.

## Estados

Estados de job:

- `pending`
- `processing`
- `completed`
- `failed`

Estados principais de batch:

- `pending`
- `processing`
- `completed`

## Onde continuar

- Para entender o carregamento e as tabelas, veja [Bootstrap, configuração e banco de dados](bootstrap-and-schema.md).
- Para usar as funções públicas, veja [API pública e contratos](api-and-contracts.md).
- Para criar um fluxo novo, veja [Guia para criar um novo fluxo](new-flow-guide.md).

V1.0.1