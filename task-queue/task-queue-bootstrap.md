# Task Queue

Esta é a documentação do módulo `task-queue`, carregado por:

`wordpress/wp-content/themes/siteorigin-corp/task-queue/bootstrap.php`

O módulo permite enfileirar trabalhos demorados em batches, processar cada item por um worker PHP e acompanhar progresso por rotas REST.

## Índice

1. [Visão geral](overview.md)
   - objetivo do módulo
   - quando usar
   - fluxo mental de batch e job

2. [Bootstrap, configuração e banco de dados](bootstrap-and-schema.md)
   - arquivos carregados pelo `bootstrap.php`
   - hooks do WordPress
   - constantes
   - tabelas e campos principais

3. [API pública e contratos](api-and-contracts.md)
   - funções para criar batch e job
   - consulta de progresso
   - contrato do payload
   - contrato do retorno do handler
   - registries de handlers e hooks de batch

4. [Worker, retries e recuperação](worker-and-retries.md)
   - execução do worker
   - ciclo de vida de um job
   - retries
   - recuperação de jobs presos em `processing`

5. [Rotas REST e fluxo de sincronização de pedidos](rest-and-sync-flow.md)
   - rotas de progresso e resumo
   - endpoint `POST /wp-json/lemis/v1/sync-pedidos`
   - uso atual em `page-sincronizar.php` e `page-sincronizar-teste.php`
   - contrato do resumo de pedidos

6. [Assets e progresso no frontend](frontend-assets.md)
   - helper PHP `lemis_task_queue_enqueue()`
   - script base `batch-progress.js`
   - adaptador `sync-page-test.js`
   - contrato de elementos da UI

7. [Guia para criar um novo fluxo](new-flow-guide.md)
   - passo a passo
   - exemplo de handler
   - exemplo de função iniciadora
   - exemplo de integração frontend
   - boas práticas
   - checklist

8. [Migração do fluxo de promoções Mercado Livre](migrating-check-ml-promotions-flow.md)
   - registro com `lemis_task_queue_register_flow()`
   - `prepare_callback` para chunks de produtos
   - alteração da página PHP frontend
   - acompanhamento de jobs pelo painel genérico

9. [Referências de código](code-reference.md)
   - lista dos arquivos principais do módulo
   - comandos de verificação recomendados

## Leitura recomendada

Para apenas usar o módulo em um novo fluxo, leia nesta ordem:

1. [Visão geral](overview.md)
2. [API pública e contratos](api-and-contracts.md)
3. [Guia para criar um novo fluxo](new-flow-guide.md)
4. [Assets e progresso no frontend](frontend-assets.md)
5. [Worker, retries e recuperação](worker-and-retries.md)

Para dar manutenção no módulo em si, leia também:

1. [Bootstrap, configuração e banco de dados](bootstrap-and-schema.md)
2. [Rotas REST e fluxo de sincronização de pedidos](rest-and-sync-flow.md)
3. [Assets e progresso no frontend](frontend-assets.md)
4. [Referências de código](code-reference.md)
