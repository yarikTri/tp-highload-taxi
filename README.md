# Проектирование высоконагруженного агрегатора такси

Курсовая работа в рамках 3-го семестра программы по Веб-разработке ОЦ VK x МГТУ им. Н.Э. Баумана (ex. "Технопарк") по дисциплине "Проектирование высоконагруженных сервисов"

#### Автор - [Ярослав Кузьмин](https://park.vk.company/profile/iar.kuzmin/ "Страница на портале VK x МГТУ")
#### Задание - [Методические указания](https://github.com/init/highload/blob/main/homework_architecture.md)

#### Содержание:
1. [Тема, функционал и аудитория](#1)
2. [Расчёт нагрузки](#2)
3. [Глобальная балансировка нагрузки](#3)
4. [Локальная балансировка нагрузки](#4)
5. [Логическая схема базы данных](#5)
6. [Физическая схема базы данных](#6)
7. [Алгоритмы](#7)
8. [Технологии](#8)
9. [Обеспечение надёжности](#9)
10. [Схема проекта](#10)
11. [Расчёт ресурсов](#11)


## Часть 1. Тема, функционал и аудитория <a name="1"></a>

### Тема курсовой работы - **"Проектирование сервиса агрегатора такси"**
В качестве примера и аналога выбран ведущий в России сервис заказа такси - [Яндекс.Такси](https://taxi.yandex.ru/)

### Ключевой функционал сервиса
- Регистрация и авторизация пользователей
- Заказ такси с выбором класса поездки (эконом/комфорт/бизнес и т.д.)
- Динамическое отслеживание машины/маршрута до и во время поездки - интеграция с картами
- Отдельная платформа для водителей
- Возможность внутрисервисной оплаты по привязанной банковской карте (не исключает возможность наличной оплаты или безналичного перевода водителю)

### Ключевые продуктовые решения
- Сбор CSAT'а с оцениванием водителя, поездки или удобства сервиса/приложения
- Фокус на увеличение доступности на любых мобильных устройствах с помощью минимально нагруженного клиента
- Динамическое ценообразование на основе доступности водителей в определённой зоне

### Целевая аудитория
- 40.2 млн активных пользователей в месяц в странах СНГ и EMEA [^1]
- Каждый пользователь в среднем совершает 7.3 поездок в месяц [^1]
- 67% заказов приходится на российский рынок [^1]


## Часть 2. Расчёт нагрузки <a name="2"></a>

### Продуктовые метрики

#### MAU - 40.2 млн пользователей [^1]
#### DAU - 6 млн пользователей [^5]

#### Средний размер харанилища пользователя по типам:
| Хранимые данные   | Оценочный размер на пользователя         |
|-------------------|------------------------------------------|
| Персональные данные (ФИО, почта, пароль и т.д.) | `1 КБ`     |
| Аватар              | `1 МБ`                                 |
| Адреса              | `Max 1 КБ`                             |

**Итого оценочно:** `1 МБ 1.2 КБ` на пользователя при регистрации и вводе персональных данных, без поездок.

P.S.:
- Каждый адрес весит по `0.1 КБ`. Ограничение на количество адресов - `10` - отсюда `Max 1 КБ` в таблице
- Предполагается, что у пользователя в среднем 2 адреса - работа и дом. Отсюда `0.2 КБ` в "Итого оценочно"

Возьмём общее оценочное число пользователей с запасом = `100 млн` - будем использовать далее в расчётах.

Тогда общий размер хранилища в худшем случае: `100 млн пользователей * (1 МБ на аватарку + 1 КБ на персональные данные) = 100 ТБ 100 ГБ`

Существенным образом размер хранилища (без разбиения на реляционное или S3) будет определяться количеством пользователей с аватарками.

#### Среднее количество действий пользователя по типам в день:
- `7.3 поездок/мес / 30 сут/мес ~= 0.24 поездок/сут`
- `0.24 поездок/сут / 2 = 0.12 обращений к сервису оплаты/сут` при условии, что половина заказов будет оплачиваться через сервис
- `0,24 поездок/сут / 2 = 0.12 оставлений обратной связи/сут` при условии, что каждый второй пользователь её оставляет

### Технические метрики

#### Размер хранения в разбивке по типам данных:
| Тип данных           | Оценочный размер на 1 пользователя, КБ | Суммарный прирост, ГБ/месяц |
|----------------------|----------------------------------------|-----------------------------|
| Данные пользователей | `Max 1026`                             | `Max 1197`                  | 
| История поездок      | `1`                                    | `300.5`                     |
| CSAT                 | `0.3`                                  | `45`                        |

Расчёты хранилища:
- Данные пользователей: `1026 КБ * (0.14[^4] коэф прироста в год * 100 млн пользователей / 12 мес/год) = 1 ТБ 173 ГБ/мес`
- История поездок: `7.3 поездок/мес * 40.2 млн MAU * 1 КБ = 300.5 ГБ/мес`
- Собранный CSAT: `7.3 поездок/мес * 40.2 млн MAU * 300 Б * 0.5 коэф пользователей, оставляющих обратную связь = 45 ГБ/мес`

#### RPS по типам запросов:
| Тип запроса | Средний оценочный RPS   |
|-------------|-------------------------|
| Заказ такси | `17`                    |
| Обращение к геолокации | `67930`      |
| Обращение к оплате     | `8.5`        |
| CSAT        | `8.5`                   |
| Авторизация | `1.4`                   |
| Регистрация | `0.44`                  |
| Просмотр профиля       | `70`         |

Расчёты RPS: 
- Заказ такси: `0.24 заказов/сут/пользователь * 6 млн DAU / (24 * 3600) с/сут ~= 17 RPS`
- Обращение к геолокации/картам: `40.2 млн MAU * 7.3[^1] поездок/мес * 20 RPM к геолокации * (26 мин[^6] средняя продолжительность поездки + 4 мин[^6] среднее время ожидания такси) / (30 * 24 * 3600) с/мес ~= 67930 RPS`. 20 RPM к геолокации - например, short-polling раз в 3 секунды
- Обращение к сервису оплаты: `17 RPS заказов / 2 = 8.5 RPS` при условии, что каждый второй оплачивает через внутреннюю оплату
- Обратная связь: `17 RPS заказов / 2 = 8.5 RPS` при условии, что после каждой второй поездки пассажир оценивает водителя или сервис
- Авторизация: `0.02 * 6 млн DAU / (24 * 3600) с/сут ~= 1.4 RPS`, где 0.02 - каждая 50-я сессия в приложении просит авторизации
- Регистрация: `0.14[^4] годовой коэф. прироста пользователей * 100 млн пользователей / (365 * 24 * 3600) с/год * 1.5 ~= 0.66 RPS`, где 1.5 - среднее количество попыток зарегистрироваться
- Просмотр профиля: `6 млн DAU / (24 * 3600) с/сут = 70 RPS` при условии, что пользователь просматривает свой профиль 1 раз в день

Суммарный RPS по основным запросам: = `68036` и существенным образом определяется RPS'ом к сервисам геолокации/навигации/карт

#### Сетевой трафик

**Пиковое потребление в течение суток (Гбит/с) по типам трафика:**
| Тип трафика        | Пиковое потребление, Гбит/с       | Суммарный суточный трафик, Гбит/сутки  |
|--------------------|-----------------------------------|----------------------------------------|
| Cтатические файлы  | `1.62`                            | `46656`                                |
| API                | `2.34`                            | `67392`                                |

Расчёты трафика:

**Средний:**
  - API: `68036 RPS * 1.5 КБ средний размер запроса / (1024 * 1024 / 8) КБ/Гбит = 0.78 Гбит/с`
  - Статика: `70 RPS * 1 МБ средний размер запроса / (1024 / 8) МБ/Гбит = 0.54 Гбит/с`

**Пиковый:**

  Пиковый коэф трафика от среднего с запасом = 3, т.к. недельный пиковый коэф = 2.07[^6]
   - API: `3 пиковый коэф * 0.78 Гбит/с средний трафик = 2.34 Гбит/с`
   - Статика: `3 пиковый коэф * 0.54 Гбит/с средний трафик = 1.62 Гбит/с`

**Суммарный суточный:**
  - API: `0.78 Гбит/с средний трафик * (24 * 3600) с/сут = 67392 Гбит/сут`
  - Статика: `0.54 Гбит/с средний трафик * (24 * 3600) с/сут = 46656 Гбит/сут`

 
## Часть 3. Глобальная балансировка нагрузки <a name="3"></a>

### Расположение ЦОДов

Для обеспечения минимального latency основную часть дата-центров следует размещать на территории, наиболее близкой к наибольшему количеству пользователей.
Так как в случае Яндекс.Такси 67% заказов приходится на российский рынок[^1], то ЦОДы размещать стоит в первую очередь в наиболее густонаселённых регионах РФ с наибольшим уровнем жизни, потому что именно там наибольшая ЦА сервиса - расположим ЦОДы под ***Москвой*** и ***Санкт-Петербургом***, также уменьшим latency для большой части пользователей РФ с помощью ЦОДа около ***Екатеринубрга***.

Обозначенный целевой рынок - не только Россия, но и страны СНГ и EMEA. Следовательно, необходимо также разместить дата-центры в центральной Европе - ***Нидерланды, Амстердам***, на Ближнем Востоке - ***ОАЭ, Дубай*** (т.к. именно там активно развивается рынок комфортного такси по типам поездок и есть большая ЦА состоятельных пользователей) и трафик из Африки "забиндим" на ЦОДы в Нидерландах и ОАЭ.

При выборе конкретных населённых пунктов для размещения ЦОДов стоит также обращать внимание на стоимость электроэнергии и географическую доступность - для более простой поддержки, обслуживания и проведения технических работ на серверах.

### Методы глобальной балансировки

Для глобальной балансировки запросов и нагрузки будем использовать:
- Сначала для определения региона - latency-based DNS (например, с помощью Amazon Route 53), так как он совмещает себе возможность обрабатывать запросы ближайшими ЦОДами и "мониторинг" RTT в сети
- Далее в рамках своей Autonomous System будем балансировать запросы между ЦОДами с помощью Routing - BGP Anycast с помощью метрик минимальных хопов


## Часть 4. Локальная балансировка нагрузки <a name="4"></a>

### Cхема балансировки

Так как в проекте используется роутинг с помощью BGP Anycast, балансировка на L4 может быть полезна в минимальном количестве сценариев, так как роутинг считается эффективнее, чем LVS, а список их задач существенно пересекается.

Для балансировки внутри ЦОДов будем использовать надёжный и гибкий в конфигурации L7-балансировщик Nginx. Мы будем его применять для следующих процессов:
- Равномерная балансировка запросов между бэкендами с помощью Round-Robin алгоритма с циклическим списком и Least Connection
- Реализация API-Gateway (функциональная балансировка)
- Разрешение "задачи медленных клиентов" в случае синхронных бэкендов - отсутствие блокировки воркеров бэкенда с помощью асинхронной обработки медленных соединений
- Обеспечение перезапуска сервиса без остановки обслуживания
- Терминация SSL
- Отдача статики
- Кеширование запросов
- Сжатие контента с помощью gzip
- Обеспечение походов в сервисы авторизации на уровне web-сервера для увеличения RPS, разгрузки бэкенда от авторизации и, в следствие, защиты от ddos-атак
- Retry идемпотентных запросов с помощью парсинга оригинального протокола (HTTP)
- Простановка HTTP-заголовков на уровне web-сервера, например, X-Real-IP - настоящий IP клиента

Далее для оркестрации сервисов будем использовть Kubernetes. k8s будет в первую очередь обеспечивать:
- Auto-scaling
- Service discovery
- Распределение stateless сервисов по кластеру
- Управление deployment-циклом приложений

### Схема отказоустойчивости

Nginx и k8s в связке обеспечат нам высокий уровень отказоустойчивости сервиса:
- Nginx во многих сценариях обеспечивает существенное увеличение rps к сервису, retry запросов после падения бэкенда, перезапуск web-сервера без downtime и равномерное распределение запросов по бэкендам
- k8s с помощью readiness-проб обеспечит исключение из кластера упавших подов и добавление новых

Используемые дополнительные методы обеспечения надёжности и отказоустойчивости:
- Активный сбор метрик с машин и наблюдение за ними
- Алёртинг (сообщения, sms, звонки) в критичных ситуациях, таких как OOM, CPU-полка или 500-ки
- Наличие графика дежурств и SRE-инженеров


## Часть 5. Логическая схема базы данных <a name="5"></a>

<img width="850" alt="image" src="https://github.com/yarikTri/tp-highload-taxi/assets/91901091/05e6eb30-5a01-4b62-b246-4b6a60478662">


## Часть 6. Физическая схема базы данных <a name="6"></a>

### Выбор хранилищ данных
1. В качестве основного (но не самого нагруженного) хранилища данных выбрана СУБД ***PostgreSQL***. В нём будет содержаться вся основная информация сервиса, у которой нет большого профиля нагрузки на изменение, а в основном чтение - Поездки (`Rides`), Пользователи (`Passangers`), Водители (`Drivers`), их организации и права (`Organisations` и `Driver_license`), адресы (`Adresses`) и так далее.
2. Для хранения большого количества time-series данных, таких как локации водителей (`Drivers_location`), история отзывов (`Rating_history`) и подробные данные поездок (`Rides_metrics`), будем использовать ***ClickHouse***. Каждый клиент водителя на линии будет прокидывать в него данные о своей геопозиции раз в ~10 секунд.
В последствии эта информация будет использоваться для алгоритмов подбора водителей и ценообразования.
3. Для задач аналитики акже заведём столбчатое OLAP-ХД ***Clickhouse*** - в него с помощью распределённой `CRON-task` будут дублицироваться аналитические данные сервиса - время ожидания, длительности поездок, время поездок, геопозиции, CSAT и т.д..
4. Для оптимизации узких мест (например, хранилище геопозицй с большим RPS на запись) заведём топики в брокере сообщений ***Kafka***, читать из каждого из которых будут специальные для каждого кейса `consumer'ы`.
5. Хранить пользовательские сессии доверим key-value ХД в системе ***YT***, где в качестве ключа будет `session_id`.
6. Аватарки пользователей и водителей будем хранить в объектном хранилище `Amazon S3`.
7. Для хранения больших данных будем использовать систему хранения ***YTsaurus*** - там будут храниться логи сервиса

### Денормализация
1. Для каждого водителя будет храниться его рейтинг в отдельной таблице в другом хранилище, т.к. профиль нагрузки отличается от таблицы `Driver`. Это поле на основе показателей CSAT'а водителя будет обновляться после каждого оставленного пассажиром отзыва.
3. Также в таблице `Ride` будем хранить поля `driver_full_name`, `` и `car_info`, которые будут отображать данные о водителе и помогут избавиться от JOIN'ов для последующего шардирования
4. Также в таблице `Ride` будем хранить поле `Ride_class`, которое будет отображать класс поездки и также поможет избавиться от JOIN'ов для последующего шардирования

### Шардинг
- Таблицу `Ride_metrics` будем шардировать по полю `ride_id` - заведём 100 шардов и будем относительно равномерно распределять по хешу
- Так как мы на предыдущем этапе избавились от лишних JOIN'ов для нагруженной таблицы `Ride` - можем настроить на неё шардинг по `passanger_id`

### Партицирование
- Big Data таблицы логов в ***YT*** спартицируем по датам
- В ***Clickhouse*** для метрик поездок настроим партицирование по датам

### Индексы
- В общем случае для оптимизации чтения будем покрывать внешние ключи **BTREE-индексами**: `Driver.user_id`, `Passanger.user_id`, `Passanger_Address.passanger_id`, `Passanger_Address.address_id`
- Для таблицы `Driver_location` построим GIST-индекс, чтобы быстро находить вхождение геолокации в определённый шестиугольник (`Hexagon`)

### Реплицирование
- `PostgreSQL` - 1 ведущий узел и 3 ведомых узла (1 master, 3 slaves)
- `ClickHouse` - 2 ведущих узла (т.к. большой профиль нагрузки на запись/обновление) и 2 ведомых узла (2 masters, 2 slaves)

## Часть 7. Алгоримты <a name="7"></a>

### Алгоритм поиска и подбора водителей:
Главной составляющей алгоритма ценообразования является разделение карты (городов, в которых проект запущен) на шестиугольные области - хексагоны. 

Ориентировочно сторона такого хексагона будет составлять 600 метров, периметр 3600 метров и площадь 935 км^2. Для наглядности привожу приблизительный размер на карте:

<img width="706" alt="image" src="https://github.com/yarikTri/tp-highload-taxi/assets/91901091/014a30f0-6ca3-4463-962a-4613089425a8">

В каждом таком хексагоне будет некоторое количество свободных/занятых водителей, доступных для пассажиров для заказа.

Устройства водителей, вышедших на линию, будут каждые 10 секунд отправлять свою геолокацию в хранилище данных системы. 
В свою очередь, пассажир при заказе такси будет анонсировать свою геопозицию и определяться его хексогон. 
В первую очередь заказ будет предлагаться принять водителям, находящимся по самым свежеотправленным данным в том же хексагоне, что и заказчик. 
Если все водители в данном хексагоне отказались или водителей нет, то заказ сразу переходит в предложение для водителей в шести приграничных хексагонах и так далее.

В ранних частях курсовой работы было описано, что хексагоны для оптимизации поиска вхождений водителей в зоны координаты водителей будут проиндексированы GIST-индексом - 
популярным решением для ускорения поиска вхождения по геокоординатам.

### Алгоритм ценообразования:
Ключевым фактором образования цены заказа для пассажиров будет коэффициент загруженности `КЗ` (0 ≤ `КЗ` ≤ 1) хексагона пассажира: чем он выше, тем больше будет цена заказа. 
`КЗ` определяется количеством водителей в хексагоне, что означает, что на него косвенно будут влиять погодные условия, время суток, час пик и так далее. 
Прежде всего это необходимо для того чтобы большой ценой привлекать водителей брать заказ, даже если они заняты или далеко и балансировать между удовлетворением как водителей честными оплатами за большую нагрузку, 
так и пассажиров относительно дешёвой (или хотя бы обоснованной) ценой за заказ.

Кроме того на цену заказа также влияет "база" каждого тарифа в рублях или иной валюте с коэффициентом в зависимости от средней платёжеспособности в городе и дальность заказа также со своим коэффициентом для каждого города.

Таким образом, для Москвы в описанной схеме приблизительна справедлива следующая схема:
- "Эконом": `200 база + (300 * КЗ) + километраж * 80`
- "Комфорт": `300 база + (400 * КЗ) + километраж * 100`
- "Комфорт+": `400 база + (600 * КЗ) + километраж * 120`
- И т.д. в зависимости от потребностей бизнеса


## Часть 8. Технологии <a name="8"></a>
| Технология  |           Применение            | Обоснование |
|-------------|---------------------------------|-------------|
| Go          | Backend, основной язык сервисов | Производительность, "богатая асинхронность", низкий порог входа, большое количество технологий из коробки, высокая утилизация CPU, Garbage Collector, конкуренты Rust, Java, .NET, Kotlin |
| С/С++       | Backend, скрипты                | Оптимизация узких мест или мест с критичной необходимостью в высокой нагрузке, выбор языка зависит от целей |
| Python/Lua  | Backend, скрипты, расширение    | Быстрое написание скриптов без требовательной производительности, внешнее расширение функционала имеющихся технологий, например Nginx |
| PostgreSQL  | SQL ХД, основная БД сервисов    | Отлично подходит для реляционного хранения данных большинства CRUD-сервисов, низкий порог входа, конкурент MySQL |
| ClickHouse  | Хранилище time-series данных для алгоритмов, хранилище аналитики  | Эффективная работа с OLAP-нагрузкой, column-family DB, конкурент Greenplum |
| YT/YTsaurus | Распределённое хранение и обработка больших данных: хранение сессий в NoSQL key-value ХД (Конкурент Redis), хранение логов | Более современные и "чистые" решение по сравнению с главным конкурентом *Hadoop*, MapReduce алгоритмы распределённое файловое хранилище |
| Nirvana     | Вычислительные задачи, распределённый CRON, например, выгрузка аналитики в ClickHouse | Запуск MPP-CRON-тасок на ресурсах и алгоритмах YT |
| Kafka       | Асинхронный стриминговый сервис, брокер сообщений | Отложенное эффективное выполнение задач, партицирование из коробки, конкурент RabbitMQ |
| Amazon S3   | Хранилище статики: аватарки     | Простое взаимодействие, стандарт |
| Kotlin      | Mobile, платформа Android       | Лидер рынка в своей категории, быстрая разработка, JVM => мультиплатформенность, много популярных технологий и Java-совместимых библиотек, большое коммьюнити |
| Swift       | Mobile, платформа iOS           | Лидер рынка в своей категории, быстрая разработка и большое коммьюнити |
| Typescript  | Frontend                        | Строгая типизация и множество готовых решений, огромное коммьюнити |
| React       | Frontend                        | Компонентный подход, быстрая разработка, множество решений из коробки |
| Nginx       | Reverse proxy balancer          | Многофункциональное высокопроизводительное решение. Возможность расширять функционал кодом; подробнее в [4. Локальная балансировка нагрузки](#4), конкурент Envoy |
| Vault       | Хранилище секретов              | Возможность хранить секреты и настраивать политики доступа |
| Gitlab      | CI/CD, Система контроля версий, монорепозиторий, командная разработка | Удобное сопровождение всех циклов разработки, обеспечения качества проекта и возможность настроек прав проекта |
| Kubernetes  | Deploy                          | Масштабирование, отказоустойчивость, оптимальная утилизация ресурсов; подробнее в [4. Локальная балансировка нагрузки](#4) |
| VictoriaMetrics | Хранилище метрик и система работы с ними | Производительнее Prometheus'а |
| Grafana     | Графики, мониторинг и алёрты    | Удобное популярное решение для визуализации состояния сервиса (метрик), мониторинга и уведомления об инцидентах |


## Часть 9. Обеспечение надёжности <a name="9"></a>

### Надёжность сервисов
- Failover policy межсервисных/внутрисервисных запросов: Retry и Timeout - потенциально уменьшим количество 500-к и 504-к
- Сегментирование/кластеризация API по группам (например, по функциональности или профилю нагрузки)
- Пробивание дырок между сервисами для обеспечения безопасности
- Graceful shutdown сервисов: не получаем новые запросы, но выполняем текущие
- Graceful degradation

### Аппаратная надёжность и оптимизация утилизации ресурсов
- Регулярное перерасчитывание аппаратного потребления и его необходимого запаса и резервирование ресурсов и физических компонентов (сеть, CPU, RAM, RAID и т.д.) исходя из него
- Выделение квот сервисам
- Кэши на балансерах/БД/бэкенде
- Rate limiter клиентских запросов по IP (обдумать кейс с NAT) или ID сессии
- Урезанная выдача при пиковых нагрузках

### Проверка состояния сервисов
- Healthcheck'и сервисов в k8s с динамическим выделением ресурсов и отключения от сети упавших машин
- Максимально презентативные дашборды в мониторинге
- Перезапуск и реплицирование stateless-сервисов
- Масштабный сбор метрик: утилизация ресурсов машин БД/балансера/бэкенда, тайминги выполнения запросов там же, перцентили
- Алёртинг инцидентов (рост 500-к, отказавшая реплика, упавший сервер/балансер) с эскалацией по иерархии: чат с ботами-мониторинга, смс, звонок дежурному или руководителю
- Хранение логов: месяц всех, год+ важных (error или 500) или "участвовавших" в инциденте

### Deploy circle
- Жёсткие ограничения на CI/CD цикл
- Невозможность merge в main без аппрувов, пройденых check'ов
- Максимально возможное покрытие Unit/Integration/E2E-тестами
- Highload-тестирование, также в рамках учения с отключением ДЦ
- Деплой новой версии сервиса не инициирует отказ в проде старой версии
- Одновременный деплой бэкенда(+миграции) и фронтенда для сохранения совместимости

### Дата-центры
- Распределённые независимые ЦОДы
- Периодические учения с отключением ЦОДов и проверки держания нагрузки
- БД, балансеры и бэкенд в разных кластерах

### Базы данных
- Резервирование БД
- Репликация с несколькими мастер-нодами
- Мониторинг реплик на падение с отключением от запросов. При включении, постепенное восстановление многократным чтением WAL'а.
- Fallback policy при падении мастер-реплики
- Снапшоты/резервные копии PostgreSQL, S3

### Human resources
- SRE-инженеры
- Жёсткая политика код-ревью
- Квалифицированные разраюотчики
- Команда DevOps, платформы разработки
- Команда саптеха
- Команда антифрода


## Часть 10. Схема проекта <a name="10"></a>

![HL_taxi_scheme drawio](https://github.com/yarikTri/tp-highload-taxi/assets/91901091/b5f9f7b7-d2cb-40ab-b948-1000fd2c7d39)


## Часть 11. Расчёт ресурсов <a name="11"></a>

- S3:
  - Прирост хранения: 1.3 ТБ/год (без резервных копий)
  - 1 резервная копия
  - `5 лет * 2 копии * 1.3 ТБ/год / 1 ТБ конфигурации` = 13 серверов
  - Выбранная конфигурация: 1U 2x6430/4x8GB/1xNVMe1TB/10Gb/s
  
- БД:
  - Прирост хранения: ~1 ТБ/год, включая все основные ХД: PG, CH, YT
  - 4 реплики как в ***PostgreSQL***, так и в ***Clickhouse***
  - `5 лет * 4 реплики * 1 ТБ/год / 1 TB конфигурации` = 20 серверов
  - Выбранная конфигурация: 1U 2x6430/4x16GB/1xNVMe1TB/10Gb/s

- Сервисы в Kubernetes
  - ~100 RPS/CPU
  - 70000 RPS пик => `70000 RPS / 100 PRS/CPU = 700 CPU => 700 CPU / 64 CPU/U` = 11 серверов
  - Выбранная конфигурация: 1U 2x6430/8x64GB/2xNVMe256GB/2x20Gb/s

- BFF, Balancers, API Gateway:
  - 500 RPS/CPU по таблице
  - 70000 RPS пик => `70000 RPS / 500 RPS/CPU = 140 CPU => 140 CPU / 64 CPU/U` = 3 сервера
  - 2.34 Гбит/с пиковый трафик. Т.к. нужен всё равно большой запас, т.к. канал заполняется не полностью - выберем 10 Gb/s железо
  - Выбранная конфигурация: 1U 2x6430/8x64GB/1xNVMe128GB/10Gb/s
 
- Стойки: `47 U суммарно / 48 U/стойка = 1 стойка`

### Список источников:
[^1]: [Финансовый отчёт Яндекса за Q2'2023](https://yastatic.net/s3/ir-docs/events/2023/Supplementary_slides_2Q23_RUS.pdf)
[^4]: [Прирост количества пользователей Яндекс Go за год](https://tass.ru/ekonomika/17054865)
[^5]: [Дневная аудитория Яндекс.Такси в 2020](https://investim.guru/obzory/skolko-polzovateley-polzuetsya-yandeks-taksi-v-den-statistika-i-aktualnye-dannye)
[^6]: [Исследование Яндекса о такси (2015)](https://yandex.ru/company/researches/2015/moscow/taxi)
