# Автор: Федорчук Дмитрий Сергеевич DEVOPS-33

# Домашнее задание к занятию 13 «Введение в мониторинг»

## Обязательные задания

1. Вас пригласили настроить мониторинг на проект. На онбординге вам рассказали, что проект представляет из себя платформу для вычислений с выдачей текстовых отчетов, которые сохраняются на диск. Взаимодействие с платформой осуществляется по протоколу http. Также вам отметили, что вычисления загружают ЦПУ. Какой минимальный набор метрик вы выведите в мониторинг и почему?
#
2. Менеджер продукта посмотрев на ваши метрики сказал, что ему непонятно что такое RAM/inodes/CPUla. Также он сказал, что хочет понимать, насколько мы выполняем свои обязанности перед клиентами и какое качество обслуживания. Что вы 
можете ему предложить?
#
3. Вашей DevOps команде в этом году не выделили финансирование на построение системы сбора логов. Разработчики в свою очередь хотят видеть все ошибки, которые выдают их приложения. Какое решение вы можете предпринять в этой ситуации, 
чтобы разработчики получали ошибки приложения?
#
4. Вы, как опытный SRE, сделали мониторинг, куда вывели отображения выполнения SLA=99% по http кодам ответов. Вычисляете этот параметр по следующей формуле: summ_2xx_requests/summ_all_requests. Данный параметр не поднимается выше 
70%, но при этом в вашей системе нет кодов ответа 5xx и 4xx. Где у вас ошибка?
#
5. Опишите основные плюсы и минусы pull и push систем мониторинга.
#
6. Какие из ниже перечисленных систем относятся к push модели, а какие к pull? А может есть гибридные?

    - Prometheus 
    - TICK
    - Zabbix
    - VictoriaMetrics
    - Nagios
#
7. Склонируйте себе [репозиторий](https://github.com/influxdata/sandbox/tree/master) и запустите TICK-стэк, 
используя технологии docker и docker-compose.

В виде решения на это упражнение приведите скриншот веб-интерфейса ПО chronograf (`http://localhost:8888`). 

P.S.: если при запуске некоторые контейнеры будут падать с ошибкой - проставьте им режим `Z`, например
`./data:/var/lib:Z`
#
8. Перейдите в веб-интерфейс Chronograf (http://localhost:8888) и откройте вкладку Data explorer.
        
    - Нажмите на кнопку Add a query
    - Изучите вывод интерфейса и выберите БД telegraf.autogen
    - В `measurments` выберите cpu->host->telegraf-getting-started, а в `fields` выберите usage_system. Внизу появится график утилизации cpu.
    - Вверху вы можете увидеть запрос, аналогичный SQL-синтаксису. Поэкспериментируйте с запросом, попробуйте изменить группировку и интервал наблюдений.

Для выполнения задания приведите скриншот с отображением метрик утилизации cpu из веб-интерфейса.
#
9. Изучите список [telegraf inputs](https://github.com/influxdata/telegraf/tree/master/plugins/inputs). 
Добавьте в конфигурацию telegraf следующий плагин - [docker](https://github.com/influxdata/telegraf/tree/master/plugins/inputs/docker):
```
[[inputs.docker]]
  endpoint = "unix:///var/run/docker.sock"
```

Дополнительно вам может потребоваться донастройка контейнера telegraf в `docker-compose.yml` дополнительного volume и 
режима privileged:
```
  telegraf:
    image: telegraf:1.4.0
    privileged: true
    volumes:
      - ./etc/telegraf.conf:/etc/telegraf/telegraf.conf:Z
      - /var/run/docker.sock:/var/run/docker.sock:Z
    links:
      - influxdb
    ports:
      - "8092:8092/udp"
      - "8094:8094"
      - "8125:8125/udp"
```

После настройке перезапустите telegraf, обновите веб интерфейс и приведите скриншотом список `measurments` в 
веб-интерфейсе базы telegraf.autogen . Там должны появиться метрики, связанные с docker.

Факультативно можете изучить какие метрики собирает telegraf после выполнения данного задания.

## Решение обязательного задания

1. В первую очередь я бы вывел отображение следующих метрик:

* CPU load average - средние значения загрузки процессора, выводится за 1, 5, и 15 минут. Даст представление величинах загрузки процессора. Далее можно будет отслеживать, какие процессы нагружают процессор сервера.

* Memory Info - Отображение информации об оперативной памяти: общее количество оперативной памяти, количество использованной оперативной памяти, количество свободной оперативной памяти; количество памяти, используемое swap файлом. По отображению этих данных можно будет искать процессы, сильно потребляющие оперативную память. А также по величине использования swap раздела можно будет сделать вывод о том, сколько оперативной памяти не достаточно для работы проекта. Большое использование swap раздела способно сильно замедлить работу проекта.

* Disk info - Отображение информации о диске: отображение свободного места на диске, для предотвращения его полного расходования; отображение количества свободных индексных дескрипторов (inodes), для возможности продолжения создания новых файлов на диске; Disk IOps для отображения производительности жесткого диска, то есть отображение числа операций чтения и записи в единицу времени.
Эти данные важны, т.к. не редко дисковая подсистема может являться "бутылочным горлышком" в производительности системы, особенно при использовании HDD дисков, а не SSD. Также процессор сервера может быть нагружен из-за недостаточной производительности дисковой подсистемы, потому, что процессор вынужден тратить больше времени на ожидание завершения операций ввода-вывода, вместо выполнения других операций.

* Состояние сетевых интерфейсов - отображение состояния сетевых интерфейсов; отображение скорости работы сетевых интерфейсов; количество сетевого трафика, проходящего через сетевой интерфейс; количество "отброшенных" пакетов. Эти данные помогут понять, справляются ли с работой сетевые контроллеры сервера, или их нужно заменить на более производительные либо выполнить какие-либо настройки программного обеспечения, взаимодействующего с сетевыми контроллерами.

* Статус сервиса web-сервера - Отображение информации о состоянии web-сервера, запущен ли он в настоящее время (Up/Down). Если сервис не запущен, то сайт не будет работать. Потребуется разобраться с причиной падения или невозможности запуска web-сервера.

* Коды ответов web-сервера. Коды ответов нужны для того, чтобы информировать клиентов (например web-браузеры), о статусе выполнения запросов. По кодам ответов можно будет понять о том, что запрос успешно обработан, произошла ошибка или требуется выполнение каких-либо дополнительных действий.

* Количество запросов к web-серверу в секунду. Чтобы своевременно отреагировать на возросшее количество запросов к web-серверу, понять о наличии проблем в его работе и возможной DDoS-атаке сервера.
#
2. Описание метрик для менеджера:

* RAM - оперативная память. Может быть множество показателей работы оператвной памяти, такие как общее количество, доступное количество, скорость ее работы, коррекция ошибок. Размер опративной памяти сервера влияет на скорость работы сервера, т.к. при большом объеме памяти сервер может выполнять больше задач и процессов одновременно.

* inodes - структуры данных, которые хранят метаданных о файлах или каталогах в файловой системе диска. Каждый файл на диске содержит свой inode, который содержит информацию о размещении этого файла на диске, его временных метках, размере, правах доступа. Количество inodes в файловой системе ограничено, поэтому при создании большого количества маленьких файлов может возникнуть проблема исчерпания inodes. В данном проекте при создании множества файлов текстовых отчетов может быстро закончиться количество доступных inodes, при том, что место на идске все ещё может быть много. Для решения этой пробелмы может потребоваться изменение настроек файловой системы диска.

* CPUla - показатель того, на сколько интенсивно нагружен процессор. Значение средней загрузки процессора поможет определить, на сколько процессор справляется с работой. Но важно понимать, что каждый случай индивидуален и перед решением о замене процессора на более производительный нужно проанализировать, какие процессы и нагружают процессор и почему.

* Для того, чтобы понять, насколько компания выполняет свои обязанности перед клиентами и какое качество обслуживания предосталвяет нужно и дальше развивать систему мониторинга. Построить мониторнг баз данных web-сервера, чтобы видеть время ответа на запросы к БД и вовремя на это реагировать. Мониторить срок действия SSL сертификатов, для исключения ситуаций, когда клиент не может попать на сайт из-за проблем с его безопасностью. Обязательно выполнять резервное копировние. Перидоичес во время с минимальным обращениями к сайту переводить его в режим технического обслуживания и выполнять это обслуживание. Позаботиться об энергоснабжении серверной инфраструктуры, чтобы исключить недостукпность сайта в связи с перебоями питания.

Также можно внедрить в компании SLI (Service Level Indicator). SLI представляет собой измерение производительности, качества работы или надежности услуги в информационной технологии. С помощью SLI могут быть определены конкретные показатели, такие как время ответа, доступность системы, пропускная способность и другие индикаторы, позволяющие оценить уровень предоставляемой услуги.
#
3. В случае, если не выделено финансирование на построение системы сбора логов, я бы предложил следующее:

* Использовать локальные журналы или логи на уровне приложений, но в этом случае потребуется участие разработчиков, для анализа этих логов и журналов. А также разработчики должны "научить" приложения выводить логи в удобном для их чтения виде.

* Выводить показатели работы систем с помощью bash или python скриптов. Также можно использовать стандартные службы, такие как rsyslog.

* Использовать такие бесплатные системы сбора логов, как Clickhouse + Vector + LightHouse или Elasticsearch + Logstash + Kibana.

Если система мониторинка в каком-то виде уже существует, то можно сделать выведение алертов в необходимые каналы (E-mail, Telegraf и т.д.).

Выбор каждого способа зависит от того, в каком виде нужны логи приложений, сколько времени их нужно хранить и в каком виде их нужно просматривать.
#
4. Должно быть в формуле не учли коды 1xx и 3xx, которые не являются ошибками. `summ_all_requests` учитывает и неуспешные запросы, которые могут снизить значение SLA.
#
5. Push-системы мониторинга:

Плюсы:

1) Меньшая задержка времени: данные передаются в режиме реального времени, система мониторинга также отображает информацию почти в реальном времени. Это особенно важно для критически важных систем, где даже кратковременная задержка может иметь серьезные последствия.

2) Большая гибкость настройки: в push-системах мониторинга можно гибко настроить, выбирать какие данные и как часто отправлять.

Минусы:

1) Высокая нагрузка на сеть: поскольку данные активно на сервер мониторинга, то значительно вырастает нагрузка на сеть. Это может вызывать проблемы с пропускной способностью сети.
2) Настройка push-систем мониторинга может вызвать сложности при ее настройке, т.к. нужно будет определиться с метриками, которые нужно передавать на сервер мониторинга.

Pull-системы мониторинга:

Плюсы:

1. Меньшая нагрузка на сеть: в pull-системах мониторинга мониторинговая система запрашивает данные у мониторируемых систем только тогда, когда это необходимо. Это помогает снизить нагрузку на сеть.

2. Единая точка контроля: при использовании pull-системы мониторинга весь процесс мониторинга и получение данных осуществляется на сервере мониторинга. Это упрощает управление и обеспечивает единую точку контроля.

Минусы:

1. Задержка времени: в pull-системах мониторинга данные получаются только после того, как система мониторинга запросит их у мониторируемых устройств или приложений. Это может затруднить моментальную реакцию на проблемы и отсрочить их решение.

2. Отказоустойчивость: если сервер мониторинга недоступен или некорректно настроен, данные могут быть недоступны или неполными.
#
6.


## Дополнительное задание* (со звёздочкой) 

# Дополнительное задание пока решил не выполнять