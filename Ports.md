Коротко:
	•	Для istio-ingressgateway и istiod набор портов выглядит типично.
	•	Для istio-egressgateway порты 5022 и 5231 — явно кастомные, их нужно подтверждать по конфигу.

Дальше — как доказать назначение каждого порта и красиво описать это в документации.

⸻

1. Что обычно значит каждый из портов

С высокой вероятностью (надо подтвердить по шагам ниже):

istio-ingressgateway (порты 15021, 80, 443)
	•	80/TCP – HTTP ingress (plaintext).
	•	443/TCP – HTTPS ingress (TLS termination или passthrough).
	•	15021/TCP – служебный health-эндпоинт Envoy (/healthz/ready), на него смотрит kube-probe и внешний L4 LB.

istio-egressgateway (порты 15021, 80, 443, 5022, 5231)
	•	15021/TCP – тот же health.
	•	80/443/TCP – egress HTTP/HTTPS из mesh через gateway (обычно HTTP CONNECT или TLS через SNI).
	•	5022/5231/TCP – не дефолтные для Istio.
	•	Скорее всего, это:
	•	либо TCP-выходы под конкретные внешние системы (БД/legacy-протоколы),
	•	либо внутренние тех. каналы под что-то своё.
	•	Выясняем по Gateway/VirtualService/Envoy-config (см. ниже).

istiod (порты 15010, 15012, 443, 15014)
	•	15010/TCP – xDS без TLS (gRPC), обычно для внутренних нужд, часто «не используется снаружи» / закрыт firewall’ом.
	•	15012/TCP – xDS по mTLS (gRPC), нужный порт для sidecar’ов и gateway’ев.
	•	443/TCP – webhook + CA:
	•	admission webhook для sidecar-инжекции и/или валидации конфигов,
	•	иногда SDS/CA.
	•	15014/TCP – HTTP-метрики и debug (/metrics, /debug/…).

⸻

2. Как наверняка выяснить назначение портов

2.1. Уровень Kubernetes Service (что опубликовано наружу)

Ты это уже сделал, но формально:

kubectl --context="$CTX" get svc istio-ingressgateway -n sm-gateways -o yaml
kubectl --context="$CTX" get svc istio-egressgateway -n sm-gateways -o yaml
kubectl --context="$CTX" get svc istiod -n istio-d -o yaml

Что смотреть:
	•	spec.ports[].port / targetPort — внешний порт → порт контейнера.
	•	name: — часто подсказка (http2, https, tls, status-port и т.п.).

Это даёт «каркас» таблицы: порт → какой сервис → куда мапится в Pod.

⸻

2.2. Уровень Istio: Gateway / VirtualService / ServiceEntry

Ингресс

kubectl --context="$CTX" get gateway -A -o yaml \
  > cluster-$CTX-istio-gateways.yaml
kubectl --context="$CTX" get virtualservice -A -o yaml \
  > cluster-$CTX-istio-virtualservices.yaml

Что смотреть в Gateway:

spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts: ["example.com", ...]
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE | PASSTHROUGH | MUTUAL
    hosts: ["example.com", ...]

Так ты фиксируешь:
80 → HTTP, 443 → HTTPS (mode X, hosts Y).

Эгресс + кастомные порты

kubectl --context="$CTX" get gateway -A -o yaml | grep -nA5 "egress"
kubectl --context="$CTX" get gateway -A -o yaml | egrep -n "5022|5231"
kubectl --context="$CTX" get virtualservice -A -o yaml | egrep -n "5022|5231"
kubectl --context="$CTX" get serviceentry -A -o yaml | egrep -n "5022|5231"

Идея:
	•	Найти, где порты 5022 и 5231 фигурируют в:
	•	gateway.spec.servers[].port.number
	•	serviceentry.spec.ports[].number
	•	virtualservice (часто только название порта используется).
	•	Оттуда будет понятно:
	•	какой протокол (TCP, TLS, HTTP),
	•	какие хосты (db.corp.local, partner-api:5022),
	•	inbound/outbound.

После этого ты уже можешь написать коллегам:

Порт 5022/TCP на egressgateway используется для выхода сервисов namespace X к системам Y (ServiceEntry Z, VirtualService Q).

⸻

2.3. Уровень Envoy: что он реально слушает и куда шлёт

Самый честный источник — сам Envoy.

Список портов (listeners) на gateway

POD_IGW=$(kubectl --context="$CTX" get pods -n sm-gateways | grep istio-ingressgateway | head -n1 | awk '{print $1}')
POD_EGW=$(kubectl --context="$CTX" get pods -n sm-gateways | grep istio-egressgateway | head -n1 | awk '{print $1}')

# Какие порты слушает ingress
istioctl --context="$CTX" proxy-config listeners "$POD_IGW" -n sm-gateways

# Какие порты слушает egress
istioctl --context="$CTX" proxy-config listeners "$POD_EGW" -n sm-gateways

Что смотреть в выводе:
	•	Столбец ADDRESS и PORT: там будут 0.0.0.0:80, 0.0.0.0:443, 0.0.0.0:5022, 0.0.0.0:5231, 0.0.0.0:15021 и т.д.
	•	TYPE/NAME/ROUTE — иногда даёт намёк (как называется listener/route).

Куда идёт трафик с этого порта
Например, для 5022:

istioctl --context="$CTX" proxy-config listeners "$POD_EGW" -n sm-gateways --port 5022 --verbose

и/или:

istioctl --context="$CTX" proxy-config routes "$POD_EGW" -n sm-gateways

Если это TCP, маршруты могут быть неявными, тогда смотри clusters:

istioctl --context="$CTX" proxy-config clusters "$POD_EGW" -n sm-gateways

Идея:
	•	По связке listeners → routes → clusters ты видишь:
	•	с порта 5022 Envoy отправляет на cluster outbound|5022||db.partner.svc.cluster.local или на внешний DNS-имя.
	•	Это уже формальная основа, чтобы описать «назначение» порта.

⸻

2.4. Для istiod

Кроме Service (что уже есть), можно посмотреть сам Deployment:

kubectl --context="$CTX" get deploy istiod -n istio-d -o yaml > istiod-deploy.yaml

Что смотреть:
	•	spec.template.spec.containers[].ports[] – какие порты открыты в Pod’е.
	•	args: / env: – иногда явно указано, что 15012 = xDS gRPC, 15014 = metrics, и т.п.

⸻

3. Как описать это коллегам (структура для документации)

Рекомендую оформить в виде таблицы + короткого текста. Пример:

Istio control-plane ports (istiod)

Port	Protocol	Direction	Used by	Purpose	Source of truth
15010	gRPC	internal (cluster)	Envoy (optionally)	xDS without TLS (legacy, обычно не используется)	Service istiod, docs, proxy-config
15012	gRPC mTLS	internal (cluster)	Envoy sidecars & gateways	secure xDS (config distribution, certs)	Service istiod, Envoy bootstrap
443	HTTPS	internal (apiserver→istiod)	Kubernetes API server (webhook)	sidecar injection, config validation, CA/SDS	MutatingWebhookConfiguration, istiod args
15014	HTTP	internal (Prometheus)	Prometheus / debug tools	control-plane metrics and debug endpoints	Service istiod, Prometheus config

Istio ingressgateway ports

Port	Protocol	Direction	Purpose	Confirmed by
80	HTTP	inbound (external→cluster)	HTTP ingress (cleartext)	Service spec, Gateway servers, listeners
443	HTTPS/TLS	inbound	HTTPS ingress (TLS termination/passthrough)	Service spec, Gateway servers, listeners
15021	HTTP	internal	health (/healthz/ready) for kube-probes & LB	Deployment probes, listeners

Istio egressgateway ports

Port	Protocol	Direction	Purpose	How to document
80	HTTP	outbound from mesh	HTTP egress via gateway	Service/Gateway/listeners
443	HTTPS/TLS	outbound	HTTPS egress via gateway	Service/Gateway/listeners
15021	HTTP	internal	healthcheck	Deployment probes/listeners
5022	TCP	outbound	custom – e.g. external system X (уточнить)	Grep в Gateway/VS/ServiceEntry, listeners
5231	TCP	outbound	custom – e.g. external system Y (уточнить)	То же

Для 5022/5231 после grep и proxy-config ты сможешь заменить «external system X/Y» на что-то осмысленное: конкретный сервис, протокол, назначение.

⸻

Если хочешь, можешь сюда скинуть обезличенный фрагмент:
	•	gateway/virtualservice/serviceentry с портами 5022/5231,
	•	кусок istioctl proxy-config listeners по этим портам,

и я помогу сформулировать аккуратное текстовое описание для документа: «почему в нашем egress-gateway есть эти порты и что через них ходит».
