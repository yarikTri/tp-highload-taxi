# Проектирование высоконагруженного агрегатора такси

Курсовая работа в рамках 3-го семестра программы по Веб-разработке ОЦ VK x МГТУ им. Н.Э. Баумана (ex. "Технопарк") по дисциплине "Проектирование высоконагруженных сервисов"

#### Автор - [Ярослав Кузьмин](https://park.vk.company/profile/iar.kuzmin/ "Страница на портале VK x МГТУ")
#### Задание - [Методические указания](https://github.com/init/highload/blob/main/homework_architecture.md)

#### Содержание:
1. [Тема, функционал и аудитория](#1)
2. [Расчёт нагрузки](#2)
3. [Глобальная балансировка](#3)

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
- Каждый адрес по `100 Б`. Ограничение на количество адресов - `10` - отсюда `Max 1 КБ` в таблице
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
 
## Часть 3. Глобальная балансировка <a name="3"></a>

### Расположение ЦОДов

Для обеспечения минимального latency основную часть дата-центров следует размещать на территории, наиболее близкой к наибольшему количеству пользователей.
Так как в случае Яндекс.Такси 67% заказов приходится на российский рынок[^1], то ЦОДы размещать стоит в первую очередь в наиболее густонаселённых регионах РФ с наибольшим уровнем жизни, потому что именно там наибольшая ЦА сервиса - расположим ЦОДы под ***Москвой*** и ***Санкт-Петербургом***, также уменьшим latency для большой части пользователей РФ с помощью ЦОДа около ***Екатеринубрга***.

Обозначенный целевой рынок - не только Россия, но и страны СНГ и EMEA. Следовательно, необходимо также разместить дата-центры в центральной Европе - ***Нидерланды, Амстердам***, на Ближнем Востоке - ***ОАЭ, Дубай*** (т.к. именно там активно развивается рынок комфортного такси по типам поездок и есть большая ЦА состоятельных пользователей) и трафик из Африки "забиндим" на ЦОДы в Нидерландах и ОАЭ.

При выборе конкретных населённых пунктов для размещения ЦОДов стоит также обращать внимание на стоимость электроэнергии и географическую доступность - для более простой поддержки, обслуживания и проведения технических работ на серверах.

### Методы глобальной балансировки

Для глобальной балансировки запросов и нагрузки будем использовать:
- Сначала для определения региона - latency-based DNS (например, с помощью Amazon Route 53), так как он совмещает себе возможность обрабатывать запросы ближайшими ЦОДами и "мониторинг" RTT в сети
- Далее в рамках своей Autonomous System будем балансировать запросы между ЦОДами с помощью Routing - BGP Anycast с помощью метрик минимальных хопов

### Список источников:
[^1]: [Финансовый отчёт Яндекса за Q2'2023](https://yastatic.net/s3/ir-docs/events/2023/Supplementary_slides_2Q23_RUS.pdf)
[^4]: [Прирост количества пользователей Яндекс Go за год](https://tass.ru/ekonomika/17054865)
[^5]: [Дневная аудитория Яндекс.Такси в 2020](https://investim.guru/obzory/skolko-polzovateley-polzuetsya-yandeks-taksi-v-den-statistika-i-aktualnye-dannye)
[^6]: [Исследование Яндекса о такси (2015)](https://yandex.ru/company/researches/2015/moscow/taxi)
