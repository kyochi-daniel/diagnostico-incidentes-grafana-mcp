# Cenário de vazamento de conexões

Este diretório contém um cenário implementado: [`db-leaky-connections/main.ts`](db-leaky-connections/main.ts). Ele estende [`BaseScenario`](_base/baseScenario.ts) e é registrado em [`app.ts`](../app.ts).

## Comportamento

- `GET /students/db-leaky-connections` consulta estudantes usando um pool PostgreSQL limitado a duas conexões. As conexões adquiridas não são devolvidas ao pool, intencionalmente.
- Após esgotar o pool, novas aquisições aguardam o timeout de um segundo e o endpoint retorna HTTP 500. O erro é registrado nos logs e no span ativo.
- `POST /students/db-leaky-connections/reset` devolve as conexões retidas para permitir outra reprodução. O reset não corrige o vazamento.

## Ciclo de vida

1. `init()` chama `_init()` para criar o pool e marca o cenário como inicializado. Uma nova chamada não repete a inicialização.
2. `registerRoutes()` exige inicialização anterior e chama `_registerRoutes()` para registrar os dois endpoints.
3. `terminate()` chama `_terminate()`, que devolve as conexões retidas e encerra o pool. Em seguida, a classe base limpa seu estado de inicialização.

O hook `onClose` da aplicação aciona a finalização dos cenários quando a instância Fastify é fechada. Os testes podem selecionar este cenário pelo nome `db-leaky-connections`.

O vazamento permanece deliberado para estudo. Para instalar o ambiente, executar o experimento e consultar as evidências no Grafana/MCP, siga o [README principal](../../../README.md).
