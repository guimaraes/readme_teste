# DEVPLAN — Desacoplamento da `java-lib-reem-resi-core-multicalculo` do `java-lib-reem-resi-commons-dto`

## 1. Contexto e decisão da liderança

### 1.1 Síntese do diálogo (Tech Lead)

| Participante | Mensagem-chave |
|----------------|----------------|
| Liderança | Referências à `commons-dto` dentro da core-multicalculo não são desejáveis; não levar essa lib para dentro da outra. |
| Liderança | Proposta inicial: extrair **classes base** do `commons-dto` para a **lib-core** e manter **classes filhas** apenas na **api-unica**, para matar acoplamento com `commons-dto`. A core é **template** da api-unica **sem depender de produto** (residencial, essencial). |
| Liderança | Exemplo desejado: um serviço Spring Boot **do zero** importa só a lib-core, define **seus** DTOs, implementa **suas** interfaces e já consegue um fluxo de cálculo. |
| Liderança | Refinamento: as bases **não seriam compartilhadas** com outros sistemas; ficariam **só na lib-core**. Opinião forte: `commons-dto` “nunca deveria existir”, só gera acoplamento, pouca utilidade percebida. |
| Time (você) | Core como **mecanismo** (ports, delegate, registry, reserva, notificação), sem produto e, se possível, **sem** o pacote pesado de contratos do `commons-dto`, para não amarrar **release da core** à evolução de **todos** os DTOs. |
| Time (você) | Se as bases ainda forem contrato com BFF/outros, não duplicar sem critério: ou módulo **fino** só de contratos, ou core com **interfaces / tipos mínimos** (canal, orçamento, ids) e **api-unica mapeia na borda** o `BaseCotacaoRequest` do `commons-dto` → tipos da core, **sem** a core depender do `commons-dto` e **sem** quebrar a fonte única do JSON na entrada HTTP. |
| Liderança | **“Sim, pode seguir nessa linha.”** |

### 1.2 Decisão consolidada (o que “seguir nessa linha” significa em engenharia)

1. **Remover dependência Maven** de `java-lib-reem-resi-commons-dto` da `java-lib-reem-resi-core-multicalculo`.
2. A core expõe **tipos mínimos próprios** (records, interfaces ou classes leves **sem** anotações Jackson de contrato HTTP, **sem** validações que puxem `java-lib-reem-resi-commons`, **sem** OpenAPI acoplado ao jar da core), contendo **apenas** o que os templates usam hoje: canal, número de orçamento externo, flags de recálculo, ids (`UUID`), e o payload mínimo de callback gravável.
3. A **api-unica** (e qualquer outro host) continua sendo o lugar de **contrato de integração** (REST, JSON, Bean Validation, Swagger): pode continuar usando `commons-dto` **temporariamente** na camada de controller/DTO de entrada, e aplica **mapeamento explícito na borda** para os tipos da core antes de chamar delegate/processor/notifier.
4. **Produto-específico** (residencial, essencial, imobiliário): DTOs filhos, mappers e implementações de ports **permanecem na api-unica** (ou em módulos de produto futuros), não na core.
5. A visão de **“serviço do zero + core”** fica atendida: um terceiro projeto não precisa do `commons-dto`; só da core + seus próprios DTOs + implementações.

### 1.3 Alinhamento com a frase “bases na core, filhas na API”

Há duas leituras possíveis:

| Leitura | Onde fica o “contrato HTTP” |
|---------|-----------------------------|
| A) **Duplicar** bases na core (espelho do que hoje está no commons-dto) | Risco de divergência com BFF se ambos serializarem o mesmo JSON com tipos diferentes. |
| B) **Tipos mínimos de domínio do multicalculo** na core (não são o DTO REST público) | Contrato HTTP continua **só** na api-unica (ou módulo de API); core não serializa o corpo da requisição pública. |

A linha aprovada pelo diálogo final corresponde à **leitura B**: a core não “substitui” o `commons-dto` como contrato de wire; ela **substitui** o uso do `commons-dto` como **dependência de compilação** da core. A opinião da liderança sobre o `commons-dto` no longo prazo fica registrada na secção 8 (estratégia corporativa), sem bloquear a fase 1.

---

## 2. Estado atual (inventário técnico)

### 2.1 Dependência Maven

- [java-lib-reem-resi-core-multicalculo/pom.xml](pom.xml): dependência direta em `java-lib-reem-resi-commons-dto`.

### 2.2 Uso de pacotes `com.porto.resi.commons.dto...` no código da core (compilação)

| Artefacto | Tipos do commons-dto |
|-----------|----------------------|
| `CotacaoProcessor`, `CotacaoProcessorForUpdate` | `BaseCotacaoRequest` |
| `CotacaoProdutoServiceDelegate` | `BaseCotacaoRequest`, `BaseCotacaoCallbackRequest` |
| `CallbackGravacaoPort` | `BaseCotacaoCallbackRequest` |
| `CotacaoNotifierPort`, `AbstractCotacaoNotifierTemplate`, `CotacaoNotificacaoSupport` | `BaseCotacaoRequest` |
| `CotacaoProdutoProcessorSupport`, `CotacaoBatchFactory`, `ReservaPedidoFactory` | `BaseCotacaoRequest` |
| Testes unitários | Mesmos tipos |

Não há hoje referência a `EnriquecimentoResponse` ou DTOs de oferta **dentro** do código principal da core (o genérico `E` no processor já isola enriquecimento); o acoplamento crítico é **`BaseCotacaoRequest`** e **`BaseCotacaoCallbackRequest`**.

---

## 3. Objetivos mensuráveis

| ID | Objetivo | Critério de aceite |
|----|-----------|-------------------|
| O1 | Core **sem** dependência `java-lib-reem-resi-commons-dto` | `mvn dependency:tree` na core não lista `commons-dto`. |
| O2 | Core **sem** dependência `java-lib-reem-resi-commons` | Nenhum tipo da core referencia validações do commons no classpath da core. |
| O3 | Templates compilam só com tipos **core** + SLF4J (+ test scope Boot se mantido). | `mvn -q test` na core verde. |
| O4 | Api-unica continua expondo contratos atuais (ou plano de migração documentado) | Testes de contrato / regressão acordados com time de integração. |
| O5 | Exemplo “serviço do zero” documentado | Secção no README ou neste DEVPLAN com pacotes mínimos e interfaces a implementar. |

---

## 4. Desenho alvo (detalhado)

### 4.1 Novos tipos na core (nomes sugeridos — ajustar ao padrão do time)

Pacote sugerido: `com.porto.resi.core.multicalculo.model` (ou `...domain`).

| Tipo | Responsabilidade | Campos mínimos (derivados do uso atual) |
|------|-------------------|----------------------------------------|
| `CotacaoOperacaoContext` (ou nome equivalente) | Substituir genericamente o papel de `BaseCotacaoRequest` **dentro** da core | `Integer codigoCanal`, `Long numeroOrcamentoExterno` (nome alinhado ao getter atual `getNumeroOrcamento()`), demais campos **somente** se algum método da cadeia atual os ler (auditar `ReservaNumeroOrcamentoRequest` factory e delegate). |
| `CotacaoCallbackGravacaoInput` (record) | Substituir `BaseCotacaoCallbackRequest` no port `CallbackGravacaoPort` | Campos usados pelo fluxo de gravar: `verbo`, `statusCode`, `secret`, `url` — espelho **lógico**, sem `@Schema` / `@ValidHttpVerbo` na core. |
| Opcional: `CotacaoCallbackGravacaoPort` genérico | Se no futuro o callback carregar payload opaco | `void gravar(UUID multiOfertaId, Map<String,Object> payload)` — só se a liderança quiser máximo desacoplamento; senão manter record explícito. |

**Genéricos:** onde hoje existe `<T extends BaseCotacaoRequest>`, passar a `<C extends CotacaoOperacaoContext>` (ou interface `CotacaoOperacaoContext` implementada na API por um adapter/wrapper).

### 4.2 Interfaces de processamento

- `CotacaoProcessor` / `CotacaoProcessorForUpdate`: assinaturas passam a usar `CotacaoOperacaoContext` (ou o nome final), **não** `BaseCotacaoRequest`.
- Implementações na api-unica recebem DTOs REST e ou implementam a interface recebendo já o tipo core (preferível: **adapter** que implementa `CotacaoProcessor` e recebe dependências produto-internas).

### 4.3 Ports

- `CallbackGravacaoPort.gravar(UUID, CotacaoCallbackGravacaoInput)` — tipo core.
- `CotacaoNotifierPort` / template: método que hoje recebe `BaseCotacaoRequest` passa a receber `CotacaoOperacaoContext` (ou superinterface mínima).

### 4.4 Borda na api-unica

| Componente | Ação |
|--------------|------|
| Controllers / DTOs de entrada | Podem continuar com `BaseCotacaoRequest` do `commons-dto` **até** decisão de trocar o contrato HTTP. |
| Mapper dedicado | `CotacaoCoreMapper` (nome exemplo): `BaseCotacaoRequest` → `CotacaoOperacaoContext`; `BaseCotacaoCallbackRequest` → `CotacaoCallbackGravacaoInput`. |
| `AbstractCotacaoProdutoService` | Após mapeamento, chama `CotacaoProdutoServiceDelegate` com tipos core. |
| `AbstractCotacaoProdutoProcessor` / `CotacaoCreator` | `CotacaoCreator` hoje usa `BaseCotacaoRequest`; ou o creator recebe tipo core, ou permanece na API com assinatura commons-dto e só a chamada ao `CotacaoProdutoProcessorSupport` usa tipo core — **decisão de fase**: minimizar duplicação escolhendo **um** ponto de conversão (preferencialmente na entrada do serviço de aplicação). |

### 4.5 Testes da core

- Substituir fixtures que hoje instanciam subclasses de `BaseCotacaoRequest` por **records** ou builders de `CotacaoOperacaoContext` **dentro** da core.
- Nenhum import `com.porto.resi.commons.dto` nos testes da core.

---

## 5. Plano de fases (minucioso)

### Fase 0 — Preparação (sem mudança de comportamento)

- [ ] Listar **todos** os getters de `BaseCotacaoRequest` usados indiretamente pela core (grep em `java-lib-reem-resi-core-multicalculo` e em subclasses na api-unica que alimentam o delegate).
- [ ] Idem para `BaseCotacaoCallbackRequest`.
- [ ] Documentar no PR/commit a matriz campo-a-campo: commons-dto → tipo core.

### Fase 1 — Introduzir tipos core (additive)

- [ ] Criar pacote `...core.multicalculo.model` com records/interfaces acordados.
- [ ] Adicionar **sobrecargas** ou novos métodos internos que aceitam tipos core **em paralelo** aos antigos (opcional, se quiser migração incremental); **ou** big-bang na core com quebra de API do jar (definir versionamento semver: **2.0.0** se já houve release consumido).

### Fase 2 — Refatorar a core para tipos próprios

- [ ] Alterar `CotacaoProcessor`, `CotacaoProcessorForUpdate`, delegate, ports, notifier template, processor support, factories.
- [ ] Remover dependência `commons-dto` do `pom.xml` da core.
- [ ] Ajustar todos os testes da core.
- [ ] `mvn clean install` na core.

### Fase 3 — Api-unica (borda)

- [ ] Implementar mappers commons-dto → tipos core.
- [ ] Ajustar `AbstractCotacaoProdutoService`, `AbstractCotacaoProdutoProcessor`, notifiers, services concretos, `CotacaoCallbackService` se a assinatura pública `gravar` continuar em commons-dto (mapper na primeira linha do método).
- [ ] Ajustar `MulticalculoCoreConfiguration` e beans `Identified*` para novas assinaturas.
- [ ] `mvn clean verify` na api-unica.

### Fase 4 — Regressão e governança

- [ ] Testes de integração / contrato com BFF (se existirem).
- [ ] Atualizar [DEVPLAN-core-multicalculo.md](DEVPLAN-core-multicalculo.md): remover menção a dependência obrigatória de commons-dto na core; referenciar este documento.
- [ ] Comunicar versão nova da core aos consumidores (breaking change se aplicável).

### Fase 5 (opcional / estratégico) — Posição sobre `commons-dto`

- [ ] Workshop com arquitetura: a frase “commons-dto nunca deveria existir” implica **descontinuar** o artefato corporativo ou **restringir** a APIs legadas apenas na api-unica até migração total.
- [ ] Se descontinuar: plano de migração dos outros consumidores do `commons-dto` (fora do escopo deste DEVPLAN, mas **bloqueia** remoção global).

---

## 6. Riscos e mitigações

| Risco | Mitigação |
|-------|-----------|
| Esquecer um campo usado só em runtime na reserva | Matriz na Fase 0 + teste de integração na api-unica com payload real. |
| Duplicação semântica entre JSON público e modelo core | Uma única camada de mapper na borda; testes de mapper com fixtures JSON. |
| Breaking change para quem já consumiu a core 1.x | Bump de versão major; release notes. |
| `BaseCotacaoRequest` no commons-dto referencia validações do `commons` | Core **não** replica essas anotações; validação permanece na api-unica antes do map. |

---

## 7. Checklist rápido (TL;DR)

- [ ] Tipos mínimos na core (`CotacaoOperacaoContext`, `CotacaoCallbackGravacaoInput`, …).
- [ ] Refatorar todas as referências `BaseCotacaoRequest` / `BaseCotacaoCallbackRequest` na core.
- [ ] Remover `java-lib-reem-resi-commons-dto` do `pom` da core.
- [ ] Mappers na api-unica.
- [ ] Ajustar implementações e testes da api-unica.
- [ ] Versionamento e comunicação de breaking change.
- [ ] Atualizar documentação e alinhar roadmap do `commons-dto` com arquitetura.

---

## 8. Nota de alinhamento com a opinião da liderança sobre `commons-dto`

O pedido aprovado (“core sem commons-dto + mapeamento na borda”) **não exige**, por si só, a **extinção imediata** do artefato `java-lib-reem-resi-commons-dto` em todo o ecossistema. Cumpre o objetivo de **desacoplar a core** e permitir o exemplo “serviço do zero”. A **extinção** do `commons-dto` é uma **decisão de portfólio** (vários consumidores, BFFs, versões publicadas) e deve ser tratada como **épico separado**, com este DEVPLAN como pré-requisito ou facilitador.

---

## 9. Referências internas

- Core atual: [CotacaoProdutoServiceDelegate](src/main/java/com/porto/resi/core/multicalculo/service/CotacaoProdutoServiceDelegate.java), [CallbackGravacaoPort](src/main/java/com/porto/resi/core/multicalculo/port/CallbackGravacaoPort.java), processors e notifier sob `src/main/java/com/porto/resi/core/multicalculo/`.
- DEVPLAN anterior de arquitetura: [DEVPLAN-core-multicalculo.md](DEVPLAN-core-multicalculo.md).

---

*Documento gerado para refletir conversa com liderança e direção técnica aprovada: **tipos mínimos na core, mapeamento na api-unica, core sem dependência do commons-dto**.*
