# sboot-reem-resi-api-unica — integração com java-lib-reem-resi-core-multicalculo

Documento para replicar no **projeto original** o que este workspace já alinhou entre a API única, a lib core de multicalculo, o `commons-dto` e o `commons`.

## 1. Dependência Maven

| O quê | Onde |
|--------|------|
| Propriedade `resi-core-multicalculo.version` (ex.: `1.0-SNAPSHOT`) | [sboot-reem-resi-api-unica/pom.xml](sboot-reem-resi-api-unica/pom.xml) |
| Dependência `com.porto.resi:java-lib-reem-resi-core-multicalculo` | Mesmo `pom.xml`, junto de `java-lib-reem-resi-commons-dto` e `java-lib-reem-resi-commons` |

A API continua usando **interfaces da lib** no pacote `com.porto.resi.service` (`CotacaoProcessor`, `CotacaoProcessorForUpdate`), vindas do JAR da core-multicalculo.

## 2. Configuração Spring

| Classe | Função |
|--------|--------|
| [com.porto.resi.config.MulticalculoCoreConfiguration](sboot-reem-resi-api-unica/src/main/java/com/porto/resi/config/MulticalculoCoreConfiguration.java) | Declara dois beans `ProductRegistry`: um para `CotacaoProcessor` e outro para `CotacaoProcessorForUpdate` (hoje instanciados **vazios**; no projeto original falta **registrar** implementações com `ProductRegistries`). |

## 3. Camada “adapter” para ports da lib

A lib expõe ports; a API implementa com **lambdas** ou **serviços Spring**.

### 3.1 Fluxo de cotação (serviço)

| Classe | Integração com a lib |
|--------|----------------------|
| [com.porto.resi.service.impl.AbstractCotacaoProdutoService](sboot-reem-resi-api-unica/src/main/java/com/porto/resi/service/impl/AbstractCotacaoProdutoService.java) | Usa `CotacaoProdutoServiceDelegate` da lib. `CallbackGravacaoPort` = `CotacaoCallbackService::gravar`. `ValidacaoCanalPort` = lambda que chama `ValidacaoCanalService.validarCanal` quando `validarCanal` for true. Converte `CotacaoRecebidaResult` em [CotacaoRecebidaResponse](sboot-reem-resi-api-unica/src/main/java/com/porto/resi/domain/dto/rest/response/CotacaoRecebidaResponse.java) via builder (`multiOfertaId`, `ofertaId`). |
| [com.porto.resi.service.CotacaoCallbackService](sboot-reem-resi-api-unica/src/main/java/com/porto/resi/service/CotacaoCallbackService.java) | Contrato `gravar(multiOfertaId, BaseCotacaoCallbackRequest)` alinhado ao port de callback da lib. |
| [com.porto.resi.service.impl.CotacaoCallbackServiceImpl](sboot-reem-resi-api-unica/src/main/java/com/porto/resi/service/impl/CotacaoCallbackServiceImpl.java) | Implementação real de persistência/execução de callback (mapper, repositório, client HTTP assíncrono, métricas). |
| [com.porto.resi.service.ValidacaoCanalService](sboot-reem-resi-api-unica/src/main/java/com/porto/resi/service/ValidacaoCanalService.java) | Inclui `validarCanal(Integer codigoCanal)` usado pelo delegate. |
| [com.porto.resi.service.impl.ValidacaoCanalServiceImpl](sboot-reem-resi-api-unica/src/main/java/com/porto/resi/service/impl/ValidacaoCanalServiceImpl.java) | Implementação com repositório/cache/contexto de segurança. |

### 3.2 Fluxo de criação de cotações com reserva (processor)

| Classe | Integração com a lib |
|--------|----------------------|
| [com.porto.resi.service.impl.AbstractCotacaoProdutoProcessor](sboot-reem-resi-api-unica/src/main/java/com/porto/resi/service/impl/AbstractCotacaoProdutoProcessor.java) | `CotacaoProdutoProcessorSupport.criarCotacoesComReserva`: `CorrelationIdStrategies.uuidAleatorioPorOferta()`, fábrica de [ReservaNumeroOrcamentoRequest](sboot-reem-resi-api-unica/src/main/java/com/porto/resi/domain/dto/client/request/orcamento/ReservaNumeroOrcamentoRequest.java), `OrcamentoClient.reservarNumeroOrcamento(...).join()`, batch via `CotacaoCreator.createCotacoes` com [ListaInteligenciaOfertaResidencialResponseImpl](sboot-reem-resi-api-unica/src/main/java/com/porto/resi/domain/dto/client/response/ListaInteligenciaOfertaResidencialResponseImpl.java). `CotacaoNotificacaoSupport.notificarNovasCotacoes` para o [CotacaoNotifier](sboot-reem-resi-api-unica/src/main/java/com/porto/resi/service/CotacaoNotifier.java). |
| [com.porto.resi.client.OrcamentoClient](sboot-reem-resi-api-unica/src/main/java/com/porto/resi/client/OrcamentoClient.java) | Port HTTP assíncrono (`CompletableFuture`) para reserva de número de orçamento. |
| [com.porto.resi.client.config.OrcamentoClientConfig](sboot-reem-resi-api-unica/src/main/java/com/porto/resi/client/config/OrcamentoClientConfig.java) | Configuração do client (replicar wiring no projeto original). |
| [com.porto.resi.service.CotacaoCreator](sboot-reem-resi-api-unica/src/main/java/com/porto/resi/service/CotacaoCreator.java) | Contrato com `EnriquecimentoResponse` (commons-dto), `BaseInteligenciaOfertaResponse`, `ReservaNumeroOrcamentoResponse`, entidade `Cotacao`. |
| [com.porto.resi.domain.dto.client.response.orcamento.ReservaNumeroOrcamentoResponse](sboot-reem-resi-api-unica/src/main/java/com/porto/resi/domain/dto/client/response/orcamento/ReservaNumeroOrcamentoResponse.java) | DTO de resposta da reserva (neste snapshot pode estar mínimo; no original preencher campos reais). |

### 3.3 Notifier

| Classe | Integração com a lib |
|--------|----------------------|
| [com.porto.resi.service.CotacaoNotifier](sboot-reem-resi-api-unica/src/main/java/com/porto/resi/service/CotacaoNotifier.java) | Estende `CotacaoNotifierPort<Cotacao>` da lib. |
| [com.porto.resi.service.impl.AbstractCotacaoNotifier](sboot-reem-resi-api-unica/src/main/java/com/porto/resi/service/impl/AbstractCotacaoNotifier.java) | Estende `AbstractCotacaoNotifierTemplate<T, M, Cotacao>` da lib. |

### 3.4 Callback (mapeamento e infra)

| Classe | Nota |
|--------|------|
| [com.porto.resi.mapper.CotacaoCallbackMapper](sboot-reem-resi-api-unica/src/main/java/com/porto/resi/mapper/CotacaoCallbackMapper.java) (+ Impl) | `BaseCotacaoCallbackRequest` → entidade/DTOs de callback. |
| [com.porto.resi.repository.CotacaoCallbackRepository](sboot-reem-resi-api-unica/src/main/java/com/porto/resi/repository/CotacaoCallbackRepository.java) (+ Impl) | Persistência do callback. |
| [com.porto.resi.client.CotacaoCallbackClient](sboot-reem-resi-api-unica/src/main/java/com/porto/resi/client/CotacaoCallbackClient.java) (+ Impl, [CotacaoCallbackClientConfig](sboot-reem-resi-api-unica/src/main/java/com/porto/resi/client/config/CotacaoCallbackClientConfig.java)) | Execução HTTP do webhook. |

## 4. Contratos de domínio usados pelo processor

- [BaseInteligenciaOfertaResponse](sboot-reem-resi-api-unica/src/main/java/com/porto/resi/domain/dto/client/response/BaseInteligenciaOfertaResponse.java) (interface API).
- [ListaInteligenciaOfertaResidencialResponseImpl](sboot-reem-resi-api-unica/src/main/java/com/porto/resi/domain/dto/client/response/ListaInteligenciaOfertaResidencialResponseImpl.java) — adapta `List<InteligenciaOfertaResidencialResponse>` do **commons-dto** para o `CotacaoCreator`.

## 5. O que ainda não está fechado neste snapshot (para o checklist)

Os registries existem mas **não são populados**; não há uso de `ProductRegistries` nem beans `IdentifiedCotacaoProcessor`. Vários `*ServiceImpl` / `*ProcessorImpl` de produto podem estar **vazios** ou incompletos — o checklist abaixo cobre o desenvolvimento no projeto original.

---

## Checklist — o que desenvolver / completar no projeto original

### Integração core-multicalculo na API

- [ ] Garantir dependência Maven `java-lib-reem-resi-core-multicalculo` e versão alinhada ao artefato publicado.
- [ ] Copiar/ajustar `MulticalculoCoreConfiguration` e **registrar** processadores com `ProductRegistries.registrarIdentificados` e `registrarIdentificadosForUpdate` a partir de beans `Identified*`.
- [ ] Implementar classes de produto como `IdentifiedCotacaoProcessor` / `IdentifiedCotacaoProcessorForUpdate` com `codigoProdutoMulticalculo()` coerente com `MulticalculoProducts` na lib.
- [ ] Refatorar fluxos que hoje passam `CotacaoProcessor` manualmente para **resolver** via `ProductRegistry` (produto/canal).
- [ ] (Opcional) Bean `TelemetryBridge` na aplicação e, se desejado, instrumentar pontos do fluxo.

### Adapters e domínio já esboçados

- [ ] Implementação concreta de `OrcamentoClient` (Feign/WebClient/etc.) e testes de contrato.
- [ ] Completar `ReservaNumeroOrcamentoResponse` e qualquer mapeamento de resposta real do serviço de orçamento.
- [ ] Implementações de `CotacaoCreator` / notifiers por produto (classes que estendem `AbstractCotacaoNotifier` / usam `AbstractCotacaoProdutoProcessor`).
- [ ] Serviços de cotação que estendem `AbstractCotacaoProdutoService` e chamam `processarCotacoes` / `processarCotacao` com o processador adequado (via registry após refatoração).

### Callback e canal

- [ ] Validar `CotacaoCallbackServiceImpl.gravar` e fluxo `executar` com ambiente real (URL, secret, filtros, métricas).
- [ ] Revisar `ValidacaoCanalServiceImpl` (self-injection `@Lazy` para cache) e regras de negócio de canal.

### Qualidade e alinhamento de versões

- [ ] `mvn clean verify` na API com a versão da lib instalada/publicada.
- [ ] Alinhar `java-lib-reem-resi-commons-dto` (e `commons`) entre agregador Maven, CI e Nexus para evitar conflito de versões no classpath.
- [ ] Testes de integração dos fluxos BFF → API → clients.

### commons / commons-dto (referência)

- [ ] Manter DTOs e validações no **commons-dto** / **commons**; a core-multicalculo não substitui esses módulos — só os consome.
