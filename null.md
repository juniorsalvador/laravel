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

---
-

---

Sim, existe uma maneira de fazer o deployment sem usar um registry de imagens Docker. Você pode criar o pod diretamente com o código da aplicação usando um `ConfigMap` para armazenar o script Python e montá-lo como um volume no container. Aqui está como fazer:

## Solução sem Registry de Imagens

### 1. ConfigMap com o Código da Aplicação (app-configmap.yaml)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: pod-info-app-code
data:
  app.py: |
    from flask import Flask, request, jsonify
    import os

    app = Flask(__name__)

    @app.route('/')
    def hello():
        pod_name = os.getenv('HOSTNAME', 'unknown')
        x_sigla = request.headers.get('x-sigla', 'not-provided')
        
        response_data = {
            'pod_name': pod_name,
            'status_code': 200,
            'x_sigla': x_sigla
        }
        
        return jsonify(response_data)

    if __name__ == '__main__':
        app.run(host='0.0.0.0', port=8080)
```

### 2. Deployment usando Imagem Python Oficial (deployment-direct.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pod-info-app
  labels:
    app: pod-info
spec:
  replicas: 3
  selector:
    matchLabels:
      app: pod-info
  template:
    metadata:
      labels:
        app: pod-info
    spec:
      containers:
      - name: pod-info-container
        image: python:3.9-slim
        command: ["python", "/app/app.py"]
        ports:
        - containerPort: 8080
        env:
        - name: HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        volumeMounts:
        - name: app-code
          mountPath: /app
        # Instala as dependências no startup
        lifecycle:
          postStart:
            exec:
              command: ["sh", "-c", "pip install flask && python -c 'import flask'"]
      volumes:
      - name: app-code
        configMap:
          name: pod-info-app-code
```

### 3. Service (service-direct.yaml) - Igual ao anterior

```yaml
apiVersion: v1
kind: Service
metadata:
  name: pod-info-service
spec:
  selector:
    app: pod-info
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

## Como Implantar:

1. Aplique os arquivos YAML:
```bash
kubectl apply -f app-configmap.yaml
kubectl apply -f deployment-direct.yaml
kubectl apply -f service-direct.yaml
```

2. Verifique se os pods estão rodando:
```bash
kubectl get pods
```

3. Teste a aplicação:
```bash
kubectl run -it --rm test --image=curlimages/curl --restart=Never -- sh
curl -H "x-sigla: TEST123" pod-info-service
```

## Vantagens desta Abordagem:

1. **Sem necessidade de registry Docker**
2. **Fácil atualização do código** - basta atualizar o ConfigMap
3. **Usa imagem oficial Python** (mais segura que construir sua própria)

## Limitações:

1. **Tempo de startup mais lento** - instala as dependências a cada inicialização
2. **Não é ideal para produção** - melhor para desenvolvimento/testes
3. **Código exposto no ConfigMap** - não é o ideal para código proprietário

## Alternativa Avançada (usando InitContainer):

Para melhorar o desempenho, você pode usar um InitContainer para instalar as dependências uma vez:

```yaml
# Adicione ao template do deployment
initContainers:
- name: install-deps
  image: python:3.9-slim
  command: ["sh", "-c", "pip install flask && mkdir -p /deps && cp -r /usr/local/lib/python3.9/site-packages /deps"]
  volumeMounts:
  - name: python-deps
    mountPath: /deps
volumes:
- name: python-deps
  emptyDir: {}
```

E no container principal:
```yaml
env:
- name: PYTHONPATH
  value: /deps/site-packages
volumeMounts:
- name: python-deps
  mountPath: /deps
```

Esta solução elimina a necessidade de registry enquanto mantém um tempo de startup rápido!


---
-
---

O erro ocorre porque a partir do Istio 1.7+, a API `config.istio.io/v1alpha2` foi depreciada e substituída pelo novo modelo de configuração do Istio usando a API `telemetry.istio.io/v1alpha1`. Vou corrigir a configuração para a versão mais recente do Istio.

## Configuração Corrigida para Istio 1.7+

### 1. EnvoyFilter (envoy-filter.yaml) - Mantém o mesmo

### 2. RedisQuota Custom Resource (redisquota.yaml)

```yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: redis-rate-limit
  namespace: rl-test
spec:
  selector:
    matchLabels:
      app: rl-app-teste
  overrides:
  - match:
      any: true
    providers:
      - name: redisquota
    redisquota:
      redisServerUrl: "master_redis.svc.default:6106"
      timeout: 0.25s
      connectionPoolSize: 10
      quotas:
      - name: requestcountquota
        maxAmount: 200
        validDuration: 1m
        bucketDuration: 10s
        rateLimitAlgorithm: FIXED_WINDOW
        overrides:
        - dimensions:
            destination: rl-app-teste.default.svc.cluster.local
          maxAmount: 200
```

### 3. ServiceEntry para Redis (redis-serviceentry.yaml)

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: redis-external
  namespace: rl-test
spec:
  hosts:
  - master_redis.svc.default
  ports:
  - number: 6106
    name: tcp-redis
    protocol: TCP
  resolution: DNS
  location: MESH_INTERNAL
```

### 4. DestinationRule para Redis (redis-destinationrule.yaml)

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: redis-dr
  namespace: rl-test
spec:
  host: master_redis.svc.default
  trafficPolicy:
    tls:
      mode: DISABLE
```

## Explicação das Mudanças:

1. **Nova API `telemetry.istio.io/v1alpha1`**:
   - Substitui a antiga API `config.istio.io/v1alpha2`
   - Integra melhor com o sistema de telemetria do Istio

2. **Estrutura do RedisQuota**:
   - Agora faz parte da configuração de Telemetry
   - Mantém os mesmos parâmetros, mas com estrutura diferente

3. **ServiceEntry e DestinationRule**:
   - Necessários para que o Istio possa acessar o Redis corretamente
   - Definem como o serviço Redis é descoberto e acessado

4. **Seletor**:
   - Aplica a configuração apenas aos pods com label `app: rl-app-teste`

## Como Aplicar:

```bash
kubectl apply -f envoy-filter.yaml
kubectl apply -f redis-serviceentry.yaml
kubectl apply -f redis-destinationrule.yaml
kubectl apply -f redisquota.yaml
```

## Verificação:

1. Verifique se a configuração foi aplicada:
```bash
kubectl get telemetry -n rl-test
```

2. Verifique os logs do sidecar do Istio:
```bash
kubectl logs -l app=rl-app-teste -n rl-test -c istio-proxy | grep ratelimit
```

Esta configuração é compatível com as versões mais recentes do Istio (1.7+) e mantém a mesma funcionalidade de rate limiting usando Redis como backend distribuído.
