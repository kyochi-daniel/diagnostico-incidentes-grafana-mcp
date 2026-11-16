# Infraestrutura do laboratório

Consulte o [README principal](../README.md) para instalação, reprodução do vazamento, configuração MCP e execução dos testes. Todos os comandos abaixo devem ser executados **na raiz do repositório**.

## Serviços e fluxo

A aplicação envia traces, métricas e logs ao OpenTelemetry Collector por OTLP/gRPC na porta `4317`. O Collector encaminha traces ao Tempo, logs ao Loki e expõe as métricas na porta `8889`, coletadas pelo Prometheus.

O Grafana consulta essas fontes usando a rede interna do Compose. O serviço `mcp-grafana`, baseado em uma imagem pronta, acessa o Grafana e disponibiliza ferramentas para clientes MCP por Streamable HTTP em `http://localhost:8000/mcp`.

| Serviço | Função | Porta local |
| --- | --- | --- |
| `alumnus-postgres` | Banco descartável da demonstração | `5433` |
| `opentelemetry-collector` | Recepção OTLP e exportação de métricas | `4317`, `8889` |
| `prometheus` | Métricas e avaliação de regras de alerta | `9090` |
| `loki` | Logs | `3100` |
| `tempo` | Traces | `3200` |
| `tempo-healthcheck` | Verificação de prontidão do Tempo | Sem porta fixa publicada |
| `grafana` | Dashboards e exploração dos dados | `3000` |
| `blackbox` | Verificações HTTP, TCP e ICMP | `9115` |
| `mcp-grafana` | Interface MCP do Grafana | `8000` |

As portas publicadas ficam restritas a `127.0.0.1`. As configurações internas usam nomes de serviço como `tempo` e `loki`, não os endereços `localhost` do host.

## Operação e inspeção

```bash
npm run docker:infra:up
docker compose -f infra/docker-compose-infra.yaml ps
docker compose -f infra/docker-compose-infra.yaml logs --tail=100 opentelemetry-collector
```

Verifique a prontidão dos backends pelo host:

```bash
curl -f http://localhost:3000/api/health
curl -f http://localhost:9090/-/healthy
curl -f http://localhost:3100/ready
curl -f http://localhost:3200/ready
```

As métricas exportadas pelo Collector podem ser inspecionadas em `http://localhost:8889/metrics`. O comando `npm run docker:infra:logs` acompanha os logs de todos os serviços.

Para parar, use `npm run docker:infra:down`. Os dados persistem em `infra/storage/`; `npm run docker:infra:cleanup` remove esse diretório, além de encerrar os serviços e remover volumes. Execute a limpeza somente para descartar os dados deste laboratório.

O PostgreSQL é exclusivo para a demonstração: a aplicação recria tabelas e os testes excluem seu banco. O Grafana permite acesso anônimo administrativo e o MCP não configura autenticação neste Compose. Esta configuração se destina ao uso local.

## Arquivos de configuração

- [`docker-compose-infra.yaml`](docker-compose-infra.yaml): infraestrutura, volumes e publicação de portas.
- [`docker-compose-test.yaml`](docker-compose-test.yaml): inclui a infraestrutura e acrescenta o runner dos testes.
- [`otel-collector/otel-collector-config.yaml`](otel-collector/otel-collector-config.yaml): pipelines de traces, logs e métricas.
- [`prometheus/prometheus.yaml`](prometheus/prometheus.yaml): alvos de coleta e verificações Blackbox.
- [`prometheus/alerts.yaml`](prometheus/alerts.yaml): regras carregadas pelo Prometheus.
- [`grafana/provisioning/datasources/datasources.yaml`](grafana/provisioning/datasources/datasources.yaml): fontes e configuração de correlação entre sinais.
- [`grafana/provisioning/dashboards/dashboards.yaml`](grafana/provisioning/dashboards/dashboards.yaml) e `grafana/dashboards/`: provisionamento dos dashboards.

## Operação e inspeção

```bash
npm run docker:infra:up
docker compose -f infra/docker-compose-infra.yaml ps
```

Para consultar logs:

```bash
docker compose -f infra/docker-compose-infra.yaml logs --tail=100 opentelemetry-collector
docker compose -f infra/docker-compose-infra.yaml logs --tail=100 mcp-grafana
```

Para verificar os endpoints de prontidão:

```bash
curl -f http://localhost:3000/api/health
curl -f http://localhost:9090/-/healthy
curl -f http://localhost:3100/ready
curl -f http://localhost:3200/ready
```

Se não houver dados, confira primeiro se a aplicação está em execução, se fez requisições e se aponta para o Collector local. Depois verifique os logs do Collector e o estado dos alvos em `http://localhost:9090/targets`. Aguarde os intervalos de exportação e coleta antes de concluir que uma consulta está vazia.

## Persistência e isolamento

Os diretórios `infra/storage/` são montados nos containers e guardam os dados locais. `npm run docker:infra:down` encerra os serviços sem apagar esses diretórios. `npm run docker:infra:cleanup` os remove, incluindo bancos e telemetria.

O ambiente mantém credenciais de desenvolvimento e acesso anônimo administrativo no Grafana. É um laboratório local, sem configuração de produção. Use PostgreSQL descartável: a aplicação recria tabelas e os testes podem excluir o banco configurado.

Os alertas e dashboards são material de estudo. A presença de regras não implica que uma execução específica tenha produzido um alerta ou enviado notificações.
