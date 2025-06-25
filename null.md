# Solução de Rate Limiting com Istio e Redis em EKS

sistema de controle de tráfego usando o rate limiting do Istio com Redis como backend distribuído. A solução será composta por vários recursos do Istio configurados para trabalhar em conjunto.

## Arquitetura da Solução

1. **EnvoyFilter**: Para configurar o plugin de rate limiting no sidecar do Envoy
2. **Handler**: Define o adaptador do Redis para o rate limiting
3. **Instance**: Define como as métricas de requisição são capturadas
4. **Rule**: Conecta o Handler e a Instance
5. **Gateway e VirtualService**: Para configurar o ingress gateway

## 1. Configuração do EnvoyFilter (envoy-filter.yaml)

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: filter-ratelimit
  namespace: rl-test
spec:
  workloadSelector:
    labels:
      app: rl-app-teste
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: SIDECAR_INBOUND
        listener:
          filterChain:
            filter:
              name: "envoy.filters.network.http_connection_manager"
              subFilter:
                name: "envoy.filters.http.router"
      patch:
        operation: INSERT_BEFORE
        value:
          name: envoy.filters.http.ratelimit
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.filters.http.ratelimit.v3.RateLimit
            domain: rl-test-domain
            failure_mode_deny: true
            timeout: 0.25s
            rate_limit_service:
              grpc_service:
                envoy_grpc:
                  cluster_name: rate_limit_cluster
                timeout: 0.25s
    - applyTo: CLUSTER
      match:
        cluster:
          service: ratelimit.istio-system.svc.cluster.local
      patch:
        operation: ADD
        value:
          name: rate_limit_cluster
          type: STRICT_DNS
          connect_timeout: 10s
          lb_policy: ROUND_ROBIN
          http2_protocol_options: {}
          load_assignment:
            cluster_name: rate_limit_cluster
            endpoints:
            - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: ratelimit.istio-system.svc.cluster.local
                      port_value: 8081
```

**Explicação:**
- `workloadSelector`: Aplica o filtro apenas aos pods com label `app: rl-app-teste`
- `configPatches`: Modifica a configuração do Envoy:
  - Primeiro patch: Adiciona o filtro de rate limiting antes do filtro de roteamento
    - `domain`: Identificador único para as regras de rate limiting
    - `failure_mode_deny`: Se true, falhas no serviço de rate limiting resultam em bloqueio
    - Configura o serviço gRPC para comunicação com o serviço de rate limiting
  - Segundo patch: Configura o cluster para o serviço de rate limiting

## 2. Configuração do Redis Handler (redis-handler.yaml)

```yaml
apiVersion: config.istio.io/v1alpha2
kind: handler
metadata:
  name: redishandler
  namespace: rl-test
spec:
  compiledAdapter: redisquota
  params:
    redisServerUrl: "master_redis.svc.default:6106"
    connectionPoolSize: 10
    quotas:
    - name: requestcountquota.instance.istio-system
      maxAmount: 200
      validDuration: 1m
      bucketDuration: 10s
      rateLimitAlgorithm: FIXED_WINDOW
      overrides:
      - dimensions:
          destination: rl-app-teste.default.svc.cluster.local
        maxAmount: 200
```

**Explicação:**
- `compiledAdapter`: Especifica o adaptador Redis para rate limiting
- `redisServerUrl`: Endpoint do cluster Redis
- `connectionPoolSize`: Número de conexões mantidas no pool
- `quotas`: Define a política de rate limiting:
  - `maxAmount`: 200 requisições por minuto
  - `validDuration`: Janela de tempo de 1 minuto
  - `bucketDuration`: Intervalo de atualização do contador (10s)
  - `rateLimitAlgorithm`: Algoritmo de janela fixa
  - `overrides`: Aplica a política específica para o destino

## 3. Configuração da Instance (instance.yaml)

```yaml
apiVersion: config.istio.io/v1alpha2
kind: instance
metadata:
  name: requestcountquota
  namespace: rl-test
spec:
  compiledTemplate: quota
  params:
    dimensions:
      source: request.headers["x-forwarded-for"] | "unknown"
      destination: destination.labels["app"] | destination.service.name | "unknown"
      destinationVersion: destination.labels["version"] | "unknown"
```

**Explicação:**
- `compiledTemplate`: Usa o template de quota do Istio
- `params.dimensions`: Define como as requisições são agrupadas para contagem:
  - `source`: Pega o IP do cliente do header x-forwarded-for
  - `destination`: Usa o label app ou o nome do serviço de destino
  - `destinationVersion`: Usa a versão da aplicação se disponível

## 4. Configuração da Rule (rule.yaml)

```yaml
apiVersion: config.istio.io/v1alpha2
kind: rule
metadata:
  name: quota
  namespace: rl-test
spec:
  actions:
  - handler: redishandler
    instances:
    - requestcountquota
  match: destination.labels["app"] == "rl-app-teste"
```

**Explicação:**
- Conecta o Handler e a Instance:
  - `handler`: Referencia o handler Redis configurado anteriormente
  - `instances`: Usa a instance de quota definida
- `match`: Aplica a regra apenas para tráfego destinado ao app com label "rl-app-teste"

## 5. Configuração do Gateway (gateway.yaml)

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: rl-gateway
  namespace: rl-test
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "api-rl-teste.svc.default"
```

**Explicação:**
- Define o ingress gateway para o host "api-rl-teste.svc.default"
- `selector`: Usa o ingress gateway padrão do Istio
- Configura a porta 80 para tráfego HTTP

## 6. Configuração do VirtualService (virtualservice.yaml)

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: rl-virtualservice
  namespace: rl-test
spec:
  hosts:
  - "api-rl-teste.svc.default"
  gateways:
  - rl-gateway
  http:
  - route:
    - destination:
        host: rl-app-teste.default.svc.cluster.local
        port:
          number: 80
```

**Explicação:**
- Roteia o tráfego do gateway para o serviço de destino
- `hosts`: Corresponde ao host configurado no gateway
- `gateways`: Referencia o gateway criado
- `http.route`: Define o destino final do tráfego

## Implantação e Verificação

1. Aplique os recursos na ordem:
```bash
kubectl apply -f envoy-filter.yaml
kubectl apply -f redis-handler.yaml
kubectl apply -f instance.yaml
kubectl apply -f rule.yaml
kubectl apply -f gateway.yaml
kubectl apply -f virtualservice.yaml
```

2. Verifique o rate limiting:
```bash
# Teste com 200 requisições (deverá passar)
for i in {1..200}; do curl -H "Host: api-rl-teste.svc.default" http://<INGRESS_IP>/; done

# A 201ª requisição deverá falhar com HTTP 429
curl -v -H "Host: api-rl-teste.svc.default" http://<INGRESS_IP>/
```

## Considerações Importantes

1. **Redis como Backend Distribuído**:
   - O Redis armazena os contadores de forma centralizada, permitindo que o rate limiting funcione em um ambiente distribuído
   - O algoritmo FIXED_WINDOW garante consistência na contagem entre diferentes instâncias do Envoy

2. **Performance**:
   - O `bucketDuration` de 10s oferece um bom equilíbrio entre precisão e carga no Redis
   - O `connectionPoolSize` de 10 conexões é adequado para cargas moderadas

3. **Resiliência**:
   - `failure_mode_deny: true` garante que falhas no serviço de rate limiting resultem em bloqueio (fail secure)
   - O timeout de 250ms para o serviço de rate limiting evita impactos na latência

4. **Monitoramento**:
   - Métricas do Redis (uso de memória, conexões) devem ser monitoradas
   - O Istio fornece métricas nativas sobre requisições aceitas/rejeitadas

Esta solução oferece um rate limiting distribuído robusto, adequado para ambientes de produção em EKS, com a capacidade de escalar horizontalmente mantendo a consistência das contagens através do Redis.
