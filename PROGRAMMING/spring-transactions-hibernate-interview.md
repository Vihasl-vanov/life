# Spring: транзакции и Hibernate. Полная подготовка к собеседованию (Senior)

> Транзакции + Hibernate/JPA. Теория, механика изнутри, вопросы с кодом и ответами, задачи-головоломки.

## Содержание

**Транзакции**

- [Часть 2. Архитектура транзакционного модуля (справка)](#часть-2-архитектура-транзакционного-модуля-spring)
- [Блок A. Механика: как это работает изнутри](#блок-a-механика-как-transactional-работает-изнутри)
- [Блок B. Propagation](#блок-b-propagation--распространение-транзакций)
- [Блок C. Isolation и блокировки](#блок-c-isolation-и-блокировки)
- [Блок D. Правила отката](#блок-d-правила-отката-rollback)
- [Блок E. Транзакции + JPA/Hibernate](#блок-e-транзакции--jpahibernate)
- [Блок F. Границы транзакции и производительность](#блок-f-границы-транзакции-и-производительность)
- [Блок G. Программные транзакции и синхронизации](#блок-g-программные-транзакции-и-синхронизации)
- [Блок H. Многопоточность, события, распределённые транзакции](#блок-h-многопоточность-события-распределённые-транзакции)
- [Блок I. Тестирование транзакций](#блок-i-тестирование-транзакций)
- [Блок J. Реактивные транзакции](#блок-j-реактивные-транзакции)
- [Часть 3. Задачи «что произойдёт» (транзакции)](#часть-3-задачи-что-произойдёт)
- [Часть 4. Шпаргалка по транзакциям](#часть-4-шпаргалка-перед-собеседованием)

**Hibernate / JPA**

- [Часть 5. Hibernate: план и архитектура](#часть-5-hibernate--что-надо-знать-для-интервью)
- [Блок K. Архитектура и базовые понятия](#блок-k-архитектура-и-базовые-понятия)
- [Блок L. Жизненный цикл сущности и persistence context](#блок-l-жизненный-цикл-сущности-и-persistence-context)
- [Блок M. Маппинг: идентификаторы, ассоциации, каскады, коллекции](#блок-m-маппинг-идентификаторы-ассоциации-каскады-коллекции)
- [Блок N. Наследование и составные типы](#блок-n-наследование-и-составные-типы)
- [Блок O. Загрузка данных: lazy/eager, N+1, EntityGraph](#блок-o-загрузка-данных-lazyeager-n1-entitygraph)
- [Блок P. Запросы: JPQL/HQL, Criteria, native, проекции](#блок-p-запросы-jpqlhql-criteria-native-проекции)
- [Блок Q. Кэши: первый, второй уровень, кэш запросов](#блок-q-кэши-первый-второй-уровень-кэш-запросов)
- [Блок R. Производительность и пакетная обработка](#блок-r-производительность-и-пакетная-обработка)
- [Блок S. Полезные возможности, о которых спрашивают](#блок-s-полезные-возможности-о-которых-спрашивают)
- [Блок T. Исключения и диагностика](#блок-t-исключения-и-диагностика)
- [Часть 6. Задачи «что произойдёт» (Hibernate)](#часть-6-задачи-что-произойдёт-hibernate)
- [Часть 7. Шпаргалка по Hibernate](#часть-7-шпаргалка-по-hibernate)

---

**Три фразы, которые лучше не произносить:**

- «`@Transactional` открывает соединение» — соединение может быть взято лениво, и это отдельная тема.
- «`readOnly` запрещает запись» — не запрещает.
- «`REQUIRES_NEW` — это вложенная транзакция» — вложенная это `NESTED`, а `REQUIRES_NEW` — независимая.

---

## Часть 2. Архитектура транзакционного модуля Spring

Это справочная схема — держите её в голове, ответы на 80% вопросов выводятся отсюда.

```
@EnableTransactionManagement (в Boot включается автоматически)
   └─ TransactionManagementConfigurationSelector
        └─ ProxyTransactionManagementConfiguration регистрирует 3 бина:
             ├─ TransactionAttributeSource  (AnnotationTransactionAttributeSource)
             ├─ TransactionInterceptor      (сам advice)
             └─ BeanFactoryTransactionAttributeSourceAdvisor (pointcut + advice)
                  └─ InfrastructureAdvisorAutoProxyCreator оборачивает бины в прокси

Вызов метода
   └─ TransactionInterceptor.invoke()
        └─ TransactionAspectSupport.invokeWithinTransaction()
             ├─ tas.getTransactionAttribute(method, targetClass) → propagation/isolation/...
             ├─ determineTransactionManager()            → какой TM использовать
             ├─ createTransactionIfNecessary()
             │     └─ PlatformTransactionManager.getTransaction(definition)
             │          └─ AbstractPlatformTransactionManager
             │               ├─ doGetTransaction()        — объект транзакции
             │               ├─ isExistingTransaction()   — уже есть?
             │               ├─ handleExistingTransaction() — логика propagation
             │               ├─ doBegin()                 — начать физическую транзакцию
             │               └─ doSuspend()/doResume()    — приостановка
             ├─ invocation.proceedWithInvocation()        — ваш метод
             ├─ completeTransactionAfterThrowing()        — rollback / commit по правилам
             └─ commitTransactionAfterReturning()

Хранилище состояния (ThreadLocal!)
   TransactionSynchronizationManager
     ├─ resources: Map<DataSource|EntityManagerFactory, ConnectionHolder|EntityManagerHolder>
     ├─ synchronizations: Set<TransactionSynchronization>
     ├─ currentTransactionName / ReadOnly / IsolationLevel
     └─ actualTransactionActive: boolean

Получение ресурса внутри кода
   DataSourceUtils.getConnection(ds)                    → из TSM, иначе новое соединение
   EntityManagerFactoryUtils.doGetTransactionalEntityManager(emf) → из TSM
```

**Реализации `PlatformTransactionManager`:**

| Менеджер | Ресурс в TSM | Когда |
|---|---|---|
| `DataSourceTransactionManager` | `ConnectionHolder` по `DataSource` | JDBC, `JdbcTemplate`, MyBatis |
| `JpaTransactionManager` | `EntityManagerHolder` по `EntityManagerFactory` (+ `ConnectionHolder`) | JPA/Hibernate |
| `JtaTransactionManager` | делегирует в JTA | XA, несколько ресурсов |
| `R2dbcTransactionManager` | реактивный (`ReactiveTransactionManager`) | R2DBC/WebFlux |
| `KafkaTransactionManager`, `RabbitTransactionManager` | свои | брокеры |

**Ключевые интерфейсы:**

```java
public interface PlatformTransactionManager extends TransactionManager {
    TransactionStatus getTransaction(TransactionDefinition def) throws TransactionException;
    void commit(TransactionStatus status) throws TransactionException;
    void rollback(TransactionStatus status) throws TransactionException;
}

public interface TransactionDefinition {   // ЧТО хотим: propagation, isolation, timeout, readOnly, name
    int getPropagationBehavior();
    int getIsolationLevel();
    int getTimeout();
    boolean isReadOnly();
    String getName();
}

public interface TransactionStatus extends TransactionExecution, SavepointManager, Flushable {
    boolean isNewTransaction();   // это физически новая транзакция или мы «участвуем»?
    boolean hasSavepoint();
    void setRollbackOnly();
    boolean isRollbackOnly();
    boolean isCompleted();
}
```

**Полный список атрибутов `@Transactional`:**

```java
@Transactional(
    transactionManager = "orderTxManager",       // или value = "..."
    propagation = Propagation.REQUIRED,
    isolation   = Isolation.READ_COMMITTED,
    timeout     = 10,                            // секунды; есть timeoutString = "${app.tx.timeout}"
    readOnly    = false,
    rollbackFor          = { BusinessException.class },
    rollbackForClassName = { "com.example.LegacyException" },
    noRollbackFor          = { NotFoundException.class },
    noRollbackForClassName = { "..." },
    label = { "long-running" }                   // Spring 5.3+, для кастомной логики в TM
)
```

---

# Блок A. Механика: как `@Transactional` работает изнутри

### A1. Что происходит при вызове метода с `@Transactional`? (расскажите по шагам)

**Ответ.**

1. `@EnableTransactionManagement` (Boot включает через `TransactionAutoConfiguration`) регистрирует `TransactionInterceptor`, `TransactionAttributeSource` и advisor, а `InfrastructureAdvisorAutoProxyCreator` оборачивает подходящие бины в прокси (в Boot — CGLIB, т.к. `proxyTargetClass=true`).
2. Внешний вызов попадает в прокси → `TransactionInterceptor.invoke()`.
3. `AnnotationTransactionAttributeSource` возвращает `TransactionAttribute` для этого метода (с кэшированием). Если атрибута нет — метод вызывается без транзакции.
4. `determineTransactionManager()` выбирает менеджер: по `transactionManager`/квалификатору из аннотации, иначе — единственный/`@Primary` бин `PlatformTransactionManager`.
5. `tm.getTransaction(attr)`:
   - `doGetTransaction()` создаёт объект транзакции и смотрит в `TransactionSynchronizationManager`, нет ли уже привязанного ресурса;
   - если транзакция уже есть → `handleExistingTransaction()` реализует семантику propagation (участвовать / приостановить / бросить исключение / сделать savepoint);
   - если нет → по propagation либо `doBegin()` (взять `Connection`, `setAutoCommit(false)`, применить isolation/readOnly/timeout, привязать holder к потоку), либо работа вовсе без транзакции.
6. Вызывается целевой метод. Весь код в этом потоке, который берёт ресурс через `DataSourceUtils`/`EntityManagerFactoryUtils` (а `JdbcTemplate`, Spring Data и Hibernate делают именно так), получает **тот же** `Connection`/`EntityManager`.
7. Нормальный выход → `commitTransactionAfterReturning()`. Исключение → `completeTransactionAfterThrowing()`, где `txAttr.rollbackOn(ex)` решает: rollback или всё-таки commit.
8. `cleanupTransactionInfo()` — ресурс отвязывается от потока, соединение возвращается в пул, приостановленная транзакция восстанавливается.

**Вывод, который надо озвучить:** транзакция — это (а) прокси, (б) ThreadLocal. Всё, что ломается с `@Transactional`, ломается по одной из этих двух причин.

---

### A2. Почему `@Transactional` не работает на private-методе и при вызове изнутри?

**Ответ.** В proxy-режиме перехват возможен только на границе прокси:

- `private`/`static` методы не переопределяются CGLIB-подклассом и не входят в интерфейс → advice не применяется;
- `final` методы/классы CGLIB переопределить не может — Spring 6 логирует предупреждение, транзакции не будет;
- `protected`/package-private технически перехватываются CGLIB, но `AnnotationTransactionAttributeSource` их **игнорирует** (см. `computeTransactionAttribute` — «allow non-public methods» отключено);
- вызов `this.method()` идёт в target, минуя прокси, — **self-invocation**.

```java
@Service
public class OrderService {

    public void process(Long id) {
        save(id);                  // ❌ прокси не задействован — транзакции НЕТ
    }

    @Transactional
    public void save(Long id) { repo.save(...); }
}
```

**Решения (по убыванию правильности):**

```java
// 1. Разнести по бинам — правильная архитектура
@Service class OrderProcessor { private final OrderSaver saver; }

// 2. Самоинъекция через прокси
@Service
public class OrderService {
    @Autowired @Lazy private OrderService self;
    public void process(Long id) { self.save(id); }
}

// 3. AopContext (нужен @EnableAspectJAutoProxy(exposeProxy = true))
((OrderService) AopContext.currentProxy()).save(id);

// 4. Программная транзакция — явно и без магии
private final TransactionTemplate tx;
public void process(Long id) { tx.executeWithoutResult(s -> repo.save(...)); }
```

Если нужен перехват внутренних вызовов «по-настоящему» — `@EnableTransactionManagement(mode = AdviceMode.ASPECTJ)` + load-time weaving. Но это редкий и тяжёлый выбор.

---

### A3. Где именно живёт транзакционный контекст?

**Ответ.** В `TransactionSynchronizationManager` — наборе `ThreadLocal`:

```java
// упрощённо
private static final ThreadLocal<Map<Object, Object>> resources;           // DataSource → ConnectionHolder
private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations;
private static final ThreadLocal<String>  currentTransactionName;
private static final ThreadLocal<Boolean> currentTransactionReadOnly;
private static final ThreadLocal<Integer> currentTransactionIsolationLevel;
private static final ThreadLocal<Boolean> actualTransactionActive;
```

Практическое применение — диагностика в рантайме:

```java
boolean inTx  = TransactionSynchronizationManager.isActualTransactionActive();
boolean ro    = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
String  name  = TransactionSynchronizationManager.getCurrentTransactionName();
```

И три следствия, которые надо назвать: другой поток = другая транзакция; `EntityManager` не потокобезопасен; передача работы в пул потоков (`@Async`, `CompletableFuture`, параллельный стрим) выводит её из транзакции.

---

### A4. Как выбирается `PlatformTransactionManager`, если их несколько?

**Ответ.** Порядок: значение `transactionManager`/`value` в аннотации → кастомный квалификатор → `@Primary` → единственный бин. Результат кэшируется в `TransactionAspectSupport`.

```java
@Configuration
class TxConfig {
    @Bean @Primary
    PlatformTransactionManager orderTxManager(@Qualifier("orderDs") DataSource ds) {
        return new DataSourceTransactionManager(ds);
    }
    @Bean
    PlatformTransactionManager auditTxManager(@Qualifier("auditDs") DataSource ds) {
        return new DataSourceTransactionManager(ds);
    }
}

@Service
class AuditService {
    @Transactional("auditTxManager")            // явно указываем менеджер
    public void log(String msg) { }
}

// Красивее — свой квалификатор
@Qualifier("auditTxManager")
@Target({ METHOD, TYPE }) @Retention(RUNTIME)
@interface AuditTx { }

@AuditTx @Transactional
public void log(String msg) { }
```

**Важная оговорка:** два менеджера ≠ одна транзакция на две БД. Это две независимые транзакции, и атомарности между ними нет. `ChainedTransactionManager` (deprecated с 5.3) лишь коммитит их по очереди — «best effort 1PC», сбой между коммитами оставляет систему в несогласованном состоянии. Настоящая атомарность на два ресурса — только JTA/XA (Atomikos, Narayana), а на практике чаще — outbox/saga.

---

### A5. `@Transactional` на интерфейсе, классе или методе — где правильно?

**Ответ.** На **конкретном классе** или его методе. Это прямая рекомендация документации Spring: при CGLIB-проксировании аннотация на интерфейсе может не быть обнаружена, потому что прокси наследует класс, а не реализует интерфейс.

Порядок разрешения (побеждает самое конкретное): метод класса → класс → метод интерфейса → интерфейс.

```java
@Service
@Transactional(readOnly = true)              // дефолт для всего класса
public class OrderService {

    public Order find(Long id) { }           // readOnly = true

    @Transactional                           // переопределили: read-write
    public void save(Order o) { }
}
```

Этот паттерн (`readOnly = true` на классе, `@Transactional` точечно на пишущих методах) — хороший ответ на вопрос «как вы организуете транзакции в сервисе».

---

### A6. В каком порядке транзакционный advice встаёт относительно других аспектов?

**Ответ.** `TransactionInterceptor` имеет order `Ordered.LOWEST_PRECEDENCE` — то есть он **самый внутренний**. Ваш аспект с `@Order(1)` отработает **снаружи** транзакции: на входе — до её открытия, на выходе — после коммита.

```java
@Aspect @Component
@Order(Ordered.HIGHEST_PRECEDENCE)     // снаружи транзакции
class OuterAspect {
    @Around("@annotation(org.springframework.transaction.annotation.Transactional)")
    Object around(ProceedingJoinPoint pjp) throws Throwable {
        // здесь транзакции ЕЩЁ нет
        Object r = pjp.proceed();
        // здесь транзакция УЖЕ закоммичена
        return r;
    }
}
```

Практический смысл: аспект-ретрай при `OptimisticLockException` обязан быть **снаружи** транзакции — иначе повтор произойдёт внутри уже отравленной транзакции. Именно поэтому `@Retryable` ставят на внешний метод, а `@Transactional` — на внутренний, в другом бине.

Порядок транзакционного advice меняется через `@EnableTransactionManagement(order = ...)`.

---

# Блок B. Propagation — распространение транзакций

### B1. Перечислите все уровни propagation

**Ответ.**

| Propagation | Транзакция уже есть | Транзакции нет |
|---|---|---|
| `REQUIRED` *(default)* | присоединиться (логическая вложенность) | создать новую |
| `REQUIRES_NEW` | приостановить текущую, создать независимую | создать новую |
| `SUPPORTS` | присоединиться | выполнить без транзакции |
| `NOT_SUPPORTED` | приостановить, выполнить без транзакции | выполнить без транзакции |
| `MANDATORY` | присоединиться | `IllegalTransactionStateException` |
| `NEVER` | `IllegalTransactionStateException` | выполнить без транзакции |
| `NESTED` | вложенная через SAVEPOINT | как `REQUIRED` |

Мнемоника для устного ответа: **два «присоединяются»** (`REQUIRED`, `SUPPORTS`), **два «изолируются»** (`REQUIRES_NEW`, `NOT_SUPPORTED`), **два «проверяют»** (`MANDATORY`, `NEVER`), **один «частичный откат»** (`NESTED`).

---

### B2. Физическая и логическая транзакция — в чём разница?

**Ответ.** Это ключевое понятие, из которого выводится всё остальное поведение `REQUIRED`.

- **Физическая транзакция** — реальная транзакция БД: одно соединение, один `BEGIN`, один `COMMIT`.
- **Логическая транзакция** — область действия одного `@Transactional`-метода.

При `REQUIRED` вложенный метод создаёт **новую логическую** транзакцию внутри **той же физической**. В логах это видно как `Participating in existing transaction`. `TransactionStatus.isNewTransaction()` для внутреннего вызова вернёт `false`, и коммит на его границе не выполняется — коммитит только внешний, «настоящий» владелец транзакции.

```java
@Transactional          // физическая транзакция начинается ЗДЕСЬ
public void outer() {
    inner();            // логическая транзакция; isNewTransaction() == false
}                       // COMMIT происходит ЗДЕСЬ

@Transactional          // REQUIRED
public void inner() { } // никакого commit на выходе
```

Отсюда: `REQUIRES_NEW` создаёт новую **физическую** транзакцию (новое соединение), а `NESTED` — точку сохранения внутри той же физической.

---

### B3. Что такое `rollbackOnly` и откуда берётся `UnexpectedRollbackException`?

**Ответ.** Самый частый «убийственный» вопрос по транзакциям.

Когда внутренняя `REQUIRED`-транзакция завершается исключением, `AbstractPlatformTransactionManager` (свойство `globalRollbackOnParticipationFailure = true` по умолчанию) помечает **всю физическую транзакцию** как rollback-only. Внешний метод может исключение поймать — но транзакция уже обречена. На его коммите Spring обнаруживает флаг, выполняет rollback и бросает `UnexpectedRollbackException`.

```java
@Transactional
public void outer() {
    try {
        inner();                       // бросит RuntimeException
    } catch (RuntimeException ignored) {
        log.warn("не страшно, продолжаем");   // ❌ иллюзия
    }
    repo.save(somethingElse);          // это тоже НЕ сохранится
}                                      // → UnexpectedRollbackException

@Transactional                          // REQUIRED
public void inner() { throw new IllegalStateException(); }
```

Сообщение в логе: `Transaction silently rolled back because it has been marked as rollback-only`.

**Как правильно чинить:**

1. Если внутренняя операция действительно опциональна — `Propagation.REQUIRES_NEW` (её откат не заденет внешнюю).
2. Или `@Transactional(noRollbackFor = ThatException.class)` на внутреннем методе.
3. Или вообще убрать `@Transactional` с внутреннего метода — пусть он работает в транзакции вызывающего.
4. Диагностика в моменте: `TransactionAspectSupport.currentTransactionStatus().isRollbackOnly()`.

Есть ещё флаг `failEarlyOnGlobalRollbackOnly` — если выставить `true` на менеджере, исключение прилетит сразу в точке пометки, а не на коммите. Удобно при отладке.

---

### B4. `REQUIRES_NEW`: как работает и какие с ним проблемы?

**Ответ.** Текущая транзакция **приостанавливается** (`doSuspend()` — ресурс отвязывается от потока и сохраняется в `SuspendedResourcesHolder`), берётся **новое соединение из пула**, выполняется независимая транзакция, затем исходная восстанавливается (`doResume()`).

```java
@Service
class OrderService {
    @Transactional
    public void place(Order o) {
        repo.save(o);
        audit.log("placed " + o.getId());   // REQUIRES_NEW — переживёт откат
        throw new PaymentException();        // откат place(), но запись аудита осталась
    }
}

@Service
class AuditService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void log(String msg) { auditRepo.save(new Audit(msg)); }
}
```

**Три проблемы, которые надо назвать:**

1. **Расход соединений.** На время вложенной транзакции удерживаются два соединения. При пуле в 10 и 10 параллельных запросах, каждый из которых берёт второе соединение, — классический deadlock по пулу: все держат по одному и ждут второго. Правило: `pool size > max глубина вложенности × параллелизм`.
2. **Самоблокировка.** Новая транзакция не видит незакоммиченных изменений внешней. Если она попытается обновить строку, уже заблокированную внешней транзакцией, — ждать будет до таймаута навсегда, потому что внешняя ждёт её саму.
3. **Нет атомарности.** Внутренняя коммитится независимо; если внешняя откатится, данные разъедутся. Это фича — но она должна быть осознанной.

Типичные легальные применения: аудит, логирование попыток, счётчики, сохранение сообщения об ошибке.

---

### B5. `NESTED` — чем отличается от `REQUIRES_NEW`?

**Ответ.**

| | `NESTED` | `REQUIRES_NEW` |
|---|---|---|
| Механизм | SAVEPOINT внутри той же физической транзакции | отдельная физическая транзакция |
| Соединение | одно (то же) | новое из пула |
| Коммит | только вместе с внешней | независимый, сразу |
| Откат внешней | откатывает и вложенную | вложенная уже закоммичена, остаётся |
| Откат вложенной | rollback до savepoint, внешняя жива | внешняя не затронута |
| Видимость данных | видит незакоммиченные данные внешней | не видит |

```java
@Transactional
public void importAll(List<Row> rows) {
    for (Row r : rows) {
        try { importOne(r); }               // NESTED
        catch (Exception e) { errors.add(r); }   // откат ТОЛЬКО этой строки
    }
}   // остальные строки коммитятся вместе

@Transactional(propagation = Propagation.NESTED)
public void importOne(Row r) { repo.save(map(r)); }
```

**Ограничения:**

- работает только там, где менеджер поддерживает savepoints и `nestedTransactionAllowed = true`. `DataSourceTransactionManager` включает это по умолчанию;
- `JpaTransactionManager` по умолчанию бросает `NestedTransactionNotSupportedException` — нужно явно `setNestedTransactionAllowed(true)`, и даже тогда savepoint не откатывает состояние persistence context: сущности в кэше первого уровня останутся изменёнными. Практически всегда требуется `EntityManager.clear()` после отката к savepoint;
- не поддерживается в JTA.

Ответ на «когда применяю»: пакетная обработка, где одна плохая запись не должна валить весь батч.

---

### B6. `SUPPORTS`, `NOT_SUPPORTED`, `MANDATORY`, `NEVER` — реальные сценарии

**Ответ.**

```java
// SUPPORTS — метод-читалка, которая работает и внутри, и вне транзакции
@Transactional(propagation = Propagation.SUPPORTS, readOnly = true)
public Report read() { }
```
Нюанс `SUPPORTS`: если транзакции нет, Spring всё равно создаёт «пустую» транзакцию (`isActualTransactionActive() == false`, но синхронизации активны, т.к. `transactionSynchronization = SYNCHRONIZATION_ALWAYS` по умолчанию). Для Hibernate это означает, что каждый запрос идёт в своём auto-commit — при нескольких запросах подряд можно получить несогласованное чтение.

```java
// NOT_SUPPORTED — долгая операция, которую нельзя держать в транзакции
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public Report heavyReport() { }        // не удерживает лок и не жрёт транзакционное соединение
```

```java
// MANDATORY — защита архитектуры: метод обязан быть частью чужой транзакции
@Transactional(propagation = Propagation.MANDATORY)
public void applyDiscount(Order o) { }   // без транзакции → IllegalTransactionStateException
```
`MANDATORY` — отличный ответ на вопрос «как защититься от того, что кто-то вызовет метод вне транзакции». Ставится на низкоуровневые операции, которые бессмысленны в одиночку.

```java
// NEVER — гарантия отсутствия транзакции (например, вызов внешнего API)
@Transactional(propagation = Propagation.NEVER)
public void callExternalApi() { }
```

---

### B7. `REQUIRES_NEW` и self-invocation: почему «не сработало»?

**Ответ.** Классический баг: разработчик ставит `REQUIRES_NEW` на метод в том же классе и удивляется, что вложенной транзакции нет.

```java
@Service
class OrderService {
    @Transactional
    public void place(Order o) {
        repo.save(o);
        this.audit("ok");        // ❌ прокси минован → выполняется в ТОЙ ЖЕ транзакции
        throw new RuntimeException();   // аудит откатится вместе со всем
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void audit(String msg) { auditRepo.save(new Audit(msg)); }
}
```

Проверяется просто: `TransactionSynchronizationManager.getCurrentTransactionName()` внутри `audit()` покажет имя внешнего метода — значит, это та же транзакция.

---

### B8. Меняется ли isolation при присоединении к существующей транзакции?

**Ответ.** Нет. Isolation и timeout применяются только при **старте физической** транзакции. Если метод присоединяется к существующей (`REQUIRED`), его `isolation` игнорируется.

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public void outer() { inner(); }

@Transactional(isolation = Isolation.SERIALIZABLE)   // ⚠ БУДЕТ ПРОИГНОРИРОВАН
public void inner() { }
```

Чтобы Spring не молчал об этом, у `AbstractPlatformTransactionManager` есть флаг `validateExistingTransaction` (по умолчанию `false`). Включите — и несовпадение isolation или read-only при участии в существующей транзакции бросит `IllegalTransactionStateException`. Хороший ответ на вопрос «как поймать такие ошибки»: включить валидацию в dev-профиле.

То же касается `readOnly`: `readOnly = true` внутри read-write транзакции не сделает её read-only.

---

# Блок C. Isolation и блокировки

### C1. Уровни изоляции и аномалии

**Ответ.**

| Isolation | Dirty read | Non-repeatable read | Phantom read |
|---|---|---|---|
| `READ_UNCOMMITTED` | + | + | + |
| `READ_COMMITTED` | − | + | + |
| `REPEATABLE_READ` | − | − | +* |
| `SERIALIZABLE` | − | − | − |

- **Dirty read** — прочитали чужие незакоммиченные изменения.
- **Non-repeatable read** — второе чтение той же строки дало другое значение.
- **Phantom read** — повторный запрос по условию вернул новые строки.
- **Lost update** — две транзакции прочитали, обе записали, одно изменение потеряно. В стандарте ANSI его нет, но на собеседовании про него спрашивают: на `READ_COMMITTED` он возможен, и защита от него — не изоляция, а `@Version` (оптимистичная блокировка) или `SELECT ... FOR UPDATE`.

`Isolation.DEFAULT` = уровень БД: PostgreSQL, Oracle, SQL Server — `READ_COMMITTED`; MySQL/InnoDB — `REPEATABLE_READ`.

\* Практические нюансы, которые ценятся:

- **PostgreSQL** реализует `REPEATABLE_READ` как snapshot isolation — фантомов нет, но возможна ошибка `could not serialize access due to concurrent update`, которую надо обрабатывать ретраем. `SERIALIZABLE` реализован как SSI и тоже падает с ошибками сериализации, а не блокирует.
- **MySQL/InnoDB** на `REPEATABLE_READ` предотвращает фантомы для обычных `SELECT` за счёт консистентного снапшота и gap-локов для блокирующих чтений.
- `READ_UNCOMMITTED` в PostgreSQL не поддерживается — молча работает как `READ_COMMITTED`.

Практический ответ на «что выбираете»: `READ_COMMITTED` + `@Version`. Повышение изоляции — последнее средство: оно даёт блокировки/ретраи и хуже масштабируется.

---

### C2. Оптимистичная блокировка

**Ответ.** `@Version`: при UPDATE версия попадает в `WHERE`, при 0 обновлённых строк Hibernate бросает `OptimisticLockException`, Spring транслирует в `ObjectOptimisticLockingFailureException`.

```java
@Entity
class Account {
    @Id Long id;
    @Version Long version;        // UPDATE account SET balance=?, version=? WHERE id=? AND version=?
    BigDecimal balance;
}
```

Обработка — ретрай **снаружи** транзакции:

```java
@Service
class TransferFacade {
    @Retryable(retryFor = ObjectOptimisticLockingFailureException.class,
               maxAttempts = 3, backoff = @Backoff(delay = 50, multiplier = 2))
    public void transfer(Long from, Long to, BigDecimal sum) {
        service.doTransfer(from, to, sum);   // ← другой бин, там @Transactional
    }
}
```

**Почему ретрай не может быть внутри транзакции:** исключение оптимистичной блокировки возникает на flush/commit, транзакция уже помечена rollback-only, persistence context невалиден. Повторять нужно всю транзакцию целиком.

Типы: `@Version` на `long`/`Integer`/`Timestamp`. `OPTIMISTIC_FORCE_INCREMENT` инкрементирует версию родителя, даже если менялся только его child — способ защитить агрегат целиком.

---

### C3. Пессимистичная блокировка

**Ответ.**

```java
public interface AccountRepository extends JpaRepository<Account, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)                 // SELECT ... FOR UPDATE
    @QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000"))
    @Query("select a from Account a where a.id = :id")
    Optional<Account> findForUpdate(@Param("id") Long id);
}
```

| Режим | SQL | Смысл |
|---|---|---|
| `PESSIMISTIC_READ` | `FOR SHARE` / `LOCK IN SHARE MODE` | другие читают, но не пишут |
| `PESSIMISTIC_WRITE` | `FOR UPDATE` | эксклюзивный доступ |
| `PESSIMISTIC_FORCE_INCREMENT` | `FOR UPDATE` + version++ | лок + инкремент версии |

Обязательно упомянуть:

- лок держится **до конца транзакции** — значит, транзакция должна быть короткой;
- без `lock.timeout` можно висеть неограниченно; в PostgreSQL альтернатива — `FOR UPDATE NOWAIT` / `SKIP LOCKED` (последнее — правильный способ разбирать очередь задач из таблицы несколькими воркерами);
- при `PESSIMISTIC_WRITE` в JPA действует ещё и правило порядка блокировок — блокируйте строки в стабильном порядке (например, по возрастанию id), иначе получите deadlock;
- deadlock в БД разрешается сама БД: одна транзакция получает `DeadlockLoserDataAccessException`, её надо ретраить.

**Как выбирать:** оптимистичная — при низкой конкуренции (карточки, формы, справочники); пессимистичная — при высокой конкуренции за конкретную строку (баланс, остатки на складе, разбор очереди).

---

### C4. Что делает `timeout` и почему он иногда «не работает»?

**Ответ.** `@Transactional(timeout = 5)` устанавливает дедлайн на `ResourceHolder`. Spring применяет его, вызывая `Statement.setQueryTimeout()` для каждого запроса, идущего через Spring-инфраструктуру (`DataSourceUtils.applyTimeout`), и проверяет дедлайн при получении ресурса.

Ограничения:

- это **не** «убить транзакцию через 5 секунд». Это лимит на каждый отдельный запрос + проверка на границах;
- запросы, идущие мимо Spring (нативный `Connection`, внешние клиенты), timeout не увидят;
- «зависание» не в БД, а в коде (внешний HTTP-вызов внутри транзакции) timeout не прервёт;
- надёжнее задавать таймауты на уровне БД/пула: `lock_timeout` и `statement_timeout` в PostgreSQL, `idle-in-transaction-session-timeout`, `spring.datasource.hikari.max-lifetime`.

`timeoutString = "${app.tx.timeout:10}"` — можно вынести в конфиг.

---

# Блок D. Правила отката (rollback)

### D1. По каким исключениям Spring откатывает транзакцию?

**Ответ.** По умолчанию — `RuntimeException` и `Error`. Checked-исключения приводят к **коммиту**.

Историческая причина — соглашение EJB, которое Spring унаследовал: checked-исключение трактуется как часть бизнес-контракта («ожидаемый альтернативный исход»), unchecked — как сбой.

```java
@Transactional(rollbackFor = Exception.class)        // откат на всё
public void a() throws IOException { }

@Transactional(noRollbackFor = NotFoundException.class)
public void b() { }

// Программно — работает всегда, даже если исключение не бросается
@Transactional
public void c() {
    if (invalid) TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
}
```

Правило выбора победителя при нескольких правилах: выигрывает **наиболее специфичное** по глубине наследования, а не порядок объявления.

**Главная грабля:** исключение, пойманное **внутри** транзакционного метода и не проброшенное, откат не вызывает вообще. Транзакция закоммитится с частично применёнными изменениями.

---

### D2. Что реально делает `readOnly = true`?

**Ответ.** Три разных эффекта — и уточняющий вопрос обычно именно про это.

1. **Hibernate/JPA.** `JpaTransactionManager` ставит сессии `FlushMode.MANUAL`. Следствия: автоматический flush не происходит, сущности загружаются в read-only режиме, **снимок состояния для dirty checking не создаётся** → заметно меньше памяти и CPU на больших выборках.
2. **JDBC.** `DataSourceUtils.prepareConnectionForTransaction` вызывает `Connection.setReadOnly(true)` — подсказка драйверу. Дополнительно `DataSourceTransactionManager.setEnforceReadOnly(true)` заставляет выполнить `SET TRANSACTION READ ONLY` — вот это уже реальный запрет на уровне сессии PostgreSQL/Oracle.
3. **Маршрутизация.** `readOnly` — стандартный признак для `AbstractRoutingDataSource`, чтобы отправить запрос на реплику:

```java
class RoutingDataSource extends AbstractRoutingDataSource {
    @Override protected Object determineCurrentLookupKey() {
        return TransactionSynchronizationManager.isCurrentTransactionReadOnly()
                ? "replica" : "primary";
    }
}
```

**Что сказать обязательно:** сам по себе `readOnly = true` **не гарантирует** запрет записи — нативный запрос запишет данные без проблем. Это оптимизация и декларация намерения. И: `readOnly` внутри уже открытой read-write транзакции игнорируется.

---

### D3. Можно ли продолжать работу после пойманного исключения персистентности?

**Ответ.** Нет. После `PersistenceException` (например, нарушение constraint на flush) `EntityManager` переходит в неопределённое состояние — JPA-спецификация прямо запрещает его дальнейшее использование, кроме `rollback()`. Транзакция помечена rollback-only.

```java
@Transactional
public void bad(User u) {
    try {
        repo.saveAndFlush(u);                    // ConstraintViolationException
    } catch (DataIntegrityViolationException e) {
        repo.save(fallbackUser());               // ❌ EntityManager уже невалиден
    }
}                                                // → UnexpectedRollbackException
```

Правильные варианты: проверить условие **до** записи (`existsByEmail`), либо выполнить рискованную операцию в `REQUIRES_NEW` отдельного бина, либо поймать исключение **снаружи** транзакции и начать новую.

Отдельно: `@Repository` включает `PersistenceExceptionTranslationPostProcessor`, который транслирует `HibernateException`/`SQLException` в иерархию `DataAccessException`. Это важно и для транзакций: `DataAccessException` — `RuntimeException`, значит откат по умолчанию работает.

---

# Блок E. Транзакции + JPA/Hibernate

### E1. Persistence context и транзакция — как они связаны?

**Ответ.** В типичном Spring-приложении `EntityManager` создаётся на транзакцию (transaction-scoped persistence context) и хранится в `TransactionSynchronizationManager` как `EntityManagerHolder`. Границы транзакции = границы кэша первого уровня.

Что из этого следует:

- один и тот же `findById` в пределах транзакции второй раз не пойдёт в БД;
- сущность, загруженная в транзакции, после её завершения становится **detached**;
- изменения managed-сущности сохраняются **без** вызова `save()` — за счёт dirty checking;
- `flush` происходит: перед commit, перед JPQL-запросом, затрагивающим изменённые таблицы (`FlushMode.AUTO`), и по явному `flush()`.

```java
@Transactional
public void rename(Long id, String name) {
    Order o = repo.findById(id).orElseThrow();  // MANAGED
    o.setName(name);                            // repo.save(o) НЕ НУЖЕН
}                                               // flush + commit здесь
```

Классический вопрос-ловушка: «почему после `findById` и сеттера данные изменились, хотя `save()` не вызывали?» Ответ — dirty checking на flush перед коммитом.

---

### E2. В какой момент реально берётся соединение с БД?

**Ответ.** Тонкий вопрос, который отличает senior.

- `DataSourceTransactionManager` берёт соединение **сразу** в `doBegin()`.
- `JpaTransactionManager` + Hibernate по умолчанию в Spring работает в режиме `DELAYED_ACQUISITION_AND_RELEASE_AFTER_TRANSACTION`: `EntityManager` создаётся сразу, но физическое JDBC-соединение берётся при **первом реальном обращении к БД** и удерживается до конца транзакции.
- `LazyConnectionDataSourceProxy` откладывает получение соединения даже для `DataSourceTransactionManager`.

Почему это важно на практике: метод, помеченный `@Transactional`, но начинающийся с долгого вызова внешнего API, будет удерживать соединение всё это время (если обращение к БД уже было). Отсюда правило: **никаких сетевых вызовов внутри транзакции**.

```java
// ❌ соединение удерживается на время HTTP-запроса
@Transactional
public void process(Long id) {
    Order o = repo.findById(id).orElseThrow();
    var resp = paymentClient.charge(o);          // 2 секунды сетевой задержки
    o.setStatus(resp.status());
}

// ✅ разделить: читаем → вызываем внешний сервис → пишем в короткой транзакции
public void process(Long id) {
    OrderDto o = txTemplate.execute(s -> toDto(repo.findById(id).orElseThrow()));
    var resp = paymentClient.charge(o);
    txTemplate.executeWithoutResult(s -> repo.updateStatus(id, resp.status()));
}
```

Тюнинг: `spring.jpa.properties.hibernate.connection.handling_mode`, `spring.datasource.hikari.maximum-pool-size`, `leak-detection-threshold`.

---

### E3. `LazyInitializationException` и OSIV — как это связано с транзакциями?

**Ответ.** Обращение к lazy-ассоциации после закрытия persistence context, то есть после завершения транзакции.

```java
@Transactional
public Order get(Long id) { return repo.findById(id).orElseThrow(); }

// в контроллере, вне транзакции:
order.getItems().size();      // ❌ LazyInitializationException
```

**Решения по убыванию правильности:** загрузить нужное внутри транзакции (`join fetch` / `@EntityGraph`) → отдавать DTO, а не сущность → `Hibernate.initialize()` точечно → OSIV.

**Почему OSIV — плохой ответ.** `spring.jpa.open-in-view=true` (значение по умолчанию в Boot, с предупреждением в логе) держит `EntityManager` открытым весь HTTP-запрос. Транзакции при этом уже нет, но сессия жива, поэтому lazy-загрузка работает — каждая в **своём auto-commit соединении**. Итог: скрытые N+1 в слое представления, удержание соединения на всё время запроса, невозможность понять по коду, где заканчивается работа с БД. Правильно — `spring.jpa.open-in-view=false` и явная загрузка.

---

### E4. `@Modifying`-запросы и persistence context

**Ответ.** Bulk-запросы (`UPDATE`/`DELETE` в JPQL) выполняются **напрямую в БД, мимо кэша первого уровня**. Сущности, уже загруженные в контекст, останутся со старыми значениями — и dirty checking может затереть ваш bulk-апдейт обратно.

```java
@Modifying(flushAutomatically = true,   // сбросить несохранённые изменения ДО запроса
           clearAutomatically = true)   // очистить контекст ПОСЛЕ запроса
@Query("update Order o set o.status = :s where o.createdAt < :t")
int expireOld(@Param("s") Status s, @Param("t") Instant t);
```

Требует активной read-write транзакции. Также bulk-запросы не запускают каскады и callbacks (`@PreUpdate`, `@PreRemove`) и не обновляют `@Version` автоматически (её нужно инкрементировать в запросе руками).

---

### E5. `save()`, `merge()`, `persist()` в контексте транзакции

**Ответ.**

```java
// SimpleJpaRepository (упрощённо)
@Transactional
public <S extends T> S save(S entity) {
    if (entityInformation.isNew(entity)) { em.persist(entity); return entity; }
    else { return em.merge(entity); }        // ⚠ возвращает КОПИЮ
}
```

- `persist` — transient → managed, тот же объект.
- `merge` — копирует состояние в managed-экземпляр и возвращает его; аргумент остаётся detached; часто делает лишний SELECT.

```java
Order detached = ...;
repo.save(detached);
detached.setStatus(PAID);        // ❌ не сохранится — объект detached

Order managed = repo.save(detached);
managed.setStatus(PAID);         // ✅
```

Методы `SimpleJpaRepository` **сами транзакционные**: читающие — `readOnly = true`, пишущие — обычные. Поэтому вызов репозитория без транзакции на сервисе создаёт транзакцию на каждый вызов:

```java
public void transfer(Long from, Long to, BigDecimal sum) {   // ❌ нет @Transactional
    repo.withdraw(from, sum);   // транзакция 1 — закоммичена
    repo.deposit(to, sum);      // транзакция 2 — упала → деньги исчезли
}
```

---

### E6. Где ставить `@Transactional`: сервис или репозиторий?

**Ответ.** На **сервисном слое** — там проходит граница бизнес-операции (unit of work). Репозиторий не знает, входит ли его вызов в более крупную операцию, поэтому не может корректно определять границу.

Рабочий шаблон:

```java
@Service
@Transactional(readOnly = true)      // дефолт — чтение
class OrderService {

    public OrderDto find(Long id) { }

    @Transactional                   // пишущие методы помечаем явно
    public void place(OrderRequest req) { }
}
```

Контроллер транзакционным делать не надо: транзакция растянется на сериализацию ответа и на любую логику представления.

---

# Блок F. Границы транзакции и производительность

### F1. Что нельзя делать внутри транзакции?

**Ответ (готовый список — очень выигрышный ответ):**

1. **Сетевые вызовы** — HTTP, gRPC, отправка в Kafka/Rabbit. Удерживают соединение, а откатить их нельзя.
2. **Отправку писем и уведомлений** — при откате пользователь получит уведомление о несуществующем событии. Решение — `@TransactionalEventListener(AFTER_COMMIT)`.
3. **Долгие вычисления и работу с файлами** — то же удержание соединения.
4. **`Thread.sleep`, ожидание блокировок, взятие распределённых локов** — прямой путь к исчерпанию пула.
5. **Запуск асинхронных задач, ожидающих данные этой транзакции** — они их ещё не увидят (транзакция не закоммичена).
6. **Работу с несколькими потоками** — транзакция в ThreadLocal.

Правило формулируется так: **транзакция должна быть настолько короткой, насколько возможно, и содержать только работу с БД.**

---

### F2. Как транзакции влияют на пул соединений?

**Ответ.** Транзакция удерживает соединение от `doBegin` (или первого запроса) до commit/rollback. Отсюда:

- время удержания × RPS = требуемый размер пула;
- `REQUIRES_NEW` удваивает потребление на время вложенной транзакции;
- вложенность глубже, чем `pool_size / concurrency`, даёт deadlock по пулу: все потоки держат по соединению и ждут следующего;
- OSIV удерживает соединение (в режиме auto-commit) весь HTTP-запрос.

Диагностика:

```properties
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.leak-detection-threshold=30000   # логировать «протечки»
logging.level.com.zaxxer.hikari.pool.HikariPool=DEBUG
```
Плюс на стороне PostgreSQL: `idle_in_transaction_session_timeout`, мониторинг `pg_stat_activity` по состоянию `idle in transaction` — это прямой индикатор «транзакция открыта, а работы нет».

Ориентир по размеру пула (формула HikariCP): `connections = ((core_count * 2) + effective_spindle_count)`. Большой пул почти всегда хуже маленького.

---

### F3. Как правильно делать пакетную обработку в транзакции?

**Ответ.** Три отдельные проблемы: размер транзакции, размер persistence context, batching JDBC.

```properties
spring.jpa.properties.hibernate.jdbc.batch_size=50
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
spring.jpa.properties.hibernate.batch_versioned_data=true
# и обязательно: rewriteBatchedStatements=true в JDBC URL для MySQL
```

```java
@Transactional
public void importAll(List<Row> rows) {
    for (int i = 0; i < rows.size(); i++) {
        em.persist(map(rows.get(i)));
        if (i % 50 == 0) {          // сбросить батч и очистить контекст
            em.flush();
            em.clear();             // ← иначе OutOfMemory на больших объёмах
        }
    }
}
```

Что добавить в ответ: (1) IDENTITY-генератор идентификаторов **отключает** batching для INSERT — нужен `SEQUENCE` с `allocationSize`; (2) для больших объёмов чище использовать `JdbcTemplate.batchUpdate` или Hibernate `StatelessSession` — без persistence context вообще; (3) миллионы строк не стоит держать в одной транзакции — лучше чанки по N записей, каждый в своей транзакции (и тогда нужна идемпотентность и точка возобновления).

---

# Блок G. Программные транзакции и синхронизации

### G1. `TransactionTemplate` — когда лучше декларативного подхода?

**Ответ.** Когда нужна **точная и узкая** граница транзакции внутри метода, или когда декларативный подход не работает (self-invocation, лямбды, код вне бинов).

```java
@Service
class OrderService {
    private final TransactionTemplate tx;

    OrderService(PlatformTransactionManager tm) {
        this.tx = new TransactionTemplate(tm);
        this.tx.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
        this.tx.setTimeout(5);
    }

    public void process(Long id) {
        Order o = tx.execute(status -> repo.findById(id).orElseThrow());   // короткая tx
        var result = externalApi.call(o);                                  // ВНЕ транзакции
        tx.executeWithoutResult(status -> repo.updateStatus(id, result));  // короткая tx
    }
}
```

Важные детали: `TransactionTemplate` потокобезопасен, если не менять его настройки после конфигурации (иначе — отдельный экземпляр на каждый набор настроек). Откат внутри callback — `status.setRollbackOnly()` или проброс `RuntimeException`. Есть и низкоуровневый вариант с ручными `getTransaction`/`commit`/`rollback`, но он нужен редко.

Ответ на «декларативно или программно»: декларативно по умолчанию, программно — когда нужен контроль над границей внутри метода.

---

### G2. `TransactionSynchronization` — зачем и какие фазы?

**Ответ.** Хук в жизненный цикл транзакции. Позволяет выполнить код строго до/после коммита.

```java
TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
    @Override public void beforeCommit(boolean readOnly) { }
    @Override public void beforeCompletion() { }
    @Override public void afterCommit() {          // транзакция УЖЕ закоммичена
        kafkaTemplate.send("orders", event);
    }
    @Override public void afterCompletion(int status) {
        if (status == STATUS_ROLLED_BACK) { /* компенсация */ }
    }
});
```

Порядок: `beforeCommit` → `beforeCompletion` → **COMMIT** → `afterCommit` → `afterCompletion`.

Ключевое: в `afterCommit`/`afterCompletion` транзакции **уже нет**. Любая работа с БД оттуда потребует новой транзакции (`REQUIRES_NEW` или `TransactionTemplate`). Исключение, брошенное из `afterCommit`, откат уже не вызовет — данные закоммичены.

---

### G3. `@TransactionalEventListener` — как безопасно сделать side-effect после коммита?

**Ответ.** Обычный `@EventListener` выполняется синхронно, в том же потоке и **внутри той же транзакции**. Значит, письмо уйдёт до коммита — и при откате клиент получит уведомление о несуществующем заказе.

```java
record OrderPlaced(Long orderId) { }

@Service
class OrderService {
    private final ApplicationEventPublisher events;

    @Transactional
    public void place(Order o) {
        repo.save(o);
        events.publishEvent(new OrderPlaced(o.getId()));   // публикуем внутри транзакции
    }
}

@Component
class Notifier {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onPlaced(OrderPlaced e) { mail.send(e.orderId()); }
}
```

Фазы: `BEFORE_COMMIT`, `AFTER_COMMIT` (по умолчанию), `AFTER_ROLLBACK`, `AFTER_COMPLETION`.

**Грабли, которые надо назвать сам:**

- в `AFTER_COMMIT` транзакции уже нет — запись в БД требует `@Transactional(propagation = REQUIRES_NEW)`;
- если транзакции нет вообще, листенер **не сработает** — нужен `fallbackExecution = true`;
- по умолчанию всё синхронно: медленный листенер задерживает возврат из метода. `@Async` решает это, но тогда исключение в листенере уже никак не связано с исходной операцией;
- это **не** гарантия доставки: если приложение упадёт между коммитом и отправкой, событие потеряется. Гарантия — только **transactional outbox**: в той же транзакции пишем строку в таблицу `outbox`, отдельный воркер её вычитывает и отправляет с ретраями.

---

# Блок H. Многопоточность, события, распределённые транзакции

### H1. Что произойдёт с транзакцией в другом потоке?

**Ответ.** Ничего — её там не будет. Контекст в ThreadLocal.

```java
@Transactional
public void broken(Order o) {
    repo.save(o);
    CompletableFuture.runAsync(() -> repo.save(other));  // ДРУГАЯ транзакция
    throw new RuntimeException();                        // откатится только save(o)
}
```

Список следствий:

- `@Async`-метод не наследует транзакцию вызывающего;
- параллельные стримы (`parallelStream`) выводят работу из транзакции;
- `EntityManager` не потокобезопасен — передавать между потоками нельзя;
- lazy-поля сущности из другого потока → `LazyInitializationException`;
- асинхронная задача, запущенная **до** коммита, не увидит незакоммиченных данных — типичный баг «сущность не найдена в консьюмере».

Правильные решения: запускать асинхронную работу из `@TransactionalEventListener(AFTER_COMMIT)`; каждый поток открывает **свою** транзакцию; передавать между потоками id, а не сущности.

Единственный способ распределить одну транзакцию на потоки — JTA/XA, и это почти всегда неверный ответ на собеседовании.

---

### H2. Как обеспечить согласованность между БД и брокером сообщений?

**Ответ.** Никакой `@Transactional` не делает атомарными запись в БД и отправку в Kafka. Классические варианты:

1. **2PC / XA** (`JtaTransactionManager` + Atomikos/Narayana). Работает, но: нужны XA-совместимые ресурсы, высокая латентность, блокировки на время фазы подготовки, риск «зависших» транзакций. Kafka XA не поддерживает.
2. **Transactional outbox** — рабочий стандарт. В одной транзакции с бизнес-данными пишем строку в таблицу `outbox`. Отдельный процесс (poller или CDC/Debezium) читает её и публикует в брокер, помечая отправленной.
3. **Saga** — последовательность локальных транзакций с компенсирующими действиями (хореография через события или оркестрация).
4. **`@TransactionalEventListener(AFTER_COMMIT)`** — простейший «почти-outbox»: без гарантий при падении процесса, но подходит для некритичных уведомлений.

```java
@Transactional
public void place(Order o) {
    orderRepo.save(o);
    outboxRepo.save(new OutboxMessage("OrderPlaced", toJson(o)));   // одна транзакция
}

@Scheduled(fixedDelay = 500)
@Transactional
public void publish() {
    outboxRepo.findTop100ByPublishedFalse().forEach(m -> {
        kafka.send(m.topic(), m.payload());
        m.markPublished();
    });
}
```

Обязательно добавьте: outbox даёт **at-least-once**, поэтому потребитель обязан быть идемпотентным. И упомяните `SKIP LOCKED` для параллельных воркеров outbox — это сильный аргумент.

---

### H3. `@Transactional` + `@Async` + `@Cacheable` на одном методе — что будет?

**Ответ.** Всё это разные advice в одной цепочке прокси, и порядок определяется их `order`:

- `@Async` (`AsyncAnnotationBeanPostProcessor`) по умолчанию `Ordered.LOWEST_PRECEDENCE` — обычно оказывается **снаружи**, то есть транзакция открывается уже в новом потоке. Это корректно, но означает, что вызывающая транзакция сюда не распространяется;
- `@Cacheable` и `@Transactional` тоже упорядочиваемы (`@EnableCaching(order = ...)`, `@EnableTransactionManagement(order = ...)`);
- все три ломаются одинаково при self-invocation.

Отдельно про кэш: `@CacheEvict` по умолчанию выполняется **после** метода, но **до** коммита транзакции. При откате кэш уже очищен (не страшно) — а вот `@CachePut` при откате запишет в кэш данные, которых в БД нет. Правильное решение — инвалидировать кэш из `@TransactionalEventListener(AFTER_COMMIT)` или использовать `TransactionAwareCacheManagerProxy`.

---

# Блок I. Тестирование транзакций

### I1. Как работает `@Transactional` в тестах?

**Ответ.** Тестовый класс/метод с `@Transactional` выполняется в транзакции, которая **откатывается после теста** — база остаётся чистой без ручной очистки. `@DataJpaTest` включает это по умолчанию.

```java
@DataJpaTest
class OrderRepositoryTest {

    @Test
    void saves() { ... }                 // откат после теста

    @Test @Commit                        // или @Rollback(false)
    void commits() { ... }
}
```

Дополнительные инструменты:

```java
@BeforeTransaction  void before() { }    // до старта тестовой транзакции
@AfterTransaction   void after()  { }    // после отката/коммита
@Sql("/data.sql")                        // executionPhase = BEFORE_TEST_METHOD по умолчанию

@Test
void manual() {
    TestTransaction.flagForCommit();
    TestTransaction.end();               // коммит текущей
    TestTransaction.start();             // новая транзакция
}
```

---

### I2. Какие ошибки чаще всего допускают в транзакционных тестах?

**Ответ — готовый список:**

1. **Тест «зелёный», прод падает.** Тестовая транзакция маскирует `LazyInitializationException`: в тесте контекст открыт весь метод, в проде — нет.
2. **Не видно реального flush.** Изменения не уходят в БД до коммита, поэтому constraint-нарушения не проявляются. Лечится явным `entityManager.flush()` или `saveAndFlush()` в тесте.
3. **`@SpringBootTest(webEnvironment = RANDOM_PORT)` + `@Transactional`.** Запрос идёт по HTTP в другой поток — тестовая транзакция на него не распространяется, откат не работает, данные остаются в БД.
4. **Проверка «сохранилось» без очистки контекста.** `assertThat(repo.findById(id))` вернёт объект из кэша первого уровня, а не из БД. Нужен `em.flush(); em.clear();`.
5. **Тестирование `REQUIRES_NEW` внутри откатываемой тестовой транзакции** даёт неинтуитивный результат: вложенная реально закоммитится и переживёт откат теста.

Хороший ответ на «как тестировать транзакционное поведение по-настоящему»: без `@Transactional` на тесте, с Testcontainers и явной очисткой данных — тогда поведение совпадает с продом.

---

# Блок J. Реактивные транзакции

### J1. Как устроены транзакции в WebFlux/R2DBC?

**Ответ.** ThreadLocal там не работает — запрос обрабатывается разными потоками event loop. Поэтому Spring вводит параллельную иерархию: `ReactiveTransactionManager` (реализация — `R2dbcTransactionManager`), а контекст транзакции хранится в **Reactor Context**, а не в `TransactionSynchronizationManager`.

```java
@Service
class OrderService {
    private final TransactionalOperator tx;    // из ReactiveTransactionManager

    public Mono<Order> place(Order o) {
        return repo.save(o)
                   .flatMap(this::charge)
                   .as(tx::transactional);     // границы транзакции — весь reactive-пайплайн
    }
}
```

`@Transactional` на методе, возвращающем `Mono`/`Flux`, тоже поддерживается — Spring оборачивает возвращаемый Publisher.

Ключевые отличия, которые стоит назвать: транзакция привязана к **подписке**, а не к потоку; блокирующий JDBC-код внутри реактивной транзакции недопустим; `NESTED` в R2DBC не поддерживается; смешивать `PlatformTransactionManager` и `ReactiveTransactionManager` в одном потоке выполнения нельзя.

Если реактивщины в вашем опыте нет — так и скажите, но покажите понимание причины: «ThreadLocal не подходит, потому что цепочка выполняется на разных потоках, поэтому контекст переехал в Reactor Context».

---
## Часть 3. Задачи «что произойдёт»

Формат, в котором транзакции спрашивают чаще всего: дают код и спрашивают результат. Закройте ответы и пройдите все 12.

---

### Задача 1

```java
@Service
class A {
    @Transactional
    public void outer() {
        repo.save(new Order("1"));
        inner();
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void inner() {
        repo.save(new Order("2"));
        throw new RuntimeException();
    }
}
```

**Вопрос:** какие заказы окажутся в БД?

<details><summary>Ответ</summary>

**Ни одного.** Это self-invocation: `inner()` вызван через `this`, прокси минован, `REQUIRES_NEW` не применяется. Метод выполняется в той же транзакции, исключение доходит до `outer()`, откатывается всё.

Если бы `inner()` был в другом бине, исключение всё равно пробросилось бы в `outer()` и откатило его — сохранился бы только `"2"`, и только при условии, что `outer()` исключение ловит.
</details>

---

### Задача 2

```java
@Transactional
public void outer() {
    try { inner(); } catch (Exception e) { log.warn("ignored"); }
    repo.save(new Order("after"));
}

// в ДРУГОМ бине
@Transactional              // REQUIRED
public void inner() { throw new IllegalStateException(); }
```

**Вопрос:** сохранится ли `"after"`?

<details><summary>Ответ</summary>

**Нет.** `inner()` участвует в транзакции `outer()`. При его падении Spring помечает физическую транзакцию `rollbackOnly` (`globalRollbackOnParticipationFailure = true`). Поймать исключение можно, снять пометку — нет. На коммите `outer()` произойдёт откат и вылетит `UnexpectedRollbackException`.

Починка: `REQUIRES_NEW` на `inner()`, либо `noRollbackFor`, либо вовсе убрать `@Transactional` с `inner()`.
</details>

---

### Задача 3

```java
@Transactional
public void update(Long id) {
    Order o = repo.findById(id).orElseThrow();
    o.setStatus(Status.PAID);
    // repo.save(o) не вызывается
}
```

**Вопрос:** изменится ли статус в БД?

<details><summary>Ответ</summary>

**Да.** Сущность managed, на flush перед коммитом Hibernate сравнит её со снимком (dirty checking) и сгенерирует UPDATE.

Уточнение: при `@Transactional(readOnly = true)` — **нет**, потому что `FlushMode` = `MANUAL` и снимок не создаётся.
</details>

---

### Задача 4

```java
@Transactional
public void process() throws IOException {
    repo.save(new Order("1"));
    throw new IOException("boom");
}
```

**Вопрос:** будет ли откат?

<details><summary>Ответ</summary>

**Нет, транзакция закоммитится.** `IOException` — checked, а по умолчанию откат только на `RuntimeException` и `Error`. Заказ `"1"` останется в БД.

Починка: `@Transactional(rollbackFor = Exception.class)`.
</details>

---

### Задача 5

```java
@Transactional
public void run() {
    repo.save(new Order("1"));
    CompletableFuture.runAsync(() -> repo.save(new Order("2"))).join();
    throw new RuntimeException();
}
```

**Вопрос:** что останется в БД?

<details><summary>Ответ</summary>

**Только `"2"`.** Асинхронная задача выполняется в другом потоке, где транзакционного контекста нет, — Spring Data откроет для неё собственную транзакцию, и она закоммитится сразу. Транзакция `run()` откатится, `"1"` пропадёт.

Дополнительно: если бы `"2"` ссылался на `"1"` по внешнему ключу, было бы нарушение FK — другой поток не видит незакоммиченных данных.
</details>

---

### Задача 6

```java
@Service
@Transactional(readOnly = true)
class ReportService {
    @Transactional
    public void refresh() { repo.save(new Report()); }
}
```

**Вопрос:** будет ли метод read-only?

<details><summary>Ответ</summary>

**Нет.** Аннотация на методе перекрывает аннотацию на классе, а `@Transactional` без атрибутов означает `readOnly = false`. Метод read-write, запись пройдёт.

Но если `refresh()` вызвать **изнутри** другого read-only метода того же класса, прокси будет минован: транзакция останется read-only, и изменения молча не сохранятся (JPA, `FlushMode.MANUAL`).
</details>

---

### Задача 7

```java
@Transactional(isolation = Isolation.SERIALIZABLE)
public void inner() { }

@Transactional(isolation = Isolation.READ_COMMITTED)
public void outer() { innerBean.inner(); }
```

**Вопрос:** с каким уровнем изоляции выполнится `inner()`?

<details><summary>Ответ</summary>

**`READ_COMMITTED`.** Isolation применяется только при старте физической транзакции. `inner()` с `REQUIRED` присоединяется к существующей, его isolation игнорируется — молча.

Чтобы такое не проходило незамеченным: `AbstractPlatformTransactionManager.setValidateExistingTransaction(true)` → `IllegalTransactionStateException`.
</details>

---

### Задача 8

```java
@Transactional
public void bulkAndRead() {
    Order o = repo.findById(1L).orElseThrow();   // status = NEW
    repo.expireAll();                            // @Modifying UPDATE ... SET status = EXPIRED
    System.out.println(o.getStatus());
}
```

**Вопрос:** что напечатается?

<details><summary>Ответ</summary>

**`NEW`.** Bulk-запрос выполнен напрямую в БД, минуя persistence context. Объект `o` в кэше первого уровня остался со старым значением.

Хуже: если бы `o` был изменён, dirty checking на коммите записал бы старое значение поверх результата bulk-апдейта. Решение — `@Modifying(clearAutomatically = true, flushAutomatically = true)`.
</details>

---

### Задача 9

```java
@Transactional
public void place(Order o) {
    repo.save(o);
    kafka.send("orders", new OrderPlaced(o.getId()));
}
```

**Вопрос:** какие здесь проблемы?

<details><summary>Ответ</summary>

1. **Нет атомарности.** Сообщение уходит до коммита. Если коммит упадёт — консьюмеры получат событие о заказе, которого нет в БД.
2. **Гонка.** Даже при успешном коммите консьюмер может обработать событие раньше, чем транзакция закоммитится, и не найдёт заказ по id.
3. **Сетевой вызов внутри транзакции** удерживает соединение с БД на время отправки.

Решение: `@TransactionalEventListener(AFTER_COMMIT)` для некритичного, transactional outbox — для критичного.
</details>

---

### Задача 10

```java
@Transactional
public void register(User u) {
    try {
        repo.saveAndFlush(u);                   // упадёт на unique constraint
    } catch (DataIntegrityViolationException e) {
        u.setEmail(u.getEmail() + ".dup");
        repo.saveAndFlush(u);                   // повторная попытка
    }
}
```

**Вопрос:** сработает ли повторная попытка?

<details><summary>Ответ</summary>

**Нет.** После `PersistenceException` `EntityManager` в неопределённом состоянии, транзакция помечена rollback-only. Вторая запись либо упадёт, либо не закоммитится — на выходе `UnexpectedRollbackException`.

Правильно: проверить `existsByEmail` до записи; либо вынести попытку в `REQUIRES_NEW` в другом бине; либо ловить исключение снаружи транзакции и начинать новую.
</details>

---

### Задача 11

```java
@Transactional
public void charge(Long id) {
    Order o = repo.findById(id).orElseThrow();
    var resp = httpClient.charge(o);      // внешний вызов, p99 = 3 сек
    o.setStatus(resp.status());
}
```

**Вопрос:** что сломается под нагрузкой?

<details><summary>Ответ</summary>

**Пул соединений.** Соединение захвачено первым же `findById` и удерживается все 3 секунды сетевого вызова. При пуле в 20 это ограничивает пропускную способность примерно 6–7 запросами в секунду; дальше — `SQLTransientConnectionException: connection is not available, request timed out`.

В PostgreSQL это видно как сессии в состоянии `idle in transaction`.

Плюс: если внешний сервис отвечает, а транзакция потом откатывается, деньги списаны, а статус не сохранён — нужна идемпотентность и сверка.

Решение: три шага — короткая транзакция на чтение, вызов вне транзакции, короткая транзакция на запись.
</details>

---

### Задача 12

```java
@Service
class OrderService {
    @Retryable(retryFor = ObjectOptimisticLockingFailureException.class, maxAttempts = 3)
    @Transactional
    public void update(Long id) {
        Order o = repo.findById(id).orElseThrow();
        o.setCounter(o.getCounter() + 1);
    }
}
```

**Вопрос:** сработает ли ретрай?

<details><summary>Ответ</summary>

**Скорее всего нет.** `OptimisticLockException` возникает на **коммите**, то есть уже за границей метода. Порядок advice здесь критичен: если `@Retryable` оказался внутри транзакционного advice, исключение до него просто не дойдёт — оно бросается из `TransactionInterceptor` уже после выхода из метода.

Правильно — разнести по бинам, чтобы retry гарантированно был снаружи:

```java
@Service
class OrderFacade {
    @Retryable(retryFor = ObjectOptimisticLockingFailureException.class, maxAttempts = 3,
               backoff = @Backoff(delay = 50, multiplier = 2))
    public void update(Long id) { orderService.update(id); }   // @Transactional внутри
}
```

Общее правило: **ретрай всегда снаружи транзакции**, потому что повторять нужно транзакцию целиком, а не её часть.
</details>

---

## Часть 4. Шпаргалка перед собеседованием

### Механика в одном абзаце

`@EnableTransactionManagement` → `BeanFactoryTransactionAttributeSourceAdvisor` + `TransactionInterceptor` → прокси (в Boot — CGLIB). Вызов: `TransactionAspectSupport.invokeWithinTransaction` читает `TransactionAttribute`, выбирает `PlatformTransactionManager`, тот через `doBegin/handleExistingTransaction` кладёт `ConnectionHolder`/`EntityManagerHolder` в `TransactionSynchronizationManager` (ThreadLocal). `JdbcTemplate` и Hibernate достают ресурс оттуда. На выходе — commit или rollback по `rollbackOn(ex)`.

### Грабли, которые надо называть самому

| Тема | Грабля |
|---|---|
| Прокси | self-invocation; `private`/`protected`/`final`/`static`; аннотация на интерфейсе при CGLIB |
| `REQUIRED` вложенный | `UnexpectedRollbackException` после пойманного исключения внутреннего метода |
| `REQUIRES_NEW` | второе соединение из пула → deadlock по пулу; самоблокировка по строке; нет атомарности |
| `NESTED` | не поддерживается `JpaTransactionManager` по умолчанию; savepoint не откатывает persistence context |
| Присоединение | `isolation`, `timeout`, `readOnly` внутреннего метода игнорируются |
| Rollback | checked-исключения не откатывают; пойманное исключение = отката нет вовсе |
| `readOnly` | не запрещает запись; отключает dirty checking; полезен для роутинга на реплику |
| `timeout` | лимит на запрос, а не на транзакцию; сетевые вызовы не прерывает |
| Persistence context | после `PersistenceException` `EntityManager` невалиден |
| `@Modifying` | идёт мимо кэша первого уровня → нужен `clearAutomatically` |
| Соединение | Hibernate берёт его лениво, но держит до конца транзакции |
| OSIV | включён по умолчанию; держит сессию весь HTTP-запрос |
| Потоки | контекст в ThreadLocal → `@Async`, `CompletableFuture`, parallelStream выпадают из транзакции |
| Kafka/HTTP | нет атомарности с БД → outbox или `@TransactionalEventListener(AFTER_COMMIT)` |
| `@CachePut` | при откате кэш содержит несуществующие данные |
| Ретрай | всегда снаружи транзакции |
| Несколько БД | два `TransactionManager` ≠ атомарность; `ChainedTransactionManager` deprecated |
| Тесты | `@Transactional` в тесте маскирует `LazyInitializationException`; с `RANDOM_PORT` откат не работает |

### Имена, которые стоит произнести вслух хотя бы раз

`TransactionInterceptor`, `TransactionAspectSupport`, `AnnotationTransactionAttributeSource`, `PlatformTransactionManager`, `AbstractPlatformTransactionManager`, `JpaTransactionManager`, `DataSourceTransactionManager`, `TransactionSynchronizationManager`, `ConnectionHolder`, `EntityManagerHolder`, `TransactionStatus.isNewTransaction()`, `TransactionTemplate`, `TransactionSynchronization.afterCommit`, `UnexpectedRollbackException`, `IllegalTransactionStateException`, `NestedTransactionNotSupportedException`, `ObjectOptimisticLockingFailureException`.

### Настройки, которые полезно знать наизусть

```properties
spring.jpa.open-in-view=false
spring.jpa.properties.hibernate.jdbc.batch_size=50
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.connection.handling_mode=DELAYED_ACQUISITION_AND_RELEASE_AFTER_TRANSACTION
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.leak-detection-threshold=30000
spring.transaction.default-timeout=30
spring.transaction.rollback-on-commit-failure=true

# диагностика
logging.level.org.springframework.transaction.interceptor=TRACE
logging.level.org.springframework.orm.jpa.JpaTransactionManager=DEBUG
logging.level.org.hibernate.SQL=DEBUG
```

### Последние 3 дня

1. **День −3.** Пройти все вопросы блоков A–E вслух, по 60 секунд на каждый. Отметить проваленные.
2. **День −2.** Только проваленные + все 12 задач Части 3. Ответы писать текстом — это фиксирует формулировки.
3. **День −1.** Нарисовать на бумаге по памяти: (а) путь вызова от прокси до commit; (б) таблицу propagation; (в) сценарий `UnexpectedRollbackException`. Плюс подготовить 2–3 истории «у нас в проекте было так» — на senior это весит больше формулировок из документации.

**Если вопрос неизвестен:** не выдумывайте. Скажите, что не сталкивались, и рассуждайте от принципов: «раз это работает через прокси и ThreadLocal, то, скорее всего…». Ход мысли ценится выше заученности.

---
---

# Часть 5. Hibernate — что надо знать для интервью

## План на 5 дней (дополнительно к транзакционному)

| День | Тема | Проверка себя |
|---|---|---|
| 1 | Архитектура, persistence context, состояния сущности, flush и dirty checking | Рисую жизненный цикл сущности и порядок операций на flush |
| 2 | Маппинг: генераторы id, ассоциации, каскады, коллекции, наследование | Объясняю, почему `IDENTITY` ломает batch-inserts |
| 3 | Загрузка: lazy/eager, прокси, N+1, `@EntityGraph`, `@BatchSize`, пагинация с fetch | Называю 4 способа убрать N+1 и границы применимости каждого |
| 4 | Запросы, проекции, кэши всех уровней, блокировки | Объясняю, когда L2-кэш вреден |
| 5 | Производительность, batch, диагностика, разбор всех исключений + задачи Части 6 | Отвечаю на все вопросы блоков K–T за ≤60 сек |

**Главная мысль всего раздела, которую стоит держать в голове:** Hibernate — это не «библиотека, которая пишет SQL за вас», а **менеджер состояния объектов в памяти**, который синхронизирует это состояние с БД в момент flush. Почти все вопросы на собеседовании сводятся к двум: «в каком состоянии сейчас объект» и «когда именно полетит SQL».

---

# Блок K. Архитектура и базовые понятия

### K1. JPA, Hibernate, Spring Data JPA — как соотносятся?

**Ответ.**

- **JPA (Jakarta Persistence)** — спецификация: интерфейсы `EntityManager`, `EntityManagerFactory`, аннотации маппинга, язык JPQL, `Criteria API`. Кода нет, только контракт.
- **Hibernate** — реализация JPA (и самостоятельный ORM со своим API, который шире спецификации: `Session`, HQL, `@BatchSize`, `@Filter`, Envers, `StatelessSession`).
- **Spring Data JPA** — надстройка **над JPA**, а не над Hibernate: генерирует реализации репозиториев, разбирает имена методов, добавляет `Pageable`, аудит, проекции.

Соответствие типов:

| JPA | Hibernate |
|---|---|
| `EntityManagerFactory` | `SessionFactory` |
| `EntityManager` | `Session` |
| `EntityTransaction` | `Transaction` |
| JPQL | HQL (надмножество) |
| `Query`, `TypedQuery` | `org.hibernate.query.Query` |

В Spring из `EntityManager` всегда можно достать нативный API:

```java
@PersistenceContext private EntityManager em;

Session session = em.unwrap(Session.class);
session.enableFilter("activeOnly");            // возможности вне JPA
```

**Что стоит добавить:** `SessionFactory` — тяжёлый, потокобезопасный, один на приложение. `Session`/`EntityManager` — лёгкий, **не потокобезопасный**, живёт в пределах транзакции. Это объясняет, почему `EntityManager` нельзя передавать между потоками и почему Spring инжектит не сам `EntityManager`, а его thread-bound прокси.

---

### K2. Что такое persistence context и зачем он нужен?

**Ответ.** Persistence context — область памяти, где Hibernate держит managed-сущности. Это одновременно:

1. **Кэш первого уровня** — повторный `find` по тому же id не идёт в БД.
2. **Гарантия идентичности** — в пределах одного контекста для одной строки БД существует **ровно один** объект: `em.find(User.class, 1L) == em.find(User.class, 1L)` вернёт `true`.
3. **Механизм отслеживания изменений** — при загрузке сохраняется снимок полей, на flush он сравнивается с текущим состоянием (dirty checking).
4. **Буфер записи (write-behind)** — операции накапливаются в `ActionQueue` и выполняются пачкой на flush, а не сразу.

```java
@Transactional
public void demo() {
    User a = em.find(User.class, 1L);   // SELECT
    User b = em.find(User.class, 1L);   // из кэша, SQL нет
    assert a == b;                      // тот же объект

    a.setName("new");                   // пока только в памяти
}                                       // flush → UPDATE → COMMIT
```

В Spring контекст по умолчанию **transaction-scoped**: создаётся при старте транзакции, закрывается при её завершении. Отсюда `LazyInitializationException` за границей транзакции.

---

### K3. Когда происходит flush и что это такое?

**Ответ.** Flush — синхронизация состояния persistence context с БД: генерация и отправка накопленных INSERT/UPDATE/DELETE. **Flush ≠ commit**: данные уходят в БД, но транзакция ещё открыта и может откатиться.

**Триггеры flush при `FlushMode.AUTO` (по умолчанию):**

1. Перед `commit`.
2. Перед выполнением JPQL/HQL/Criteria-запроса, который **может задеть** изменённые таблицы (Hibernate анализирует, какие пространства запросов затронуты).
3. При явном `em.flush()`.

**Режимы:**

| Режим | Поведение |
|---|---|
| `AUTO` (JPA, по умолчанию) | как описано выше |
| `COMMIT` | только перед коммитом; запросы могут видеть устаревшие данные |
| `ALWAYS` (Hibernate) | перед каждым запросом |
| `MANUAL` (Hibernate) | только по явному `flush()`; включается при `@Transactional(readOnly = true)` |

**Грабля с нативными запросами:** нативный SQL Hibernate не парсит, поэтому не знает, какие таблицы он трогает, и flush перед ним может не произойти — нативный запрос увидит старые данные. Лечится `em.flush()` вручную или `@Modifying(flushAutomatically = true)` в Spring Data.

**Порядок операций на flush** (`ActionQueue`, важный и редко известный факт):

```
1. OrphanRemovalAction
2. INSERT сущностей (в порядке persist)
3. UPDATE сущностей
4. Удаление элементов коллекций
5. Обновление элементов коллекций
6. Вставка элементов коллекций
7. DELETE сущностей
```

**Следствие:** DELETE всегда идёт **последним**, поэтому классический сценарий «удалить строку и вставить новую с тем же уникальным ключом» падает на constraint — Hibernate выполнит INSERT раньше DELETE. Решение — явный `em.flush()` между операциями. Это отличная деталь для ответа.

---

### K4. Что такое dirty checking и сколько он стоит?

**Ответ.** При загрузке сущности Hibernate сохраняет **снимок** (loaded state) её полей. На flush он проходит по всем managed-сущностям и сравнивает текущие значения со снимком; для изменённых генерирует UPDATE.

```java
@Transactional
public void rename(Long id) {
    Order o = repo.findById(id).orElseThrow();
    o.setName("new");             // save() НЕ нужен
}                                 // dirty checking → UPDATE
```

**Стоимость.** Проверка — O(количество managed-сущностей × количество полей), выполняется на **каждом** flush. При тысячах сущностей в контексте это заметно. Способы снизить:

- `@Transactional(readOnly = true)` — снимок не создаётся вообще;
- периодический `em.clear()` в пакетной обработке;
- **bytecode enhancement** (`hibernate-enhance-maven-plugin` с `enableDirtyTracking`) — сущность сама отслеживает изменения через сгенерированный код, полное сравнение не нужно;
- `StatelessSession` — persistence context отсутствует.

**Связанные аннотации:**

```java
@Entity
@DynamicUpdate      // UPDATE только изменённых колонок вместо всех
@DynamicInsert      // INSERT только непустых колонок
class Order { }
```
`@DynamicUpdate` полезен для широких таблиц и при конкурентных обновлениях разных колонок, но лишает Hibernate возможности кэшировать подготовленный SQL — применять точечно.

---

# Блок L. Жизненный цикл сущности и persistence context

### L1. Четыре состояния сущности

**Ответ.**

```
        new Entity()
             │
         TRANSIENT ──persist()──► MANAGED ──remove()──► REMOVED
                                    │  ▲                    │
                          detach()  │  │ merge()            │ flush
                          clear()   │  │ find()             ▼
                          close()   ▼  │                  DELETE
                                 DETACHED
```

| Состояние | Есть id | В контексте | Синхронизируется с БД |
|---|---|---|---|
| **Transient** | нет | нет | нет |
| **Managed** | да | да | да (dirty checking) |
| **Detached** | да | нет | нет |
| **Removed** | да | да | будет DELETE на flush |

```java
Order o = new Order();          // TRANSIENT
em.persist(o);                  // MANAGED, id присвоен
em.detach(o);                   // DETACHED
o.setStatus(PAID);              // изменения НЕ отслеживаются
Order merged = em.merge(o);     // merged — MANAGED, o остаётся DETACHED
em.remove(merged);              // REMOVED
```

Полезные операции: `em.refresh(e)` — перезагрузить из БД, затерев локальные изменения; `em.contains(e)` — проверить, managed ли объект; `em.clear()` — отсоединить всё.

---

### L2. `persist` vs `merge` vs `save` vs `saveOrUpdate`

**Ответ.**

| Метод | Аргумент | Возврат | Поведение |
|---|---|---|---|
| `persist(e)` | transient | `void` | тот же объект становится managed; для detached — `EntityExistsException` |
| `merge(e)` | любое | **новый** managed-объект | копирует состояние; аргумент остаётся detached; может сделать SELECT |
| `save(e)` (Hibernate) | transient/detached | id | немедленно назначает id, часто вызывая INSERT сразу |
| `update(e)` (Hibernate) | detached | `void` | привязывает объект к сессии; при наличии другого экземпляра с тем же id — `NonUniqueObjectException` |
| `repository.save(e)` (Spring Data) | любое | managed-объект | `isNew() ? persist : merge` |

```java
// Классическая ошибка
Order detached = getFromCache();
repo.save(detached);
detached.setStatus(PAID);        // ❌ не сохранится: detached не managed

Order managed = repo.save(detached);
managed.setStatus(PAID);         // ✅
```

**Как `isNew()` определяется в Spring Data:** по `null` в поле id, либо по `null` в `@Version`, либо через `Persistable.isNew()`. Отсюда важная проблема: при **назначаемых вручную** идентификаторах (UUID) объект всегда выглядит «не новым», и `save()` делает `merge` с лишним SELECT на каждую вставку.

```java
@Entity
class Document implements Persistable<UUID> {
    @Id private UUID id = UUID.randomUUID();
    @Transient private boolean isNew = true;

    @Override public UUID getId() { return id; }
    @Override public boolean isNew() { return isNew; }

    @PostPersist @PostLoad void markNotNew() { this.isNew = false; }
}
```

---

### L3. Как работает `remove()` и почему сущность «не удаляется»?

**Ответ.** `em.remove(e)` требует **managed**-сущности. Для detached — `IllegalArgumentException`. Spring Data `deleteById` сначала делает `findById`, потом `remove` — отсюда лишний SELECT.

Частые причины «не удалилось»:

1. Сущность на самом деле detached → надо `em.remove(em.merge(e))` или загрузить заново.
2. Есть ссылки из других сущностей → нарушение FK. Нужно почистить обе стороны двунаправленной связи.
3. `@OneToMany` без `orphanRemoval`/каскада: удаление родителя не удаляет детей → FK-ошибка.
4. Удаление произошло, но объект остался в коллекции родителя в памяти — при flush коллекция «воскресит» ссылку.

```java
// Правильный helper для двунаправленной связи
public void removeItem(Item item) {
    items.remove(item);
    item.setOrder(null);        // синхронизируем ОБЕ стороны
}
```

**Bulk-удаление** быстрее, но обходит каскады и коллбэки:

```java
@Modifying(clearAutomatically = true)
@Query("delete from Order o where o.createdAt < :t")
int deleteOld(@Param("t") Instant t);
```

---

### L4. Callbacks сущностей и аудит

**Ответ.** JPA-коллбэки: `@PrePersist`, `@PostPersist`, `@PreUpdate`, `@PostUpdate`, `@PreRemove`, `@PostRemove`, `@PostLoad`. Можно вынести в отдельный класс через `@EntityListeners`.

```java
@Entity
@EntityListeners(AuditingEntityListener.class)     // Spring Data JPA auditing
class Order {
    @CreatedDate  Instant createdAt;
    @LastModifiedDate Instant updatedAt;
    @CreatedBy    String createdBy;                // нужен AuditorAware<String>

    @PrePersist void onCreate() { this.uuid = UUID.randomUUID(); }
}
```

Для Spring Data auditing нужен `@EnableJpaAuditing` и бин `AuditorAware`.

**Ограничения, которые надо назвать:** внутри коллбэка нельзя обращаться к `EntityManager` и другим сущностям — поведение не определено спецификацией и приводит к трудноуловимым багам. Коллбэки **не срабатывают** на bulk-запросах (`@Modifying`, `delete from ...`). Для полноценного аудита изменений с историей — Hibernate Envers (`@Audited`), который ведёт отдельные `_AUD`-таблицы.

---

# Блок M. Маппинг: идентификаторы, ассоциации, каскады, коллекции

### M1. Стратегии генерации идентификаторов — какую выбрать?

**Ответ.**

| Стратегия | Как работает | Batch INSERT | Комментарий |
|---|---|---|---|
| `IDENTITY` | auto-increment колонка БД | **❌ отключается** | id известен только после INSERT |
| `SEQUENCE` | последовательность БД | ✅ | рекомендуемый вариант для PostgreSQL/Oracle |
| `TABLE` | отдельная таблица-счётчик | ✅ | медленно, блокировки — не использовать |
| `AUTO` | выбирает провайдер | зависит | Hibernate 6 выбирает `SEQUENCE`, где это возможно |
| назначаемый вручную | UUID и т.п. | ✅ | нужен `Persistable`, иначе лишний SELECT |

**Почему `IDENTITY` ломает batching — обязательный к пониманию момент.** Hibernate использует write-behind: операции копятся и выполняются пачкой на flush. Но при `IDENTITY` идентификатор известен только после реального INSERT, а id нужен немедленно, чтобы положить сущность в persistence context. Поэтому Hibernate вынужден выполнять INSERT сразу при `persist()`, и объединять их в JDBC-батч уже невозможно.

```java
@Entity
class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "order_seq")
    @SequenceGenerator(name = "order_seq", sequenceName = "order_seq",
                       allocationSize = 50)     // ⚠ должно совпадать с INCREMENT BY в БД
    private Long id;
}
```

`allocationSize` — размер «пула» идентификаторов, выбираемых за одно обращение к последовательности (по умолчанию 50). Если в БД `INCREMENT BY 1`, а в коде `allocationSize = 50`, получите конфликты id. Либо приведите их в соответствие, либо поставьте `allocationSize = 1` (ценой обращения к БД на каждую вставку).

Ещё деталь: в Hibernate 6 исчез глобальный `hibernate_sequence` — по умолчанию для каждой сущности используется своя последовательность `<таблица>_SEQ`. Это ломало миграции с Hibernate 5.

---

### M2. Типы ассоциаций и какие fetch-стратегии у них по умолчанию

**Ответ.**

| Ассоциация | FetchType по умолчанию |
|---|---|
| `@ManyToOne` | **EAGER** |
| `@OneToOne` | **EAGER** |
| `@OneToMany` | LAZY |
| `@ManyToMany` | LAZY |

**Практическая рекомендация:** явно ставить `LAZY` везде, включая `@ManyToOne`. EAGER по умолчанию — источник неконтролируемых JOIN'ов и N+1: каждый `@ManyToOne` тянет за собой родителя, тот — своего родителя, и один `findAll()` превращается в дерево запросов.

```java
@Entity
class OrderItem {
    @ManyToOne(fetch = FetchType.LAZY)          // ← всегда явно
    @JoinColumn(name = "order_id")
    private Order order;
}
```

**Проблема lazy `@OneToOne`.** На **необладающей** стороне (`mappedBy`) `LAZY` не работает без bytecode enhancement: Hibernate обязан знать, есть ли связанная запись, чтобы решить — подставить прокси или `null`, а узнать это можно только запросом. Обходы: `@OneToOne(optional = false)` + `@MapsId` (общий первичный ключ), либо bytecode enhancement, либо смоделировать связь как `@ManyToOne` на владеющей стороне.

**Двунаправленные связи** требуют синхронизации обеих сторон — Hibernate ориентируется только на владеющую сторону (ту, где есть `@JoinColumn`):

```java
@Entity
class Order {
    @OneToMany(mappedBy = "order", cascade = ALL, orphanRemoval = true)
    private List<Item> items = new ArrayList<>();

    public void addItem(Item i) { items.add(i); i.setOrder(this); }     // обе стороны
    public void removeItem(Item i) { items.remove(i); i.setOrder(null); }
}
```

**`@ManyToMany`:** на собеседовании хороший ответ — «стараюсь не использовать». Причины: нельзя добавить атрибуты в связующую таблицу, при изменении коллекции Hibernate часто удаляет все связи и вставляет заново, каскады работают неочевидно. Практика — разложить на две `@OneToMany` через явную сущность-связку.

---

### M3. Каскады и `orphanRemoval`

**Ответ.**

| Cascade | Что распространяет |
|---|---|
| `PERSIST` | `em.persist()` |
| `MERGE` | `em.merge()` |
| `REMOVE` | `em.remove()` |
| `REFRESH` | `em.refresh()` |
| `DETACH` | `em.detach()` |
| `ALL` | всё перечисленное |

Hibernate добавляет свои: `SAVE_UPDATE`, `DELETE`, `LOCK`, `REPLICATE`.

**`orphanRemoval = true` vs `CascadeType.REMOVE` — частый уточняющий вопрос:**

- `CascadeType.REMOVE` — удаление **родителя** удаляет детей.
- `orphanRemoval = true` — дополнительно: удаление ребёнка **из коллекции** удаляет его строку из БД.

```java
@OneToMany(mappedBy = "order", cascade = ALL, orphanRemoval = true)
private List<Item> items;

order.getItems().remove(item);      // orphanRemoval → DELETE FROM item
                                    // без него → просто ошибка FK или «висячая» строка
```

**Правило:** каскады уместны только для отношения **композиции** (ребёнок не существует без родителя: заказ → позиции заказа). Для ассоциаций между самостоятельными сущностями (`@ManyToOne` на справочник) каскады — путь к случайному удалению половины базы.

---

### M4. Коллекции: `Set`, `List`, bag — в чём разница и где грабли?

**Ответ.**

| Тип | Семантика Hibernate | Особенности |
|---|---|---|
| `Set<T>` | set | без дубликатов; требует корректных `equals`/`hashCode`; безопасен для нескольких fetch join |
| `List<T>` без `@OrderColumn` | **bag** | допускает дубликаты, порядок не сохраняется |
| `List<T>` + `@OrderColumn` | indexed list | порядок хранится в колонке; обновления дороже |
| `Map<K,V>` | map | `@MapKey`/`@MapKeyColumn` |

**Две классические проблемы:**

1. **`MultipleBagFetchException`** — нельзя одновременно `join fetch` двух bag-коллекций: получилось бы декартово произведение, а восстановить из него исходные коллекции невозможно.

```java
// ❌ org.hibernate.loader.MultipleBagFetchException
@Query("select o from Order o join fetch o.items join fetch o.payments")
```
Решения: заменить `List` на `Set`; сделать два отдельных запроса (второй попадёт в тот же persistence context и «дособерёт» объекты); использовать `@BatchSize` или `hibernate.default_batch_fetch_size` для второй коллекции.

2. **Удаление-и-переставка для однонаправленных `@OneToMany`.** Однонаправленная `@OneToMany` без `mappedBy` создаёт связующую таблицу, и удаление одного элемента приводит к DELETE всех строк связи с последующей повторной вставкой. Лечится переходом на **двунаправленную** связь.

**Про `equals`/`hashCode` для сущностей в `Set`** (спрашивают почти всегда):

```java
// Рекомендуемый вариант: equals по id с проверкой на null + КОНСТАНТНЫЙ hashCode
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Order other)) return false;
    return id != null && id.equals(other.getId());
}

@Override
public int hashCode() { return getClass().hashCode(); }
```
Почему так: до `persist` id равен `null`, и если `hashCode` зависит от id, объект «потеряется» в `HashSet` после присвоения идентификатора. Константный `hashCode` корректен (все элементы в одном бакете — для небольших коллекций это несущественно). Ещё лучше, если есть **бизнес-ключ** (email, артикул) — тогда `equals`/`hashCode` строятся по нему. Никогда не используйте Lombok `@Data`/`@EqualsAndHashCode` на сущностях: он включает все поля, инициализирует lazy-прокси и уходит в рекурсию на двунаправленных связях.

---

# Блок N. Наследование и составные типы

### N1. Стратегии наследования

**Ответ.**

| Стратегия | Схема | Плюсы | Минусы |
|---|---|---|---|
| `SINGLE_TABLE` *(по умолчанию)* | одна таблица + discriminator | самая быстрая, без JOIN | колонки подклассов должны быть nullable → нет NOT NULL-констрейнтов; таблица широкая |
| `JOINED` | таблица на класс, PK = FK | нормализовано, констрейнты работают | JOIN на каждый запрос, дороже вставка |
| `TABLE_PER_CLASS` | таблица на конкретный класс со всеми полями | нет JOIN при чтении конкретного типа | полиморфные запросы через `UNION ALL`, проблемы с генерацией id |

```java
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "type", discriminatorType = STRING)
abstract class Payment { @Id Long id; BigDecimal amount; }

@Entity @DiscriminatorValue("CARD")
class CardPayment extends Payment { String maskedPan; }
```

**`@MappedSuperclass`** — не сущность: даёт наследование полей и маппинга, но не участвует в полиморфных запросах и не имеет своей таблицы. Идеальна для базовых классов с `id`, `createdAt`, `version`.

```java
@MappedSuperclass
abstract class BaseEntity {
    @Id @GeneratedValue Long id;
    @Version Long version;
    @CreatedDate Instant createdAt;
}
```

Ответ на «что выбираете»: `SINGLE_TABLE` — если подклассы отличаются несколькими полями и важна скорость; `JOINED` — если у подклассов много собственных обязательных полей и нужна целостность на уровне БД; `TABLE_PER_CLASS` — практически никогда. И самый зрелый ответ: наследование сущностей часто вообще избыточно, зачастую лучше композиция или отдельные сущности.

---

### N2. `@Embeddable`, `@ElementCollection`, конвертеры

**Ответ.**

```java
@Embeddable
record Address(String city, String street, String zip) { }   // value object, без своего id

@Entity
class Customer {
    @Embedded private Address address;

    @Embedded
    @AttributeOverride(name = "city", column = @Column(name = "billing_city"))
    private Address billingAddress;                          // второй раз тот же тип

    @ElementCollection                                        // коллекция значений, не сущностей
    @CollectionTable(name = "customer_tags", joinColumns = @JoinColumn(name = "customer_id"))
    @Column(name = "tag")
    private Set<String> tags = new HashSet<>();
}
```

`@Embeddable` — часть сущности-владельца, отдельного жизненного цикла и id не имеет, хранится в тех же колонках. `@ElementCollection` — всегда LAZY, всегда полностью перезаписывается при изменении (Hibernate удаляет все строки и вставляет заново), поэтому для больших коллекций не годится.

**Конвертеры атрибутов:**

```java
@Converter(autoApply = true)
class MoneyConverter implements AttributeConverter<Money, BigDecimal> {
    public BigDecimal convertToDatabaseColumn(Money m) { return m.amount(); }
    public Money convertToEntityAttribute(BigDecimal v) { return new Money(v); }
}
```

Для JSON-колонок в Hibernate 6 есть встроенная поддержка:

```java
@JdbcTypeCode(SqlTypes.JSON)
private Map<String, Object> metadata;
```

Enum-ы: **всегда** `@Enumerated(EnumType.STRING)`. `ORDINAL` (значение по умолчанию!) хранит порядковый номер — вставка нового значения в середину enum молча портит все существующие данные. Это частый вопрос-ловушка.

---

# Блок O. Загрузка данных: lazy/eager, N+1, EntityGraph

### O1. Как реализована ленивая загрузка?

**Ответ.** Двумя разными механизмами:

1. **Для `@ManyToOne`/`@OneToOne`** — Hibernate подставляет **прокси**: сгенерированный подкласс сущности (Byte Buddy) с инициализированным только идентификатором. При обращении к любому полю, кроме id, прокси идёт в БД.
2. **Для коллекций** — подставляется собственная реализация (`PersistentBag`, `PersistentSet`, `PersistentList`), которая загружает данные при первом обращении.

```java
Order o = em.getReference(Order.class, 1L);   // прокси, SQL НЕ выполнен
o.getId();                                     // из прокси, SQL всё ещё нет
o.getStatus();                                 // ← вот здесь SELECT
```

**Практические следствия, которые надо назвать:**

- `entity.getClass()` для прокси вернёт сгенерированный класс вида `Order$HibernateProxy$xYz`, поэтому в `equals` нужен `instanceof`, а не `getClass() == ...`;
- `instanceof` тоже может подвести при наследовании: прокси создаётся на объявленный тип, и `payment instanceof CardPayment` может дать `false`. Обход — `Hibernate.unproxy(entity)`;
- проверить, инициализирован ли объект: `Hibernate.isInitialized(o)`; инициализировать вручную: `Hibernate.initialize(o)`;
- `em.getReference()` полезен, когда нужна только ссылка для установки FK — он позволяет не делать лишний SELECT:

```java
item.setOrder(em.getReference(Order.class, orderId));   // без загрузки заказа
```

- **bytecode enhancement** расширяет ленивость: позволяет лениво грузить отдельные базовые атрибуты (`@Basic(fetch = LAZY)`, например большой `byte[]`) и делает работающим lazy `@OneToOne` на необладающей стороне.

---

### O2. N+1: причины и все способы решения

**Ответ.** Один запрос за список + по запросу на ассоциацию каждого элемента.

```java
List<Order> orders = repo.findAll();          // 1 запрос
orders.forEach(o -> o.getItems().size());     // N запросов
```

Причины: LAZY-коллекция, к которой обращаются в цикле; **EAGER-ассоциация** (Hibernate часто не может собрать её в один JOIN и выполняет отдельный SELECT на каждую сущность); JPQL-запрос, который игнорирует EAGER-маппинг и догружает ассоциации отдельно.

**Способы решения:**

```java
// 1. JOIN FETCH — один запрос
@Query("select distinct o from Order o join fetch o.items where o.status = :s")
List<Order> findWithItems(@Param("s") Status s);

// 2. @EntityGraph — декларативно, работает с derived-методами и Pageable
@EntityGraph(attributePaths = {"items", "customer"})
List<Order> findByStatus(Status s);

// 3. @BatchSize — N запросов превращаются в N/size запросов с IN (...)
@OneToMany(mappedBy = "order")
@BatchSize(size = 50)
private List<Item> items;

// 4. FetchMode.SUBSELECT — коллекции всех загруженных родителей одним подзапросом
@OneToMany(mappedBy = "order")
@Fetch(FetchMode.SUBSELECT)
private List<Item> items;

// 5. DTO-проекция — ассоциации вообще не нужны
@Query("select new com.example.OrderDto(o.id, o.total) from Order o")
List<OrderDto> findDtos();
```

Глобально: `spring.jpa.properties.hibernate.default_batch_fetch_size=100` — простой способ существенно снизить ущерб от N+1 во всём приложении.

**Сравнение подходов в одной фразе (сильный ответ):** `join fetch` — когда точно знаешь, что нужно, и коллекция одна; `@EntityGraph` — то же самое, но декларативно и совместимо с пагинацией для `@ManyToOne`; `@BatchSize`/`SUBSELECT` — когда заранее неизвестно, понадобится ли ассоциация; DTO — когда сущность вообще не нужна (лучший вариант для чтения).

---

### O3. Почему `join fetch` ломает пагинацию?

**Ответ.** При `join fetch` коллекции результат SQL содержит по строке на **каждый элемент коллекции**, а не на каждую корневую сущность. `LIMIT/OFFSET` применяется к строкам SQL, а нужен — к корневым сущностям. Hibernate не может это совместить и **тянет весь результат в память**, выполняя пагинацию там:

```
WARN HHH000104: firstResult/maxResults specified with collection fetch; applying in memory
```

На большой таблице это OOM.

**Правильные обходы:**

```java
// Вариант 1: два запроса — сначала страница id, потом загрузка с fetch
@Query("select o.id from Order o where o.status = :s")
Page<Long> findIdsPage(@Param("s") Status s, Pageable p);

@Query("select distinct o from Order o join fetch o.items where o.id in :ids")
List<Order> findByIdsWithItems(@Param("ids") List<Long> ids);
```

```java
// Вариант 2: @EntityGraph только для @ManyToOne — с ними пагинация безопасна
@EntityGraph(attributePaths = "customer")
Page<Order> findByStatus(Status s, Pageable p);
```

```java
// Вариант 3: пагинация без fetch + @BatchSize на коллекции
```

Отдельно: при `join fetch` коллекции нужен `distinct` в JPQL, чтобы убрать дубликаты корневых сущностей. В Hibernate 6 дедупликация на уровне результата выполняется автоматически, а `hibernate.query.passDistinctThrough` больше не нужен — но `distinct` в запросе всё ещё влияет на генерируемый SQL, поэтому применять его надо осознанно.

---

### O4. `@EntityGraph`: fetch graph vs load graph

**Ответ.**

- **FETCH graph** (`jakarta.persistence.fetchgraph`) — атрибуты в графе грузятся EAGER, **все остальные — LAZY**, независимо от маппинга.
- **LOAD graph** (`jakarta.persistence.loadgraph`, значение по умолчанию в Spring Data) — атрибуты в графе грузятся EAGER, остальные — **согласно маппингу**.

```java
@Entity
@NamedEntityGraph(name = "Order.withItems",
    attributeNodes = @NamedAttributeNode(value = "items", subgraph = "items.product"),
    subgraphs = @NamedSubgraph(name = "items.product",
                               attributeNodes = @NamedAttributeNode("product")))
class Order { }

public interface OrderRepository extends JpaRepository<Order, Long> {
    @EntityGraph(value = "Order.withItems", type = EntityGraph.EntityGraphType.FETCH)
    Optional<Order> findById(Long id);
}
```

Практический смысл разницы: FETCH graph — способ «выключить» неудачно расставленные EAGER-маппинги для конкретного запроса.

---

# Блок P. Запросы: JPQL/HQL, Criteria, native, проекции

### P1. Какие способы писать запросы есть и когда что применять?

**Ответ.**

| Способ | Плюсы | Минусы |
|---|---|---|
| Derived-методы Spring Data | нулевой код | нечитаемы при 4+ условиях |
| JPQL/HQL (`@Query`) | переносимо, работает с сущностями | строка проверяется только при старте |
| Criteria API | типобезопасно, динамические фильтры | очень многословно |
| Spring Data `Specification` | Criteria с человеческим лицом, композиция | всё та же многословность |
| Native SQL | все возможности БД | не переносимо, нет автоматического маппинга |
| Querydsl / jOOQ | типобезопасно и читаемо | внешняя зависимость и кодогенерация |

```java
// JPQL
@Query("select o from Order o where o.status = :s and o.total > :t")
List<Order> find(@Param("s") Status s, @Param("t") BigDecimal t);

// Native
@Query(value = "select * from orders where total > :t", nativeQuery = true)
List<Order> findNative(@Param("t") BigDecimal t);

// Specification — динамические фильтры
public static Specification<Order> hasStatus(Status s) {
    return (root, q, cb) -> s == null ? null : cb.equal(root.get("status"), s);
}
repo.findAll(where(hasStatus(status)).and(createdAfter(from)), pageable);
```

Хороший ответ на «что выбираете»: derived-методы для простого, JPQL для основного, `Specification`/Querydsl для динамических фильтров, native SQL — для отчётов и тяжёлой аналитики, где ORM только мешает.

---

### P2. Проекции: как не тащить сущность целиком?

**Ответ.** Три механизма, о которых надо знать.

```java
// 1. Интерфейсная проекция (Spring Data) — выбираются только нужные колонки
public interface OrderSummary {
    Long getId();
    BigDecimal getTotal();
    @Value("#{target.customer.name}") String getCustomerName();   // open projection
}
List<OrderSummary> findByStatus(Status s);

// 2. Constructor expression в JPQL — классические DTO
@Query("select new com.example.OrderDto(o.id, o.total, c.name) from Order o join o.customer c")
List<OrderDto> findDtos();

// 3. Динамические проекции — один метод, разные представления
<T> List<T> findByStatus(Status s, Class<T> type);
```

**Важные нюансы:** «закрытая» интерфейсная проекция (только геттеры, без `@Value`) позволяет Hibernate выбрать **только нужные колонки**; «открытая» (с SpEL) вынуждает загрузить сущность целиком и тем самым теряет весь смысл. Для native-запросов проекции работают, но требуют совпадения имён колонок и алиасов.

Ключевая мысль для интервью: **для операций чтения сущность обычно не нужна.** DTO-проекция снимает разом dirty checking, lazy-загрузку, `LazyInitializationException` и лишние колонки. Сущности нужны там, где данные меняются.

---

### P3. Чем опасны native-запросы в связке с persistence context?

**Ответ.** Тремя вещами:

1. **Flush может не сработать.** Hibernate не парсит нативный SQL и не знает, какие таблицы он затрагивает, — несохранённые изменения могут не попасть в БД до запроса. Явное решение — `em.flush()` или `@Modifying(flushAutomatically = true)`; можно также указать затрагиваемые таблицы через `query.addSynchronizedEntityClass(...)`.
2. **Кэш первого уровня не обновляется.** Нативный UPDATE меняет БД, а объекты в контексте остаются со старыми значениями — и dirty checking может записать их обратно.
3. **Второй уровень кэша не инвалидируется** (для нативных запросов Hibernate инвалидирует регионы только при указании синхронизируемых таблиц).

Правило: после нативных модифицирующих запросов делайте `em.clear()` (или `clearAutomatically = true`) и не смешивайте их с работой с managed-сущностями в одной транзакции.

---

# Блок Q. Кэши: первый, второй уровень, кэш запросов

### Q1. Три уровня кэширования в Hibernate

**Ответ.**

| Кэш | Область | Включён | Что хранит |
|---|---|---|---|
| **Первого уровня** | `Session`/транзакция | всегда, отключить нельзя | managed-сущности |
| **Второго уровня** | `SessionFactory`, всё приложение | нужно включать явно | сущности, коллекции (по id) |
| **Кэш запросов** | `SessionFactory` | явно, требует L2 | id результатов запроса |

```properties
spring.jpa.properties.hibernate.cache.use_second_level_cache=true
spring.jpa.properties.hibernate.cache.use_query_cache=true
spring.jpa.properties.hibernate.cache.region.factory_class=jcache
spring.jpa.properties.hibernate.javax.cache.provider=org.ehcache.jsr107.EhcacheCachingProvider
spring.jpa.properties.hibernate.generate_statistics=true
```

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE, region = "orders")
class Order {
    @OneToMany(mappedBy = "order")
    @Cache(usage = CacheConcurrencyStrategy.READ_WRITE)   // коллекции кэшируются ОТДЕЛЬНО
    private List<Item> items;
}
```

**Стратегии конкурентности:**

| Стратегия | Когда |
|---|---|
| `READ_ONLY` | справочники, которые никогда не меняются; самая быстрая |
| `NONSTRICT_READ_WRITE` | редкие изменения, допустимо кратковременное устаревание |
| `READ_WRITE` | обычные изменяемые данные; использует «мягкие» блокировки |
| `TRANSACTIONAL` | только с JTA, полностью транзакционный кэш |

---

### Q2. Почему L2-кэш часто вреден и когда его стоит включать?

**Ответ (это и есть senior-ответ, а не просто описание настроек).**

Аргументы против:

- **кэш сущностей хранит не объекты, а «разобранное» состояние** (dehydrated state) — при чтении сущность собирается заново, экономия не такая большая, как ожидается;
- **коллекции кэшируются отдельно** и хранят только id элементов — без кэширования самих сущностей это даёт N дополнительных обращений;
- **кэш запросов хранит только идентификаторы** — при промахе по кэшу сущностей превращается в N+1;
- **любая модификация мимо Hibernate** (нативный SQL, миграции, другой сервис, работающий с той же БД) делает кэш неконсистентным;
- **в кластере** нужен распределённый или реплицированный кэш, что добавляет собственный класс проблем;
- кэш запросов инвалидируется целиком при любом изменении затронутых таблиц — на пишущей нагрузке его hit rate стремится к нулю.

Когда включать: редко меняющиеся справочники, конфигурации, словари; сценарии с явным преобладанием чтения; единственное приложение-владелец БД.

**Практичная альтернатива, которую стоит озвучить:** кэшировать не сущности, а **готовые DTO** через `@Cacheable` Spring Cache на сервисном слое — там понятны границы, TTL и момент инвалидации.

---

### Q3. Как проверить, что кэш вообще работает?

**Ответ.**

```java
Statistics stats = em.getEntityManagerFactory()
        .unwrap(SessionFactory.class).getStatistics();

stats.getSecondLevelCacheHitCount();
stats.getSecondLevelCacheMissCount();
stats.getQueryCacheHitCount();
stats.getEntityLoadCount();
stats.getPrepareStatementCount();       // сколько SQL реально ушло
```

Плюс `hibernate.generate_statistics=true` и логирование `org.hibernate.stat` на DEBUG. Отдельно полезно в тестах утверждать количество запросов — это единственный надёжный способ не пропустить N+1 в регрессии.

---

# Блок R. Производительность и пакетная обработка

### R1. Как правильно вставить 100 000 строк?

**Ответ.**

```properties
spring.jpa.properties.hibernate.jdbc.batch_size=50
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
spring.jpa.properties.hibernate.batch_versioned_data=true
# MySQL: обязательно rewriteBatchedStatements=true в JDBC URL
```

```java
@Transactional
public void importAll(List<Row> rows) {
    for (int i = 0; i < rows.size(); i++) {
        em.persist(map(rows.get(i)));
        if (i % 50 == 0) {
            em.flush();     // отправить батч
            em.clear();     // освободить persistence context
        }
    }
}
```

**Обязательные оговорки:**

1. `GenerationType.IDENTITY` **полностью отключает** batching для INSERT — нужен `SEQUENCE` с адекватным `allocationSize`.
2. Без `em.clear()` контекст растёт линейно: растёт и память, и стоимость dirty checking на каждом flush → `OutOfMemoryError`.
3. `order_inserts`/`order_updates` группируют операции по таблицам, чтобы батчи не разрывались.
4. Для по-настоящему больших объёмов Hibernate — не лучший инструмент: `JdbcTemplate.batchUpdate`, `StatelessSession`, `COPY` в PostgreSQL работают на порядок быстрее.
5. Не держите миллион строк в одной транзакции: чанки по N записей, каждый в своей транзакции, плюс идемпотентность и точка возобновления.

```java
// StatelessSession — без persistence context, кэша и dirty checking
try (StatelessSession ss = sessionFactory.openStatelessSession()) {
    Transaction tx = ss.beginTransaction();
    rows.forEach(r -> ss.insert(map(r)));
    tx.commit();
}
```

---

### R2. Топ-10 причин медленной работы Hibernate

**Ответ (готовый чек-лист — очень выигрышно звучит):**

1. **N+1** из-за LAZY в цикле или EAGER-маппинга.
2. **EAGER по умолчанию** на `@ManyToOne`/`@OneToOne`, никогда не переопределённый.
3. **Загрузка сущностей вместо DTO** для операций чтения.
4. **Огромный persistence context** — dirty checking на каждом flush.
5. **`IDENTITY`-генератор** вместо `SEQUENCE`, из-за чего не работает batching.
6. **Отсутствие пагинации** — `findAll()` на таблице с миллионами строк.
7. **`join fetch` + `Pageable`** → пагинация в памяти (`HHH000104`).
8. **OSIV** — соединение удерживается весь HTTP-запрос, скрытые N+1 в слое представления.
9. **Отсутствие индексов** под FK и колонки фильтрации — ORM их не создаёт автоматически (кроме случая `ddl-auto`).
10. **`@ElementCollection` и однонаправленные `@OneToMany`** — полная перезапись коллекции при любом изменении.

Плюс два бонусных: `@ManyToMany` с большими коллекциями и отсутствие `@BatchSize` там, где ассоциации нужны не всегда.

---

### R3. Как диагностировать проблемы?

**Ответ.**

```properties
# SQL с параметрами
spring.jpa.properties.hibernate.format_sql=true
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.orm.jdbc.bind=TRACE     # Hibernate 6 (в 5: org.hibernate.type.descriptor.sql)

# Статистика
spring.jpa.properties.hibernate.generate_statistics=true
logging.level.org.hibernate.stat=DEBUG

# Порог медленного запроса (мс)
spring.jpa.properties.hibernate.session.events.log.LOG_QUERIES_SLOWER_THAN_MS=100
```

Инструменты, которые стоит назвать: **datasource-proxy** или **p6spy** для подсчёта запросов и логирования с параметрами; `EXPLAIN ANALYZE` на стороне БД; `pg_stat_statements`; тесты, утверждающие количество SQL-запросов (например, через `Statistics.getPrepareStatementCount()`).

Фраза, которая хорошо звучит на интервью: «первое, что я делаю с незнакомым Hibernate-проектом, — включаю `generate_statistics` и смотрю количество запросов на один HTTP-вызов. Обычно этого достаточно, чтобы найти основную проблему».

---

# Блок S. Полезные возможности, о которых спрашивают

### S1. Оптимистичные и пессимистичные блокировки в Hibernate

Подробно разобрано в [блоке C](#блок-c-isolation-и-блокировки). Коротко, что относится именно к Hibernate:

```java
@Entity
class Account {
    @Version private Long version;      // long, Integer, Instant, Timestamp
}

// Форсированный инкремент версии родителя при изменении ребёнка — защита агрегата
em.lock(order, LockModeType.OPTIMISTIC_FORCE_INCREMENT);

// Пессимистичная
em.find(Account.class, id, LockModeType.PESSIMISTIC_WRITE);
```

Отдельная деталь: `@OptimisticLocking(type = OptimisticLockType.DIRTY | ALL)` — версионирование без колонки `@version`, через сравнение значений в `WHERE`. Работает только в пределах одной сессии и почти всегда хуже честного `@Version`, но знать о нём полезно.

---

### S2. `@NaturalId`, `@Immutable`, `@Formula`, `@SQLRestriction`, `@Filter`

**Ответ.**

```java
@Entity
class Product {
    @Id Long id;

    @NaturalId                    // бизнес-ключ; даёт отдельный кэшируемый способ поиска
    private String sku;

    @Formula("(select avg(r.score) from review r where r.product_id = id)")
    private Double avgScore;      // вычисляемое поле, read-only

    @OneToMany(mappedBy = "product")
    @SQLRestriction("deleted = false")   // Hibernate 6.3+; раньше @Where
    private List<Review> reviews;
}

// Поиск по natural id — с собственным кэшем
session.byNaturalId(Product.class).using("sku", "ABC-1").load();

@Entity @Immutable
class Country { }                 // read-only сущность: не отслеживается dirty checking

// Динамический фильтр — включается на уровне сессии
@FilterDef(name = "tenantFilter", parameters = @ParamDef(name = "tenantId", type = Long.class))
@Filter(name = "tenantFilter", condition = "tenant_id = :tenantId")
@Entity class Document { }

session.enableFilter("tenantFilter").setParameter("tenantId", currentTenant());
```

Разница `@SQLRestriction` (`@Where`) и `@Filter`: первый — статическое условие, применяется всегда; второй — динамический, включается программно с параметрами. `@Filter` — стандартный способ реализации мультитенантности на уровне строк.

Мягкое удаление в Hibernate 6.4+ — встроенная аннотация `@SoftDelete`.

---

### S3. Что изменилось в Hibernate 6 (и почему об этом спрашивают)

**Ответ.** Полезно знать, если проект мигрировал или мигрирует:

- **`jakarta.persistence`** вместо `javax.persistence` — самое заметное изменение (Spring Boot 3).
- **Новый движок запросов SQM** (Semantic Query Model) вместо старого транслятора: лучше оптимизация, строже проверка JPQL — часть валидных ранее запросов перестала компилироваться.
- **Переработана система типов.** `@Type(type = "...")` со строковыми именами удалён; вместо него `@JdbcTypeCode`, `@JavaType`, `AttributeConverter`. Для JSON — `@JdbcTypeCode(SqlTypes.JSON)` из коробки.
- **Исчез глобальный `hibernate_sequence`** — каждая сущность по умолчанию получает свою последовательность `<таблица>_SEQ`. Частая причина поломки миграции.
- **Автоматическая дедупликация** результатов при fetch join — `distinct` в JPQL больше не нужен для этой цели.
- **`Instant`/`LocalDateTime`** и прочий `java.time` поддерживаются нативно.
- Улучшены batching, чтение по массивам и поддержка оконных функций в HQL.

Если не работали с 6 — скажите об этом прямо, но назовите пару пунктов: это показывает, что вы следите за экосистемой.

---

# Блок T. Исключения и диагностика

### T1. Разберите основные исключения Hibernate

**Ответ — таблица, которую полезно знать наизусть:**

| Исключение | Причина | Решение |
|---|---|---|
| `LazyInitializationException` | обращение к lazy-полю вне сессии | fetch join / `@EntityGraph` / DTO |
| `MultipleBagFetchException` | два `join fetch` на bag-коллекции | `Set`, два запроса, `@BatchSize` |
| `NonUniqueObjectException` | в сессии уже есть другой объект с тем же id | `merge()` вместо `update()` |
| `TransientObjectException` / `TransientPropertyValueException` | ссылка на несохранённую сущность | сохранить связанную сущность или добавить `CascadeType.PERSIST` |
| `StaleObjectStateException` / `ObjectOptimisticLockingFailureException` | конфликт версий | ретрай всей транзакции снаружи |
| `ConstraintViolationException` (Hibernate) | нарушение констрейнта БД | проверка до записи; не продолжать в той же транзакции |
| `DataIntegrityViolationException` (Spring) | обёртка над предыдущим | то же |
| `EntityNotFoundException` | прокси указывает на удалённую строку | `em.find` вместо `getReference` |
| `QuerySyntaxException` / `SemanticException` | ошибка в JPQL | проверяется при старте контекста |
| `ObjectDeletedException` | изменение удалённой сущности | убрать ссылки на неё |
| `HHH000104` (warning) | пагинация с fetch коллекции | двухзапросный подход |
| `UnexpectedRollbackException` | транзакция помечена rollback-only | см. [блок B3](#b3-что-такое-rollbackonly-и-откуда-берётся-unexpectedrollbackexception) |

---

### T2. Как связаны `ddl-auto`, миграции и прод?

**Ответ.**

| `spring.jpa.hibernate.ddl-auto` | Действие |
|---|---|
| `none` | ничего (**единственно верное значение для прода**) |
| `validate` | проверить соответствие схемы маппингу и упасть при расхождении |
| `update` | достроить недостающее (не удаляет и не меняет типы) |
| `create` | пересоздать при старте |
| `create-drop` | пересоздать и удалить при остановке |

Правильный ответ на собеседовании: схемой управляет **Flyway или Liquibase**, а Hibernate — `validate` (или `none`). Почему не `update`: он не удаляет колонки, не меняет типы, не версионируется, ведёт себя по-разному между версиями Hibernate и не даёт отката. По сути это «схема как побочный эффект кода», что на проде неприемлемо.

Полезная связка: `validate` в тестах и на проде + Flyway для изменений. Тогда несоответствие маппинга и схемы падает на старте, а не в рантайме.

---

## Часть 6. Задачи «что произойдёт» (Hibernate)

### Задача 13

```java
@Transactional
public void test() {
    Order o = em.find(Order.class, 1L);
    o.setStatus(Status.PAID);
    em.detach(o);
}
```

**Вопрос:** сохранится ли изменение?

<details><summary>Ответ</summary>

**Нет.** `detach()` отсоединяет объект до flush, изменения теряются. Если бы между `setStatus` и `detach` был `em.flush()` — UPDATE ушёл бы в БД и закоммитился.
</details>

---

### Задача 14

```java
@Entity
class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;
}
// hibernate.jdbc.batch_size=50
@Transactional
public void insert1000() {
    for (int i = 0; i < 1000; i++) em.persist(new Order());
}
```

**Вопрос:** сколько будет INSERT-запросов?

<details><summary>Ответ</summary>

**1000 отдельных INSERT.** `IDENTITY` требует получить id немедленно, поэтому Hibernate выполняет INSERT сразу на `persist()` и не может собрать батч — настройка `batch_size` просто игнорируется.

С `GenerationType.SEQUENCE` и `allocationSize = 50` было бы 20 батчей по 50 плюс ~20 обращений к последовательности.
</details>

---

### Задача 15

```java
@Transactional
public void test() {
    Order o = new Order("A");
    em.persist(o);
    List<Order> all = em.createQuery("select o from Order o", Order.class).getResultList();
    System.out.println(all.size());
}
```

**Вопрос:** попадёт ли новый заказ в результат (в БД он ещё не закоммичен)?

<details><summary>Ответ</summary>

**Да.** `FlushMode.AUTO` вызывает flush перед JPQL-запросом, затрагивающим таблицу `Order`, — INSERT уходит в БД внутри той же транзакции, и запрос его видит.

А вот при `FlushMode.COMMIT` или при **нативном** запросе (Hibernate не парсит его и не понимает, какие таблицы затронуты) — новый заказ мог бы не попасть в выборку.
</details>

---

### Задача 16

```java
@Entity
class Order {
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    List<Item> items = new ArrayList<>();
}

@Transactional
public void test(Long id) {
    Order o = repo.findById(id).orElseThrow();
    o.getItems().remove(0);
}
```

**Вопрос:** удалится ли позиция из БД?

<details><summary>Ответ</summary>

**Нет.** `CascadeType.ALL` распространяет `remove` только при удалении **родителя**. Удаление элемента из коллекции — это «сирота», и для его удаления нужен `orphanRemoval = true`.

Что произойдёт на самом деле: Hibernate попытается обнулить FK в строке `item` (`UPDATE item SET order_id = null`). Если колонка `NOT NULL` — ошибка констрейнта. Плюс нужно синхронизировать обратную сторону: `item.setOrder(null)`.
</details>

---

### Задача 17

```java
Set<Item> items = new HashSet<>();
Item item = new Item("A");     // equals/hashCode по id
items.add(item);
em.persist(item);              // здесь назначается id
System.out.println(items.contains(item));
```

**Вопрос:** что напечатается?

<details><summary>Ответ</summary>

**`false`** — если `hashCode()` зависит от id. При добавлении id был `null`, объект попал в бакет по хэшу от `null`; после `persist` id присвоен, хэш изменился, и `contains` ищет в другом бакете.

Это и есть причина рекомендации: константный `hashCode()` (`getClass().hashCode()`) плюс `equals` по id с проверкой на `null`, либо бизнес-ключ, либо назначение UUID в конструкторе.
</details>

---

### Задача 18

```java
@Query("select distinct o from Order o join fetch o.items")
Page<Order> findAllWithItems(Pageable pageable);
```

**Вопрос:** что произойдёт при вызове с `PageRequest.of(0, 20)` на таблице в миллион заказов?

<details><summary>Ответ</summary>

Hibernate выведет `WARN HHH000104: firstResult/maxResults specified with collection fetch; applying in memory` и **загрузит все заказы со всеми позициями в память**, после чего отрежет первые 20. На миллионе строк — `OutOfMemoryError` или многоминутный запрос.

Плюс отдельная проблема: `Page` требует count-запроса, а с `join fetch` он тоже сгенерируется некорректно (Spring Data потребует явный `countQuery`).

Решение: страница id одним запросом → загрузка с `join fetch` по `in :ids` вторым.
</details>

---

### Задача 19

```java
@Transactional
public void test(Long id) {
    Order o = em.find(Order.class, id);
    em.createQuery("update Order o set o.status = 'PAID' where o.id = :id")
      .setParameter("id", id).executeUpdate();
    System.out.println(o.getStatus());
}
```

**Вопрос:** что напечатается и что окажется в БД?

<details><summary>Ответ</summary>

Напечатается **старый статус** — bulk-запрос идёт мимо persistence context, объект в кэше первого уровня не обновляется.

В БД в итоге может оказаться тоже **старый статус**: если `o` был изменён в этой же транзакции, dirty checking на коммите сгенерирует UPDATE и затрёт результат bulk-запроса. Решение — `em.clear()` после bulk-операции или `@Modifying(clearAutomatically = true, flushAutomatically = true)`.
</details>

---

### Задача 20

```java
@Transactional
public void test() {
    em.remove(em.find(Tag.class, 1L));        // уникальный ключ name = "java"
    em.persist(new Tag("java"));
}
```

**Вопрос:** пройдёт ли операция?

<details><summary>Ответ</summary>

**Нет — нарушение unique-констрейнта.** Порядок операций в `ActionQueue` фиксирован: INSERT выполняются **раньше** DELETE. Hibernate попытается вставить новый `Tag("java")`, пока старая строка ещё существует.

Решение — явный flush между операциями:

```java
em.remove(em.find(Tag.class, 1L));
em.flush();                          // DELETE уходит в БД сейчас
em.persist(new Tag("java"));
```
Либо сделать констрейнт `DEFERRABLE INITIALLY DEFERRED` (PostgreSQL), либо просто обновить существующую строку вместо пересоздания.
</details>

---

## Часть 7. Шпаргалка по Hibernate

### Одна фраза, объясняющая почти всё

Hibernate — это **менеджер состояния объектов в памяти**, который синхронизирует их с БД на flush. Два вопроса, из которых выводится любой ответ: **в каком состоянии объект** и **когда полетит SQL**.

### Грабли, которые надо называть самому

| Тема | Грабля |
|---|---|
| Состояния | `detached` не отслеживается; `merge` возвращает копию, аргумент остаётся detached |
| Flush | DELETE выполняется последним → «удалить и вставить с тем же ключом» падает |
| Flush | нативный запрос не вызывает автоматический flush и не видит изменений |
| `IDENTITY` | полностью отключает batching для INSERT |
| `SEQUENCE` | `allocationSize` в коде должен совпадать с `INCREMENT BY` в БД |
| FetchType | `@ManyToOne`/`@OneToOne` по умолчанию **EAGER** — переопределять явно |
| lazy `@OneToOne` | не работает на необладающей стороне без bytecode enhancement |
| Каскады | `CascadeType.REMOVE` ≠ `orphanRemoval`; каскады только для композиции |
| Двунаправленные связи | надо синхронизировать обе стороны вручную |
| Коллекции | `MultipleBagFetchException`; однонаправленная `@OneToMany` перезаписывает связи целиком |
| `@ElementCollection` | полная перезапись при любом изменении |
| `equals`/`hashCode` | id-зависимый `hashCode` ломает `HashSet`; Lombok `@Data` на сущностях запрещён |
| Enum | `@Enumerated` по умолчанию `ORDINAL` — всегда ставить `STRING` |
| Пагинация | `join fetch` коллекции + `Pageable` → выборка в память (`HHH000104`) |
| Bulk-запросы | мимо persistence context, без каскадов и коллбэков, не трогают `@Version` |
| L2-кэш | коллекции и кэш запросов хранят только id; любое изменение мимо Hibernate ломает консистентность |
| Прокси | `getClass()` возвращает сгенерированный подкласс; `instanceof` подводит при наследовании |
| Batch | без `em.clear()` контекст растёт → OOM и медленный dirty checking |
| `ddl-auto` | `update` на проде недопустим — только Flyway/Liquibase + `validate` |
| OSIV | включён по умолчанию в Boot; выключать |

### Настройки, которые полезно помнить

```properties
spring.jpa.open-in-view=false
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.properties.hibernate.jdbc.batch_size=50
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
spring.jpa.properties.hibernate.default_batch_fetch_size=100
spring.jpa.properties.hibernate.generate_statistics=true
spring.jpa.properties.hibernate.connection.handling_mode=DELAYED_ACQUISITION_AND_RELEASE_AFTER_TRANSACTION
spring.jpa.properties.hibernate.session.events.log.LOG_QUERIES_SLOWER_THAN_MS=100

logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.orm.jdbc.bind=TRACE
logging.level.org.hibernate.stat=DEBUG
```

### Имена, которые стоит произнести вслух хотя бы раз

`SessionFactory`, `Session`/`EntityManager`, persistence context, `ActionQueue`, dirty checking, snapshot, `FlushMode.AUTO/MANUAL`, `PersistentBag`, `HibernateProxy`, `Hibernate.initialize()`, `em.getReference()`, `@BatchSize`, `FetchMode.SUBSELECT`, `@EntityGraph` (fetch vs load), `@NaturalId`, `@DynamicUpdate`, `StatelessSession`, `Statistics`, `MultipleBagFetchException`, `HHH000104`, SQM (Hibernate 6).

### Три вопроса, которые задают почти всегда

1. **«Почему сущность сохранилась без вызова `save()`?»** → managed-состояние + dirty checking на flush перед коммитом.
2. **«Что такое N+1 и как чинить?»** → четыре способа с границами применимости (fetch join, `@EntityGraph`, `@BatchSize`/SUBSELECT, DTO).
3. **«Почему `LazyInitializationException` и почему OSIV — плохое решение?»** → границы persistence context + удержание соединения и скрытые N+1 в слое представления.


