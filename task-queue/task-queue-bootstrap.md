# Task Queue

Esta é a documentação do módulo `task-queue`, carregado por:

`wordpress/wp-content/themes/siteorigin-corp/task-queue/bootstrap.php`

O módulo permite enfileirar trabalhos demorados em batches, processar cada item por um worker PHP e acompanhar progresso por rotas REST.

## Índice

1. [Visão geral](task-queue/overview.md)
   - objetivo do módulo
   - quando usar
   - fluxo mental de batch e job

2. [Bootstrap, configuração e banco de dados](task-queue/bootstrap-and-schema.md)
   - arquivos carregados pelo `bootstrap.php`
   - hooks do WordPress
   - constantes
   - tabelas e campos principais

3. [API pública e contratos](task-queue/api-and-contracts.md)
   - funções para criar batch e job
   - consulta de progresso
   - contrato do payload
   - contrato do retorno do handler
   - registries de handlers e hooks de batch

4. [Worker, retries e recuperação](task-queue/worker-and-retries.md)
   - execução do worker
   - ciclo de vida de um job
   - retries
   - recuperação de jobs presos em `processing`

5. [Rotas REST e fluxo de sincronização de pedidos](task-queue/rest-and-sync-flow.md)
   - rotas de progresso e resumo
   - endpoint `POST /wp-json/lemis/v1/sync-pedidos`
   - uso atual em `page-sincronizar.php` e `page-sincronizar-teste.php`
   - contrato do resumo de pedidos

6. [Guia para criar um novo fluxo](task-queue/new-flow-guide.md)
   - passo a passo
   - exemplo de handler
   - exemplo de função iniciadora
   - boas práticas
   - checklist

7. [Referências de código](task-queue/code-reference.md)
   - lista dos arquivos principais do módulo
   - comandos de verificação recomendados

## Leitura recomendada

Para apenas usar o módulo em um novo fluxo, leia nesta ordem:

1. [Visão geral](task-queue/overview.md)
2. [API pública e contratos](task-queue/api-and-contracts.md)
3. [Guia para criar um novo fluxo](task-queue/new-flow-guide.md)
4. [Worker, retries e recuperação](task-queue/worker-and-retries.md)

Para dar manutenção no módulo em si, leia também:

1. [Bootstrap, configuração e banco de dados](task-queue/bootstrap-and-schema.md)
2. [Rotas REST e fluxo de sincronização de pedidos](task-queue/rest-and-sync-flow.md)
3. [Referências de código](task-queue/code-reference.md)
