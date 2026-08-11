## Что такое API Gateway (и чем он не является)

**API Gateway** - это **программный пограничный слой**, через который проходит внешний (и часто внутренний) трафик к бэкенд-API. Его задача - **применять политики к запросам** перед тем, как они попадут к микросервисам:

- аутентификация/авторизация (OAuth2/OIDC/JWT, mTLS, RBAC/ABAC),
    
- маршрутизация и версионирование API,
    
- ограничение частоты (rate limiting), квоты, анти-DDoS,
    
- трансформации (headers/body, REST↔gRPC/GraphQL), нормализация ошибок,w
    
- кэширование и компрессия,
    
- надёжность на уровне клиента: timeouts/retries/backoff/hedging/circuit-breaker,
    
- наблюдаемость: трассировка, метрики RED, структурированные логи,
    
- трафик-менеджмент: canary/blue-green, A/B, mirroring/shadowing.
    

**Не путать:**

- **L7 LB / Ingress** - про доставку и базовую маршрутизацию.
    
- **Service Mesh** - политика **межсервисных** вызовов (mTLS, retry/circuit, telemetry) через sidecar’ы.
    
- **BFF/GraphQL** - доменно-специфический "фасад"/агрегатор для конкретного клиента (mobile/web).
    

Часто **Gateway+LB+Mesh** работают **совместно**, а не "вместо".

## Зачем он нужен (и когда - нет)

### Ключевые мотивации

- **Единая точка применения политики безопасности** и лимитов, уменьшение "размазанности" auth/ratelimit по сервисам.
    

- **Унификация контрактов** и эволюция API: версионирование, deprecation, совместимость.
    

- **Снижение когнитивной нагрузки** на команды бэкенда: кросс-срезы (observability, трафик-менеджмент) вынесены наружу.
    

- **Контроль стоимости**: кэш, квоты/тарифы, биллинг и аналитика использования API.
    

### Когда не нужен

- Один небольшой сервис, нет внешней экспозиции - достаточно L7-LB/Ingress.
    

- Критичный путь с микросекундными бюджетами и **нет** сквозных политик - любой прокси добавит p99-латентность.
    

- Вы уже используете **service mesh** и все политики применяются на **edge-proxy** (например, Envoy Gateway) - отдельный "бог-шлюз" будет избыточен.
    

## НФТ и SLO для API Gateway

- **Latency overhead (gateway-leg)**: p95 ≤ 5–10 мс (внешний трафик), p99 ≤ 15–20 мс; внутренняя зона - ещё жёстче.
    

- **Availability**: ≥ 99.99% для внешнего контура; актив-актив по регионам.
    

- **Throughput**: горизонтальная масштабируемость (линейный рост RPS на узел), zero-config-reload.
    

- **Security**: mTLS внутрь, строгая валидация JWT (clock-skew, exp/nbf/aud, key-rotation), WAF-интеграция.
    

- **Multi-tenant**: изоляция лимитов/квот/ключей, "просачивание" tenant-id в логи/трейсы.
    

- **Config & Ops**: GitOps/динам. xDS, canary для правил, быстрый rollback; SLO-дашборды (RED).
    

## Типичные функции и порядок применения политик

**Рекомендуемый конвейер (edge path):**

1. **TLS termination** (H2/H3→H1.1 upstream при необходимости), HSTS, ALPN, SNI-routing.

2. **DDoS/бот-контроль** (L7 rate, token-challenge, JA3/UA-fingerprints).

3. **Аутентификация**: OAuth2/OIDC (JWT verify по JWKS **локально**), API-keys, mTLS.

4. **Авторизация**: RBAC/ABAC, атрибуты в токене (scopes/roles/claims), PDP (OPA/Styra) при сложной политике.

5. **Rate limiting/Quotas**: per-tenant/user/route; ответ `429` + `Retry-After`.

6. **Роутинг/версии**: path/header/content-based, canary/weighted, гео-и device-based при нужде.

7. **Трансформации**: sanitize/normalize, REST↔gRPC transcoding, JSON↔Protobuf, CORS.

8. **Надёжность**: пер-роут timeouts, retry (идемпотентные), backoff+jitter, circuit-breaker, bulkhead.

9. **Кэш/компрессия**: `Cache-Control`, ETag/If-None-Match, встроенный response-cache (для GET/idempotent).

10. **Наблюдаемость**: `traceparent`/b3, `X-Request-ID`, бизнес-метки (tenant, plan, api\_key), логи со всеми атрибутами.

## Архитектуры развёртывания

- **Централизованный Edge-Gateway** (по одному на регион, актив-актив через Anycast/GSLB).
    

- **Domain Gateways** (по доменным областям: Payments, Identity, Catalog) - меньше blast-radius.
    

- **Micro-gateway** у каждого сервиса (KrakenD/Envoy-filter) - тонкие политики, но больше оперирования.
    

- **Cloud API Gateway/APIM** (AWS/GCP/Azure/Apigee/Kong Cloud) - быстрый старт, но учтите лимиты/цену/вендор-лок.
    

## Сравнения и границы ответственности

|Инструмент|На что нацелен|Что не должен делать|
|---|---|---|
|API Gateway|Edge-политики, контракты, безопасность, трафик-менеджмент|Бизнес-логика; агрегация сложных доменных сценариев (вынести в BFF/сервисы)|
|L7 LB / Ingress|Доставка и базовая маршрутизация трафика|“Богатые” authN/authZ, тарифные квоты, сложные политики и трансформации уровня продукта|
|Service Mesh|Межсервисные политики (mTLS, retries, telemetry, traffic policy внутри кластера)|Edge-аутентификация внешних клиентов; внешнее версионирование API и публичные контракты|
|BFF/GraphQL|Клиент-специфическая агрегация/оркестрация, оптимизация под UI (web/mobile)|Глобальные политики безопасности/лимитов для всей платформы (это зона gateway/edge)|

## Производительность и масштабирование

- **Stateless узлы**, конфиг из внешнего стора, sticky не обязателен (кроме WebSocket/HTTP2 session-affinity по необходимости).
    

- **Шкала**: фронт - Anycast или DNS-GSLB → L4 (ECMP/NLB) → L7 Gateway (кластер) → сервисы.
    

- **Тёплые JWKS-ключи** (cache & rotate), local-cache для ACL/OPA-решений, **asynchronous** метрики/логи.
    

- **Политики как код** (декларативные YAML/CRD/xDS), canary-выкат правил (1–5%), быстрый rollback.
    

## Безопасность

- **OAuth2/OIDC**: предпочитайте Authorization Code + PKCE; **JWT-валидация локально** (по JWKS, с кэшированием и `kid`), **интроспекция** только когда требуется (opaque token/ревокация).
    
- **mTLS внутрь**: корневой CA организации, короткоживущие сертификаты, SNI-пиннинг.
    
- **OWASP API Top-10**: валидация входа (schema/size/rate), защита от mass assignment, "лишних" полей, enumeration.
    
- **CORS/CSRF**: корректные `Origin`, preflight-кэш, SameSite для cookie-потоков.
    
- **Secrets**: менеджер секрессов (Vault/KMS), zero-trust: least privilege.
    

## Наблюдаемость и контроль качества

- **Метрики** (по маршрутам/тенантам): `requests_total`, `request_duration_seconds` (hist), `upstream_time`, `retries_total`, `429_total`, `5xx_total`, `auth_fail_total`.
    

- **Трассы**: прокидывать `traceparent`/`baggage`, спаны Gateway и upstream.
    

- **Логи**: JSON, включать correlation id, tenant, api\_key (хэш/ид), client ip/asn, user-agent, matched-route, decision (allow/reject).
    

- S**LO/алёрты**: p95/p99, доля 5xx, 429/401/403 аномалии, деградация JWKS/OPA/Redis.
    

## Частые анти-паттерны

- **"Бизнес-логика в шлюзе"** - шлюз превращается в монолит; выносите в BFF/сервисы.
    

- **Валидация JWT через удалённую интроспекцию на каждом запросе** - лишняя сетя, перегрев auth-сервера.
    

- Г**лобальный "мутный" rate limit** вместо пер-тенант/пер-route - несправедливо и неуправляемо.
    

- **Склейка Gateway и WAF без понимания порядка правил** - неожиданные блокировки.
    

- **Персистентные сессии в шлюзе** - ломает горизонталку; храните в внешнем сторе.
    

## Диаграммы (PlantUML, C4)

### Контур: CDN/WAF → API Gateway → микросервисы (+ Keycloak, Rate-Limiter, Observability)

![Рис. 1:&nbsp;Контур: CDN/WAF → API Gateway → микросервисы](https://www.storageskillspace.ru/schools/48528/lesson/2219348/image/699db645e74040.39998242ss7.webp)

Рис. 1: Контур: CDN/WAF → API Gateway → микросервисы

### Последовательность: запрос с JWT через шлюз (валидация → RL → роутинг)

![Рис. 2:&nbsp;Последовательность: запрос с JWT через шлюз](https://www.storageskillspace.ru/schools/48528/lesson/2219348/image/699db65d9b3029.22083606ss8.webp)

Рис. 2: Последовательность: запрос с JWT через шлюз

## Чек-лист внедрения (prod-ready)

- [ ] SLO: latency-budget per route; **p99** целевой.
    
- [ ] AuthN: JWKS-кэш, key-rotation, clock-skew, audience, token size limits.
    
- [ ] AuthZ: централизованные политики (OPA) либо per-route claims check.
    
- [ ] RL: per-tenant/user/route, headers `X-RateLimit-`, `429`.
    
- [ ] Надёжность: timeouts, idempotent-retries, circuit-breaker, retry-budget.
    
- [ ] Трафик-менеджмент: canary headers/weights, mirroring, shadowing.
    
- [ ] Observability: RED-дашборды, трассы, логи с полями.
    
- [ ] Security: mTLS внутрь, WAF интеграция, CORS, size limits, schema validation.
    
- [ ] GitOps: декларативные правила, canary-вкат, мгновенный rollback.
    
- [ ] DR: актив-актив по регионам, Anycast/DNS-фейловер, реплика конфигов.