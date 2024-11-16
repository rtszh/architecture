#### 1. Принципиальный процесс получения предложений по заявке на ОСАГО
1. пользователь заполняет заявку на ОСАГО и отправляет запрос;
2. запрос обрабатывается в core-app:
   1. этот запрос сохраняется в кеш вместе с информацией об устройстве, которое выполнило запрос;
   2. шедулер приходит в кеш и видит такую новую заявку. Заявка находит в статусе "Необработана";
   3. шедулер блокирует запись в кеше (чтобы другие инстансы core-app ее не обрабатывали), отправляет ее в RabbitMQ и меняет ей статус: "В обработке";
3. запрос обрабатывается в osago-aggregator:
   1. запись считывается из rabbitMQ и в соответствии с ней выполняется запрос во внешние интеграции;
   2. в ответ внешние интеграции присылают предложения по заявке на ОСАГО;
   3. предложения по заявке на ОСАГО сохраняются в outbox-таблицу;
   4. debezium считывает данные из outbox-таблицы и пушит их в RabbitMQ;
   5. если внутри osago-aggregator не удалось уложиться в требуемые 60 сек, то ничего не нужно отправлять в RabbitMQ;
4. запрос обрабатывается в core-app:
   1. core-app видит, что пришел ответ с предложениями по одной из заявок на ОСАГО;
   2. core-app обращается к кешу, чтобы сопоставить какой именно заявке пришел ответ;
   3. core-app удаляет из кеша запись и отдает ответ на клиент, после чего разрывает websocket;
   4. если не удалось получить ответ по заявке за 60 сек, то core-app сам очищает таблицу с кешем и отвечает клиенту, что заявку получить не удалось. Это также можно реализовать с помощью шедулера;


#### 2. Принципиальный процесс оформления заявки по полученному предложению
1. когда клиент получил предложения по своей заявке на ОСАГО, то дальнейший процесс обработки заявки работает также, как и для других страховок - через core-app;