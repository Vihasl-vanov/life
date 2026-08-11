## **Ключевые принципы успешного HR-собеседования**

1. **Энергия и позитив.** Уверенная, живая речь, улыбка и зрительный контакт. Цель — показать себя приятным и открытым человеком.
    
2. **Вовлеченность.** Внимательно слушайте, смотрите в камеру, создавайте ощущение диалога и искренней заинтересованности.
    
3. **Будьте собой.** Говорите честно, без приукрашиваний и попыток казаться тем, кем вы не являетесь.
    
4. **Хвалите себя.** Ярко и вдохновенно рассказывайте о своих достижениях и сильных сторонах. Акцент на успехах, а не на неудачах.
    
5. **Подготовьте самопрезентацию.** Заранее продумайте краткий (2-3 мин.), структурированный и энергичный рассказ о своем опыте и главных достижениях.
    
6. **Задавайте правильные вопросы.**
    
    - **Обязательный:** Какими качествами должен обладать кандидат, чтобы через год вас убедить, что hiring был великолепным решением?
        
    - **Уточняющий:** Какие следующие этапы собеседования?
        
7. **Про деньги.** Если спросят про ожидания, вежливо уклонитесь от цифры, сфокусировавшись на интересе к компании и своём потенциале. Не проявляйте эмоций на озвученный диапазон.
    
8. **Завершение.** Вежливо попрощайтесь, выразите интерес к дальнейшим этапам и попросите прислать план следующих шагов.
    
9. **Делайте заметки.** Пишите ручкой на бумаге (это видно), чтобы продемонстрировать внимательность и серьезный подход. Не забывайте при этом поддерживать контакт.

## Tech Interview

1. **Не зацикливайтесь на ошибках.** Одна ошибка или незнание — не приговор. Сразу забывайте и концентрируйтесь на следующем вопросе. Собеседование длится до конца.
    
2. **Проговаривайте ход мыслей.** Интервьюеру важнее увидеть ваш мыслительный процесс, чем услышать идеальный ответ. Комментируйте вслух каждое свое решение, объясняйте логику. Говорите больше интервьюера.
    
3. **Говорите прямо.** Если не знаете ответа, сразу честно скажите: «Не сталкивался с этим/не знаю». Не тратьте время на притворные размышления.

## tell me about your experience(recruters)
"I'm a backend developer with 5+ years building high-load systems for financial services. For the past several years I've worked at SberTech on SberProfile, the centralized financial profile platform used by millions of clients — my job is to keep it fast and reliable even as the load keeps growing.

A few things I'm proud of: I led an optimization effort that cut a key API's response time by 62%, and I reworked our deployment pipeline to bring container startup down from 47 seconds to 7. I also helped lead our move from a monolith to microservices, which made it much easier for different teams to ship independently. Beyond the hands-on work, I've mentored two junior engineers to full independence within their first month, and introduced a code-review checklist that cut defects reaching QA by 15%.

Before SberTech, I interned at T-Bank building REST APIs for their personal finance product, where I shipped a transaction categorization feature that ended up used by 80% of active users.

I'm looking for a role where I can keep working on systems at real scale, ideally with more ownership over architecture decisions."

## For technical specialists

"My background is Java/Spring Boot backend development, focused on high-load, fault-tolerant systems — specifically SberProfile, a centralized financial profile platform at SberTech serving millions of clients with sub-50ms latency requirements on critical paths.

The core of my work has been event-driven architecture: I built out Kafka integration across 10+ services and implemented Transactional Outbox and Retry patterns to get guaranteed delivery and eventual consistency. On the data side, I designed a DB sharding scheme for our most critical data to get linear scalability and fault isolation as load grew — that included the schema design, the routing logic, and handling cross-shard query patterns.

I spend a lot of time in query optimization — EXPLAIN ANALYZE, indexing strategy, normalization trade-offs — which is how I got a key API's p99 down from 80ms to 30ms while also reducing DB CPU load, not just shifting the bottleneck. I also led part of our monolith decomposition: defining service boundaries and API contracts based on actual data ownership rather than just organizational lines, which is where a lot of the long-term payoff came from.

On infra, I reworked our Docker images and OpenShift build pipeline — multi-stage builds, layer caching, JVM startup tuning — to cut cold-start from 47s to 7s, which mattered a lot for our deploy frequency and incident recovery time.

I use JUnit 5 and Mockito for testing, follow DDD principles for domain modeling, and I've been doing regular code reviews and mentoring alongside the IC work — currently onboarded two juniors to independent contribution within a month each."

## Conflict story

"During the monolith-to-microservices decomposition at SberTech, one of the harder parts was defining service boundaries — and not everyone agreed on where the lines should be. Some engineers wanted to split services along existing team structure, since that would be faster to migrate. I pushed back, because I thought that would just recreate the monolith's coupling problems in a distributed form — we'd get the deployment overhead of microservices without the actual autonomy benefit.

I made the case using our actual data ownership patterns — which services read and wrote which data — rather than who currently owned the code, and proposed drawing boundaries around that instead. It meant more upfront design work and a slower initial migration for a couple of services. We went with the data-ownership approach, and it's held up — team autonomy and delivery velocity improved afterward, which is what convinced the skeptics it was worth the extra design time up front."

## Failure story

"When I was building out the DB sharding system for our critical data, I underestimated how much complexity cross-shard queries would add. I designed the shard key and routing logic focused mainly on write scalability and fault isolation, which was the primary goal — but I hadn't fully accounted for how often certain read queries would need to span multiple shards, and those ended up slower than the single-DB queries they replaced, at least initially.

I had to go back and add a secondary indexing/routing layer specifically for those cross-shard read patterns, which took extra time I hadn't planned for. Since then, when I design for scalability I map out the actual query patterns — not just the write path — before committing to a partitioning scheme. It's a step I don't skip anymore, and it's saved me from repeating that mistake on later data model decisions."

## Achievements story

"One I'm most proud of: our key API on SberProfile had a response time of about 80ms, which was becoming a real constraint as load grew. I dug into it with EXPLAIN ANALYZE, restructured indexing, and cleaned up some normalization issues that were forcing expensive joins — brought it down to 30ms, a 62% improvement, and reduced DB CPU load in the process rather than just moving the bottleneck elsewhere.

Around the same time, I reworked our Docker images and OpenShift build pipeline — multi-stage builds, better layer caching, JVM startup tuning — and cut cold-start from 47 seconds to 7. That mattered beyond just deploy speed: it directly improved our incident recovery time.

I'm also proud of the DB sharding system I designed for our most critical data — that gave us linear scalability and fault isolation, which was a genuine architectural bet that paid off as load kept growing. And on the people side: I mentored two junior engineers to independent contribution within their first month each, and introduced a code-review checklist that cut defects reaching QA by 15% — so it's not just individual wins, it's things that made the team better."

## Ambitions and goals

"Right now my focus is deepening my expertise in distributed systems — I've spent the last few years building the practical side of this at SberTech (Kafka, sharding, Outbox/Saga patterns), and I want to go further into the theory and edge cases: consensus, consistency guarantees, failure modes at scale.

Medium-term, I want more ownership over architecture decisions rather than just implementing them — moving from 'here's how I built this' to 'here's why we should build it this way.' I've had a taste of that leading part of our monolith decomposition and defining service boundaries, and I want more of it.

Longer-term, I'm interested in either a staff/principal-track role focused on systems design, or a lead role where mentoring becomes a bigger part of the job — I already mentor junior engineers and enjoy that side of the work."

## why left previous job
"I've grown a lot at SberTech — led the sharding design, the Kafka event-driven integration, part of our monolith decomposition. But I've hit the limit on how much architectural ownership my current scope gives me. I'm looking for a role with bigger scale and harder distributed-systems problems, where I'm shaping the 'why,' not just executing the 'how.'"

## Weakness

"I tend to go too deep on technical correctness before confirming it's the highest-priority part of the problem — on the sharding project, I nailed write-path scalability but under-invested in mapping cross-shard read patterns early, which cost me rework later. I've since made mapping all usage patterns a deliberate step before committing to a design — it's a habit I'm still actively building, not something I've fully automated yet."

## Strengths

"Two things I'd call real strengths. First, performance and reliability at scale — I don't just make things work, I dig into why they're slow or fragile. Cutting a key API's response time by 62% through query optimization, or cutting cold-start 6× through pipeline rework — those came from actually profiling and understanding the bottleneck, not guessing.

Second, I'm good at architectural trade-offs under real constraints — the sharding design, the monolith decomposition, defining service boundaries around actual data ownership rather than convenience. I don't just implement the first design that comes to mind; I think through what breaks it at 10x load.

I'd also add that I invest in the team, not just the code — mentoring two juniors to independence within a month each, building a code-review checklist that cut defects by 15%. I care about the system staying maintainable after I've moved on to the next thing."