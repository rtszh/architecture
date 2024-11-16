#### 1. Текущая архитектура
1. с точки зрения нагрузки:
   1. *core-app* делает запрос к *ins-product-aggregator* один раз в 15 минут;
   2. *ins-comp-settlement* делает запрос к *ins-product-aggregator* один раз в сутки;
   3. про стратегию *retry*'ев в задании не указано, но судя по всему *core-app* и *ins-comp-settlement* в конечном счете все равно хотят получить свои данные;
   4. то есть, если внутри *ins-product-aggregator* произошла ошибка, то данные перезапрашиваются. А если ошибка на стороне внешнего *API*, то данные могут перезапрашиваться достаточно большое количество раз (в зависимости от количества *retry*'ев);
   5. в любом случае, такой подход ведет к увеличению нагрузки на систему;
2. ошибки при взаимодействии *ins-product-aggregator* с внешними *api*:
   1. это является проблемой, которая (судя по описанию задания) реально увеличивает нагрузку на систему;

#### 2. Предложения по изменению архитектуры
1. сервисам *core-app* и *ins-comp-settlement* необязательно делать синхронные запросы в *ins-product-aggregator* и ждать от него ответа;
2. также, сервису *ins-comp-settlement* необязательно делать синхронные запрос в *core-app*;
3. вышеописанные синхронные запросы можно изменить на **события**:
   1. *ins-product-aggregator* использует для взаимодействия с внешними *API*: *REST*;
   2. внешние *API* (**системы страховых компнаний**), в которых *ins-product-aggregator* запрашивает информацию предоставляет синхронное *API* (*REST/GraphQL/SOAP*);
4. предлагается следующее *Flow*:
   1. *ins-product-aggregator* забирает информацию по продуктам из внешних *API*;
   2. далее, *ins-product-aggregator*, получив нужную информацию, публикует ее в *kafka* в виде **события**;
   3. сервисы *core-app* и *ins-comp-settlement* подписаны на **топик** с этими событиями, они считывают это событие и обрабатывают каждый по-своему;
5. аналогичный подход предлагается и для взаимдойствия между *core-app* и *ins-comp-settlement*;

- итог:
1. **событие 1**:
   - *ins-product-aggregator* - публикует **событие** с информацией из страховых компаний, а *core-app* и *ins-comp-settlement* подписаны на это **событие**;
2. **событие 2**:
   - *core-app* публикует **событие**, которое содержит все оформленные за день страховки, а *ins-comp-settlement* подписан на это **событие**;

#### 3. Технические детали реализации
- при публикации **событий** будет применяться паттерн *Transactional Outbox* - с целью повышения надежности системы и консистентности данных;
- для реализации *Transactional outbox* используем *Transaction log tailing*:
   1. используем в БД техническую таблицу *outbox* для хранения отправляемых событий событий:
      1. для *ins-product-aggregator* необходимо поднять БД;
      2. для *core-app* создадим таблицу в существующей БД;
   2. за сохранением данных в таблицу будет следить отдельный компонент, который будет отслеживать изменения в журнале транзакций;
   3. при появлении соответвующего изменения в журнале транзакций этот компонент будет публиковать **событие в kafka**;