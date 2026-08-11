# KUBERNETES

> В требованиях: «понимание контейнеризированной среды и devops». В [[CLOUDS]] был только Docker, причём почти целиком скриншотами. Здесь — текстовая база под собеседование.

Связанные: [[CLOUDS]], [[OBSERVABILITY]], [[NGINX]], [[SCALABILITY AND MAINTAINABILITY]], [[LOAD BALANCERS]]

---

## Модель: что чем управляет

```
Deployment → ReplicaSet → Pod → Container
                 ↑
              Service (стабильный DNS + балансировка)
                 ↑
              Ingress (HTTP-роутинг снаружи)
```

| Объект | Роль |
|---|---|
| **Pod** | Минимальная единица. Один или несколько контейнеров с общим network namespace и volume'ами. Эфемерен, IP меняется |
| **ReplicaSet** | Держит заданное число копий пода |
| **Deployment** | Управляет ReplicaSet'ами, даёт rolling update и rollback |
| **StatefulSet** | Для stateful: стабильные имена (`kafka-0`, `kafka-1`), персональные PVC, упорядоченный старт |
| **DaemonSet** | По одному поду на каждую ноду (агенты логов, метрик) |
| **Job / CronJob** | Разовая задача / по расписанию |
| **Service** | Стабильный виртуальный IP и DNS-имя поверх меняющихся подов |
| **Ingress** | L7-роутинг по host/path, TLS-терминация |
| **ConfigMap / Secret** | Конфигурация и чувствительные данные отдельно от образа |
| **Namespace** | Логическая изоляция + квоты |

**Service types:** `ClusterIP` (только внутри), `NodePort` (порт на каждой ноде), `LoadBalancer` (внешний балансировщик облака), `Headless` (`clusterIP: None`, DNS отдаёт IP всех подов — нужен StatefulSet'ам и клиентам вроде Kafka).

---

## Манифест приложения, который надо уметь написать

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: request-service
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }
  template:
    spec:
      terminationGracePeriodSeconds: 45
      containers:
        - name: app
          image: registry.bank.ru/request-service:1.4.2
          resources:
            requests: { cpu: "500m", memory: "1Gi" }
            limits:   { memory: "1Gi" }
          env:
            - name: JAVA_TOOL_OPTIONS
              value: "-XX:MaxRAMPercentage=75 -XX:+UseG1GC"
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef: { name: db-creds, key: password }
          startupProbe:
            httpGet: { path: /actuator/health/liveness, port: 8080 }
            failureThreshold: 30
            periodSeconds: 5
          readinessProbe:
            httpGet: { path: /actuator/health/readiness, port: 8080 }
            periodSeconds: 5
          livenessProbe:
            httpGet: { path: /actuator/health/liveness, port: 8080 }
            periodSeconds: 10
          lifecycle:
            preStop:
              exec: { command: ["sh", "-c", "sleep 10"] }
```

---

## Probes — спрашивают всегда

| Probe | Что делает при провале | Что проверять |
|---|---|---|
| **startup** | Блокирует остальные пробы, пока не пройдёт. Провалился — рестарт | Факт запуска. Спасает медленно стартующую JVM от бесконечного kill'а liveness'ом |
| **readiness** | Под **исключается из Service**, но не рестартует | Готовность обслуживать: пул БД поднят, кэш прогрет |
| **liveness** | **Рестарт контейнера** | Только «процесс жив и не завис». Максимально дёшево |

**Главная ошибка:** проверять в liveness доступность БД. БД моргнула → liveness провалился на всех подах → Kubernetes перезапускает весь сервис → при старте все ломятся в БД → каскадный отказ. БД — это readiness, и то с осторожностью.

Spring Boot даёт готовые группы: `/actuator/health/liveness` и `/actuator/health/readiness` (включить `management.endpoint.health.probes.enabled=true`).

---

## Graceful shutdown — практический вопрос

Что происходит при удалении пода, **параллельно и асинхронно**:
1. Под помечается `Terminating`, endpoint удаляется из Service.
2. Контейнеру шлётся `SIGTERM`.
3. Через `terminationGracePeriodSeconds` — `SIGKILL`.

Гонка: удаление endpoint'а из iptables/ipvs на всех нодах не мгновенно, а SIGTERM приходит сразу. Приложение может начать выключаться, всё ещё получая трафик → 502.

Лечение:
- `preStop: sleep 5-10` — даёт времени распространиться удалению endpoint'а;
- `server.shutdown=graceful` + `spring.lifecycle.timeout-per-shutdown-phase=30s` — Spring дорабатывает текущие запросы;
- `terminationGracePeriodSeconds` > preStop + graceful timeout;
- для Kafka-консьюмеров — корректно закрыть контейнер листенеров с коммитом offset'ов.

---

## Ресурсы, QoS и JVM в контейнере

**requests** — сколько гарантированно, по ним планировщик выбирает ноду. **limits** — потолок.

| QoS class | Условие | Кого убьют первым при нехватке памяти на ноде |
|---|---|---|
| `Guaranteed` | requests == limits для всех ресурсов | Последним |
| `Burstable` | requests < limits | Вторым |
| `BestEffort` | ничего не задано | Первым |

- Превышение **memory limit** → `OOMKilled` (137). Память несжимаема, договориться нельзя.
- Превышение **CPU limit** → **throttling**, не убийство. Латентность растёт, GC-паузы растут. Метрика `container_cpu_cfs_throttled_seconds_total`.
- Практика для Java: **CPU limit не ставить** (или ставить щедро), memory `requests == limits`. CPU-троттлинг у JVM бьёт по GC и JIT сильнее, чем помогает.

**JVM в контейнере.** Начиная с Java 10 `UseContainerSupport` включён и JVM читает cgroup-лимиты. Но heap — не вся память процесса: metaspace, code cache, thread stacks, direct buffers, native. Поэтому `-XX:MaxRAMPercentage=75`, а не `-Xmx` равный лимиту. Иначе стабильные `OOMKilled` без единого `OutOfMemoryError` в логах — классический вопрос «под падает, а в логах чисто, почему».

Число доступных процессоров JVM берёт из cgroup quota — это влияет на размер ForkJoinPool.commonPool и GC-потоков.

---

## Обновления и устойчивость

**RollingUpdate** — `maxSurge` (сколько сверх нормы можно поднять), `maxUnavailable` (сколько недоступно). `maxUnavailable: 0` = без просадки ёмкости. Откат: `kubectl rollout undo deployment/request-service`.

**PodDisruptionBudget** — `minAvailable: 2` не даёт drain'у ноды выкосить кворум при обслуживании кластера.

**HPA** — автомасштабирование по CPU/памяти/кастомной метрике (через Prometheus Adapter — например, по consumer lag Kafka). Не работает без выставленных `requests`.

**Affinity / anti-affinity** — растащить реплики по разным нодам и зонам, чтобы падение ноды не унесло весь сервис.

**Init container** — отрабатывает до основного: миграции Liquibase, ожидание доступности БД.
**Sidecar** — рядом с основным: envoy, агент логов.

---

## Диагностика: что говорить при разборе инцидента

```bash
kubectl get pods -n prod -o wide
kubectl describe pod request-service-7d9f-x2k1     # Events внизу — главное
kubectl logs request-service-7d9f-x2k1 --previous  # логи упавшего контейнера
kubectl top pods -n prod
kubectl rollout history deployment/request-service
kubectl exec -it request-service-7d9f-x2k1 -- jcmd 1 Thread.print
kubectl port-forward pod/request-service-7d9f-x2k1 8080:8080
```

| Статус | Причина |
|---|---|
| `CrashLoopBackOff` | Контейнер падает при старте. Смотреть `logs --previous` |
| `ImagePullBackOff` | Нет образа/прав в registry |
| `Pending` | Планировщику некуда поставить: не хватает ресурсов, не подходят taints/affinity, PVC не привязан |
| `OOMKilled` (exit 137) | Превышен memory limit |
| `Evicted` | Нода под давлением ресурсов выселила под |
| `Init:0/1` | Init-контейнер не отработал |

Связка с [[OBSERVABILITY]]: логи в stdout в JSON (никаких файлов внутри контейнера), метрики через `/actuator/prometheus`, трассировка через OTel-агент, у каждого лога `trace_id`.

---

## DevOps-контур целиком

**CI:** сборка → unit + интеграционные тесты (Testcontainers, см. [[TESTING]]) → статический анализ (SonarQube) → проверка зависимостей на CVE → сборка образа → скан образа (Trivy) → push в registry.

**CD:** обновление тега в манифесте/Helm-values → выкат на dev → smoke-тесты → stage → prod с ручным гейтом.

**Helm** — шаблонизация манифестов: `Chart.yaml`, `values.yaml` на окружение, `helm upgrade --install`, `helm rollback`. **Kustomize** — альтернатива через оверлеи без шаблонов.

**GitOps (ArgoCD/Flux)** — состояние кластера описано в git, оператор непрерывно приводит кластер к нему. Плюсы: аудит через git-историю (для банка существенно), откат = revert коммита, никто не правит прод руками.

**Стратегии выката:** rolling (дефолт), blue-green (два полных окружения, переключение трафика), canary (5% трафика на новую версию, смотрим метрики ошибок), feature flags (выкат кода отдельно от включения функциональности).

**Секреты:** `Secret` в Kubernetes — это base64, не шифрование. Реально — HashiCorp Vault / внешний секрет-менеджер + `ExternalSecrets`, шифрование etcd at rest, запрет секретов в git (в GitOps — sealed secrets / SOPS).

**Безопасность образа:** не root (`runAsNonRoot`, `runAsUser`), `readOnlyRootFilesystem`, distroless/alpine базовый образ, многоступенчатая сборка, никаких секретов в слоях, фиксация версий по digest.

---

## Вопросы с собеседования

**1. Чем Deployment отличается от StatefulSet?**
Deployment — взаимозаменяемые поды со случайными именами и общим (или отсутствующим) хранилищем. StatefulSet — стабильная идентичность: имя `app-0`, персональный PVC, который переживает пересоздание, упорядоченные старт и остановка, headless service для прямой адресации. Нужен для БД, Kafka, ZooKeeper.

**2. Почему под OOMKilled, а в логах нет OutOfMemoryError?**
Убила не JVM, а ядро по cgroup-лимиту. Значит суммарное потребление процесса (heap + metaspace + direct + thread stacks + native) вышло за memory limit, при том что heap мог быть в порядке. Настраивать через `MaxRAMPercentage`, смотреть `NativeMemoryTracking`, проверять direct buffers и число потоков.

**3. Как выкатить без единой ошибки у пользователя?**
`maxUnavailable: 0`, корректный readiness (под входит в Service только когда реально готов), preStop-задержка + graceful shutdown, PDB, идемпотентные API и обратно совместимая схема БД (expand-contract: сначала добавили колонку, выкатили код, потом удалили старую — никогда одним релизом).

**4. Три реплики, но нагрузка идёт неравномерно. Почему?**
Service балансирует на уровне соединений, а не запросов. HTTP/1.1 keep-alive и особенно HTTP/2 держат долгое соединение к одному поду. Лечение: балансировка на L7 (Ingress/service mesh с per-request LB), ограничение времени жизни соединений, для gRPC — обязательный L7-балансировщик.

**5. Зачем readiness, если есть liveness?**
Они про разное: liveness — «перезапустить», readiness — «не давать трафик». Под может быть жив, но временно не готов (прогрев кэша, переподключение к БД) — рестартовать его вредно, достаточно вывести из балансировки.

**6. Что такое namespace и как ограничить команду?**
Логическая изоляция объектов + `ResourceQuota` (потолок cpu/memory/числа объектов) + `LimitRange` (дефолты и границы на контейнер) + RBAC (Role/RoleBinding на namespace, ClusterRole на кластер) + NetworkPolicy (кто с кем может общаться по сети — по умолчанию в Kubernetes разрешено всё).

---

## Как отвечать, если опыта администрирования нет

Разработчику и не нужно уметь поднимать кластер. Нужно уметь: написать корректный Deployment со своим приложением, объяснить probes и graceful shutdown, настроить ресурсы под JVM, прочитать `describe`/`logs` при инциденте, понимать, почему стейт нельзя держать в поде. Формулировка: «Кластер у нас держит платформенная команда, я отвечаю за манифесты и Helm-чарт своего сервиса, разбираю проблемы своих подов: OOM, троттлинг, ребалансы при выкате».
