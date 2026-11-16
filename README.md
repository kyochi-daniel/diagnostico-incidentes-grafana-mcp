# Diagnóstico de incidentes com Grafana e MCP

Laboratório de observabilidade para investigar um vazamento intencional de conexões PostgreSQL. A aplicação produz logs, métricas e traces; um cliente de IA compatível com MCP pode consultar essa telemetria pelo servidor Grafana MCP.

Projeto acadêmico realizado durante minha pós-graduação em Engenharia de Software com IA Aplicada, a partir do material didático da Unipds.

O objetivo é correlacionar sintomas HTTP, erros do banco e operações instrumentadas para explicar a causa de um incidente reproduzível.

## Arquitetura e papel da IA

- A API Fastify consulta PostgreSQL com Knex e `pg`.
- O SDK OpenTelemetry instrumenta a aplicação e envia telemetria ao Collector via OTLP/gRPC.
- O Collector encaminha traces ao Tempo e logs ao Loki, além de expor métricas para coleta pelo Prometheus.
- O Grafana reúne as três fontes e carrega dashboards provisionados.
- O servidor `mcp/grafana` permite que um cliente de IA consulte o Grafana por ferramentas MCP.
- O Blackbox Exporter complementa o ambiente com verificações de disponibilidade.

O servidor MCP é uma integração pronta incluída no Compose. Este projeto não implementa um servidor MCP próprio nem um agente autônomo. O modelo e a interface de IA pertencem ao cliente MCP escolhido pelo usuário.

## Cenário: conexões não devolvidas ao pool

O endpoint `GET /students/db-leaky-connections` adquire uma conexão, executa a consulta e não chama `client.release()`. O pool tem limite de duas conexões e timeout de aquisição de um segundo.

Com o pool vazio e requisições sequenciais, as duas primeiras consultas devem funcionar. As seguintes aguardam o timeout e retornam HTTP 500. Esse comportamento é intencional e permanece no código para permitir o diagnóstico.

O endpoint `POST /students/db-leaky-connections/reset` devolve as conexões retidas e permite repetir o experimento. Ele reinicia o cenário; não corrige sua causa.

## Pré-requisitos e isolamento

- Node.js `22.13.1`, versão indicada pela aplicação, e npm.
- Docker e Docker Compose com suporte a `--wait` e, para testes em container, `include`.
- Conexão à internet para baixar dependências e imagens.
- Cliente de IA compatível com MCP por HTTP, caso queira realizar o diagnóstico assistido.

**Use somente PostgreSQL dedicado e descartável.** A inicialização recria as tabelas `courses` e `students`. A finalização dos testes encerra conexões e exclui o banco configurado. Nunca aponte `DATABASE_URL` para um banco existente de produção ou com dados que precise preservar.

Os serviços do Compose e a API iniciada com Node ficam acessíveis em `127.0.0.1`. O ambiente usa configurações de desenvolvimento, incluindo acesso anônimo administrativo no Grafana, e deve permanecer restrito ao uso local.

## Instalação e execução

Execute na raiz do repositório:

```bash
npm ci --prefix _alumnus
npm run docker:infra:up
npm start
```

`npm start` inicia a aplicação na porta `9000`. Com a configuração padrão de `NODE_ENV=production`, a própria aplicação chama o endpoint defeituoso a cada dois segundos para gerar telemetria. Neste laboratório, esse valor controla a geração de tráfego, não indica preparo para produção.

Para reproduzir manualmente desde o início, pare a aplicação e inicie sem o tráfego automático:

```bash
NODE_ENV=development npm start
```

Em outro terminal, faça três chamadas sequenciais:

```bash
curl -i http://localhost:9000/students/db-leaky-connections
curl -i http://localhost:9000/students/db-leaky-connections
curl -i http://localhost:9000/students/db-leaky-connections
```

Para liberar o pool e repetir:

```bash
curl -X POST http://localhost:9000/students/db-leaky-connections/reset
```

| Serviço | Acesso local |
| --- | --- |
| API / health check | `http://localhost:9000/health` |
| Grafana | `http://localhost:3000` |
| Prometheus | `http://localhost:9090` |
| Loki | `http://localhost:3100/ready` |
| Tempo | `http://localhost:3200/ready` |
| Grafana MCP | `http://localhost:8000/mcp` |
| PostgreSQL | Porta `5433` |

O PostgreSQL e o Collector já possuem valores locais padrão em [`_alumnus/src/util/config.ts`](_alumnus/src/util/config.ts). As variáveis `PORT`, `DATABASE_URL`, `APPNAME`, `NODE_ENV`, `LOG_LEVEL` e `OTEL_EXPORTER_OTLP_ENDPOINT` permitem ajustes pelo ambiente do processo. O comando de execução não carrega `.env` automaticamente.

## Roteiro de diagnóstico

1. Conecte seu cliente MCP a `http://localhost:8000/mcp` usando o transporte **Streamable HTTP**. O formato da configuração depende do cliente.
2. Gere as requisições e aguarde a exportação e indexação da telemetria. Identifique o nome do serviço nos logs de inicialização/telemetria; por padrão ele contém um sufixo aleatório.
3. No Prometheus, examine a taxa de erros e os tempos de resposta do endpoint.
4. No Loki, procure os erros de aquisição de conexão e correlacione seus horários e identificadores de trace.
5. No Tempo, examine os traces relacionados e os eventos de exceção.
6. Compare as evidências com [`_alumnus/src/scenarios/db-leaky-connections/main.ts`](_alumnus/src/scenarios/db-leaky-connections/main.ts). Registre causa, evidências e a correção proposta em um novo relatório.

Exemplos de prompts do material didático, em [`docs/grafana-mcp-prompts.md`](docs/grafana-mcp-prompts.md):

```text
Query Prometheus for HTTP request rate and latency metrics from the alumnus application over the last hour. Show me the request rate per endpoint and the p95 latency.
```

```text
List all currently firing alerts from Prometheus and show me their severity, description, and how long they've been firing.
```

O roteiro em [`docs/prompt.md`](docs/prompt.md) detalha a correlação entre Prometheus, Loki e Tempo. Seus tempos, referências de linha e resultados esperados são ilustrações didáticas; produza um relatório com os dados da sua execução. A ausência de um span de liberação, isoladamente, não prova vazamento: confirme a hipótese no ciclo de vida das conexões.

O projeto não exige uma chave de API de LLM na aplicação. Eventuais custos e o envio de telemetria ao modelo dependem do cliente de IA utilizado.

## Testes de integração

Com a infraestrutura iniciada e as dependências instaladas, execute na raiz:

```bash
LOKI_URL=http://127.0.0.1:3100 TEMPO_URL=http://127.0.0.1:3200 PROMETHEUS_URL=http://127.0.0.1:9090 npm test
```

A suíte usa `node:test`, inicia uma instância da aplicação em porta dinâmica e verifica o esgotamento do pool e a presença de logs, traces e métricas nos backends reais. Ela não avalia a qualidade do diagnóstico de um modelo de IA. Uma verificação específica dos traces do endpoint permanece marcada com `it.skip`.

Como alternativa, com as portas livres e sem outra instância deste ambiente em execução:

```bash
npm run test:docker
npm run test:docker:down
```

O caminho Docker constrói o runner e inicia os serviços definidos em `infra/docker-compose-test.yaml`. O banco padrão do runner é `alumnus_test`, excluído ao encerrar a suíte. O Dockerfile utiliza pnpm internamente; não é necessário instalá-lo na máquina para esse caminho.

## Encerramento e limitações

Pare a aplicação com `Ctrl+C` e encerre a infraestrutura:

```bash
npm run docker:infra:down
```

- Os dados dos serviços ficam em `infra/storage/` e persistem após `down`. O script `docker:infra:cleanup` remove esse diretório e seus dados.
- O laboratório depende de vários containers, portas disponíveis e tempo para coleta/indexação. Uma falha de consulta não comprova ausência de telemetria.
- O vazamento é deliberado; não foi corrigido como parte da organização deste portfólio.
- A suíte de integração não cobre uma solução completa de incidentes, o cliente MCP ou a fidelidade das respostas de IA.
- Este README descreve a implementação e o procedimento de reprodução. Não apresenta medições de desempenho nem resultados de uma investigação executada neste ambiente.

Detalhes dos serviços e comandos de inspeção estão em [`infra/README.md`](infra/README.md).

## Origem, créditos e licença

Baseado no exemplo `exemplo-09-grafana-mcp/alumnus` do [material didático da Unipds](https://github.com/unipds-engenharia-de-ia-aplicada/engenharia-de-software-com-ia-aplicada), com créditos originais a Erick Wendel e à Unipds.

A organização desta cópia inclui documentação em português, ajustes de execução e configuração local e uma correção na asserção do teste de esgotamento do pool. A aplicação e o cenário didático mantêm sua origem; não são apresentados como criação original.

A licença original está preservada em [`LICENSE.md`](LICENSE.md), com as condições **CC BY-NC-ND 4.0**. Consulte o arquivo para atribuição, uso não comercial e distribuição.
