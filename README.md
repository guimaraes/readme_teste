feat(core-multicalculo): lib de multicalculo com ports, registry e telemetry

Extrai o núcleo de multicalculo para java-lib-reem-resi-core-multicalculo,
alinhado à arquitetura alvo (BFF → API → Core compartilhado → adapters).
Decisões:
- Ports e delegates (CallbackGravacao, ValidacaoCanal, Notifier, fluxo de
  cotação com reserva/batch) concentram contratos estáveis na lib e deixam
  HTTP/persistência na API, reduzindo acoplamento infra.
- CotacaoProcessor(ForUpdate) em com.porto.resi.service mantém imports
  familiares para consumidores que já usavam esse pacote na API.
- ProductRegistry + Identified* + ProductRegistries permitem registro por
  produto sem @Qualifier, caminhando para ownership por módulo.
- CorrelationIdStrategies com uuidAleatorioPorOferta e estavelPorExtrator
  deixa explícito o trade-off rastreio vs id determinístico na reserva.
- Pacote telemetry (antes chassis) e nomenclatura observabilidade no DEVPLAN
  alinham ao vocabulário do ecossistema (ex.: OpenTelemetry no commons) e
  ao padrão de pacotes em inglês (port, processor, registry).
- Remoção de DateUtil e de marcadores vazios evita API superficial não usada
  pelo fluxo de orçamento.
- commons-dto em 1.91.0-release na lib para coincidir com a API e evitar
  duas versões de DTO no classpath.
