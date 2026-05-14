package com.porto.resi.core.multicalculo.dto;

import java.util.UUID;

public record CotacaoRecebidaResult(UUID multiOfertaId, UUID ofertaId) {

    public static CotacaoRecebidaResult apenasMultiOferta(UUID multiOfertaId) {
        return new CotacaoRecebidaResult(multiOfertaId, null);
    }
}

package com.porto.resi.core.multicalculo.notifier;

import com.porto.resi.commons.dto.core.base.request.BaseCotacaoRequest;
import com.porto.resi.core.multicalculo.port.CotacaoNotifierPort;
import java.util.List;
import java.util.UUID;

public abstract class AbstractCotacaoNotifierTemplate<T extends BaseCotacaoRequest, M, C>
        implements CotacaoNotifierPort<C> {

    @Override
    public void enviarMensagemNovaCotacao(UUID multiOfertaId, List<C> cotacoes, BaseCotacaoRequest req) {
        T request = convertRequest(req);
        List<M> novasCotacoes = cotacoes.stream().map(c -> mapearCotacao(c, request)).toList();
        enviarMensagem(multiOfertaId, novasCotacoes);
    }

    protected abstract T convertRequest(BaseCotacaoRequest request);

    protected abstract M mapearCotacao(C cotacao, T request);

    protected abstract void enviarMensagem(UUID multiOfertaId, List<M> novasCotacoes);
}

package com.porto.resi.core.multicalculo.port;

import com.porto.resi.commons.dto.core.base.request.BaseCotacaoCallbackRequest;
import java.util.UUID;

public interface CallbackGravacaoPort {

    void gravar(UUID multiOfertaId, BaseCotacaoCallbackRequest callback);
}
package com.porto.resi.core.multicalculo.port;

import com.porto.resi.commons.dto.core.base.request.BaseCotacaoRequest;
import java.util.List;
import java.util.UUID;

public interface CotacaoNotifierPort<C> {

    void enviarMensagemNovaCotacao(UUID multiOfertaId, List<C> cotacoes, BaseCotacaoRequest req);
}
package com.porto.resi.core.multicalculo.port;

public interface OfertaPort<S, R> {

    R enquadrar(S solicitacao);
}
package com.porto.resi.core.multicalculo.port;

public interface ValidacaoCanalPort {

    void validarSeNecessario(Integer codigoCanal, boolean validar);
}

package com.porto.resi.core.multicalculo.processor;

import java.util.Objects;
import java.util.UUID;
import java.util.function.Function;

public final class CorrelationIdStrategies {

    private CorrelationIdStrategies() {}

    public static <O> CorrelationIdStrategy<O> uuidAleatorioPorOferta() {
        return ofertas -> ofertas.stream().map(o -> UUID.randomUUID()).toList();
    }

    public static <O> CorrelationIdStrategy<O> estavelPorExtrator(Function<O, UUID> idPorOferta) {
        Objects.requireNonNull(idPorOferta, "idPorOferta");
        return ofertas -> ofertas.stream().map(idPorOferta).toList();
    }
}
package com.porto.resi.core.multicalculo.processor;

import java.util.List;
import java.util.UUID;

@FunctionalInterface
public interface CorrelationIdStrategy<O> {

    List<UUID> buildIds(List<O> ofertas);
}

package com.porto.resi.core.multicalculo.processor;

import com.porto.resi.commons.dto.core.base.request.BaseCotacaoRequest;
import java.util.List;
import java.util.UUID;

@FunctionalInterface
public interface CotacaoBatchFactory<T extends BaseCotacaoRequest, O, E, RS, C> {

    List<C> criar(
            UUID multiOfertaId, List<O> ofertas, E enriquecimento, T request, RS reservaNumeroOrcamentoResponse);
}
package com.porto.resi.core.multicalculo.processor;

import com.porto.resi.commons.dto.core.base.request.BaseCotacaoRequest;
import com.porto.resi.core.multicalculo.port.CotacaoNotifierPort;
import java.util.List;
import java.util.UUID;

public final class CotacaoNotificacaoSupport {

    private CotacaoNotificacaoSupport() {}

    public static <T extends BaseCotacaoRequest, C> void notificarNovasCotacoes(
            UUID multiOfertaId,
            List<C> cotacoes,
            T request,
            CotacaoNotifierPort<C> cotacaoNotifier) {
        cotacaoNotifier.enviarMensagemNovaCotacao(multiOfertaId, cotacoes, request);
    }
}

package com.porto.resi.core.multicalculo.processor;

import com.porto.resi.commons.dto.core.base.request.BaseCotacaoRequest;
import java.util.List;
import java.util.UUID;

public final class CotacaoProdutoProcessorSupport {

    private CotacaoProdutoProcessorSupport() {}

    public static <T extends BaseCotacaoRequest, O, RQ, RS, C, E> List<C> criarCotacoesComReserva(
            UUID multiOfertaId,
            T request,
            List<O> ofertas,
            E enriquecimentoResponse,
            CorrelationIdStrategy<O> correlationIdStrategy,
            ReservaPedidoFactory<T, RQ> pedidoFactory,
            ReservaExecutor<RQ, RS> reservaExecutor,
            CotacaoBatchFactory<T, O, E, RS, C> cotacaoBatchFactory,
            boolean recalculo) {

        List<UUID> correlationIds = correlationIdStrategy.buildIds(ofertas);
        RQ pedido = pedidoFactory.criar(correlationIds, request, recalculo);
        RS reserva = reservaExecutor.executar(multiOfertaId, pedido);
        return cotacaoBatchFactory.criar(multiOfertaId, ofertas, enriquecimentoResponse, request, reserva);
    }
}

package com.porto.resi.core.multicalculo.processor;

import java.util.UUID;

@FunctionalInterface
public interface ReservaExecutor<RQ, RS> {

    RS executar(UUID multiOfertaId, RQ pedido);
}

package com.porto.resi.core.multicalculo.processor;

import com.porto.resi.commons.dto.core.base.request.BaseCotacaoRequest;
import java.util.List;
import java.util.UUID;

@FunctionalInterface
public interface ReservaPedidoFactory<T extends BaseCotacaoRequest, RQ> {

    RQ criar(List<UUID> correlationIds, T request, boolean recalculo);
}

package com.porto.resi.core.multicalculo.registry;

import com.porto.resi.service.CotacaoProcessor;

public interface IdentifiedCotacaoProcessor extends CotacaoProcessor {

    String codigoProdutoMulticalculo();
}

package com.porto.resi.core.multicalculo.registry;

import com.porto.resi.service.CotacaoProcessorForUpdate;

public interface IdentifiedCotacaoProcessorForUpdate extends CotacaoProcessorForUpdate {

    String codigoProdutoMulticalculo();
}

package com.porto.resi.core.multicalculo.registry;

import com.porto.resi.service.CotacaoProcessor;
import com.porto.resi.service.CotacaoProcessorForUpdate;
import java.util.Collection;
import java.util.Objects;

public final class ProductRegistries {

    private ProductRegistries() {}

    public static void registrarIdentificados(
            ProductRegistry<CotacaoProcessor> destino,
            Collection<? extends IdentifiedCotacaoProcessor> implementacoes) {
        Objects.requireNonNull(destino, "destino");
        Objects.requireNonNull(implementacoes, "implementacoes");
        for (IdentifiedCotacaoProcessor p : implementacoes) {
            destino.registrar(p.codigoProdutoMulticalculo(), p);
        }
    }

    public static void registrarIdentificadosForUpdate(
            ProductRegistry<CotacaoProcessorForUpdate> destino,
            Collection<? extends IdentifiedCotacaoProcessorForUpdate> implementacoes) {
        Objects.requireNonNull(destino, "destino");
        Objects.requireNonNull(implementacoes, "implementacoes");
        for (IdentifiedCotacaoProcessorForUpdate p : implementacoes) {
            destino.registrar(p.codigoProdutoMulticalculo(), p);
        }
    }
}

package com.porto.resi.core.multicalculo.registry;

import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

public final class ProductRegistry<V> {

    private final Map<String, V> valores = new ConcurrentHashMap<>();

    public void registrar(String codigoProduto, V valor) {
        if (codigoProduto == null || codigoProduto.trim().isEmpty()) {
            throw new IllegalArgumentException("codigoProduto invalido");
        }
        if (valor == null) {
            throw new IllegalArgumentException("valor nao pode ser nulo");
        }
        valores.put(normalizar(codigoProduto), valor);
    }

    public Optional<V> buscar(String codigoProduto) {
        if (codigoProduto == null) {
            return Optional.empty();
        }
        return Optional.ofNullable(valores.get(normalizar(codigoProduto)));
    }

    public V resolverObrigatorio(String codigoProduto) {
        return buscar(codigoProduto).orElseThrow(() -> new IllegalStateException(
                "Nenhuma implementacao registrada para o produto: " + codigoProduto));
    }

    public boolean contem(String codigoProduto) {
        return codigoProduto != null && valores.containsKey(normalizar(codigoProduto));
    }

    private static String normalizar(String codigo) {
        return codigo.trim().toLowerCase();
    }
}

package com.porto.resi.core.multicalculo.service;

import com.porto.resi.commons.dto.core.base.request.BaseCotacaoCallbackRequest;
import com.porto.resi.commons.dto.core.base.request.BaseCotacaoRequest;
import com.porto.resi.core.multicalculo.dto.CotacaoRecebidaResult;
import com.porto.resi.core.multicalculo.port.CallbackGravacaoPort;
import com.porto.resi.service.CotacaoProcessor;
import com.porto.resi.service.CotacaoProcessorForUpdate;
import com.porto.resi.core.multicalculo.port.ValidacaoCanalPort;
import java.util.UUID;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public final class CotacaoProdutoServiceDelegate<T extends BaseCotacaoRequest> {

    private static final Logger log = LoggerFactory.getLogger(CotacaoProdutoServiceDelegate.class);

    private final CallbackGravacaoPort callbackGravacaoPort;
    private final ValidacaoCanalPort validacaoCanalPort;

    public CotacaoProdutoServiceDelegate(
            CallbackGravacaoPort callbackGravacaoPort, ValidacaoCanalPort validacaoCanalPort) {
        this.callbackGravacaoPort = callbackGravacaoPort;
        this.validacaoCanalPort = validacaoCanalPort;
    }

    public CotacaoRecebidaResult processarCotacoes(
            T request,
            CotacaoProcessor processor,
            boolean validarCanal,
            boolean callbackOpcional,
            BaseCotacaoCallbackRequest callback) {

        validacaoCanalPort.validarSeNecessario(request.getCodigoCanal(), validarCanal);

        UUID multiOfertaId = processor.processCotacoes(request);
        log.info("Cotação processada com sucesso. MultiOferta ID: {}", multiOfertaId);

        gravarCallback(multiOfertaId, callbackOpcional, callback);

        return CotacaoRecebidaResult.apenasMultiOferta(multiOfertaId);
    }

    public CotacaoRecebidaResult processarCotacao(
            UUID multiOfertaId,
            UUID cotacaoId,
            T request,
            CotacaoProcessorForUpdate processorForUpdate,
            boolean validarCanal,
            boolean callbackOpcional,
            BaseCotacaoCallbackRequest callback) {

        validacaoCanalPort.validarSeNecessario(request.getCodigoCanal(), validarCanal);

        UUID ofertaId = processorForUpdate.processCotacao(multiOfertaId, cotacaoId, request);
        log.info("Cotação processada com sucesso. MultiOferta ID: {}", multiOfertaId);

        gravarCallback(multiOfertaId, callbackOpcional, callback);

        return new CotacaoRecebidaResult(multiOfertaId, ofertaId);
    }

    private void gravarCallback(
            UUID multiOfertaId, boolean callbackOpcional, BaseCotacaoCallbackRequest callback) {
        if (callbackOpcional && callback == null) {
            return;
        }
        callbackGravacaoPort.gravar(multiOfertaId, callback);
        log.info("Callback gravado para o MultiOferta ID: {}", multiOfertaId);
    }
}

package com.porto.resi.core.multicalculo.telemetry;

public final class NoOpTelemetryBridge implements TelemetryBridge {

    public static final NoOpTelemetryBridge INSTANCE = new NoOpTelemetryBridge();

    private NoOpTelemetryBridge() {}

    @Override
    public void recordEvent(String name, long durationNanos) {}
}

package com.porto.resi.core.multicalculo;

public final class MulticalculoProducts {

    public static final String RESIDENCIAL = "residencial";
    public static final String ESSENCIAL = "essencial";
    public static final String IMOBILIARIO = "imobiliario";

    private MulticalculoProducts() {}
}

package com.porto.resi.service;

import com.porto.resi.commons.dto.core.base.request.BaseCotacaoRequest;
import java.util.UUID;

public interface CotacaoProcessor {

    UUID processCotacoes(BaseCotacaoRequest request);
}

package com.porto.resi.service;

import com.porto.resi.commons.dto.core.base.request.BaseCotacaoRequest;
import java.util.UUID;

public interface CotacaoProcessorForUpdate {

    UUID processCotacao(UUID multiOfertaId, UUID cotacaoId, BaseCotacaoRequest request);
}
