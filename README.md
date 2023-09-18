# Проектирование высоконагруженного агрегатора такси

### Автор - [Ярослав Кузьмин](https://park.vk.company/profile/iar.kuzmin/ "Страница на портале VK x МГТУ")

## Часть 1. Тема и целевая аудитория

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

## Часть 2. Расчёт нагрузки (не завершено)

### Продуктовые метрики

#### MAU - 40.2 млн пользователей [^1]
#### DAU - 6 млн пользователей [^5]

#### Средний размер харанилища пользователя по типам:
| Хранимые данные   | Оценочный размер на пользователя       |
|-------------------|----------------------------------------|
| Персональные данные (ФИО, почта, пароль и т.д.) | Max 1 КБ |
| Аватар              | ~ 1 МБ                               |
| Адреса              | ~ 100 Б \* кол-во адресов            |

**Итого оценочно:** 1 МБ 1.3 КБ на пользователя при регистрации и вводе персональных данных, без поездок.

P.S.:
- При оценке учитывалась стандартная кодировка - 2 байта на символ
- Предполагается, что у пользователя в среднем 2 адреса - работа и дом

Получаем, что если брать с запасом общее число пользователей за 100 млн, то для хранения данных о пользователях
необходимо в худшем случае выделить под хранение данных пользователей 100 млн \* (1 МБ на аватарку + 1 КБ Б на персональные данные) = 100 ТБ + 100 ГБ.
Существенным образом размер хранилища (без разбиения на реляционное или S3) будет определяться количеством пользователей с аватарками.

#### Среднее количество действий пользователя по типам в день:
- 7.3 поездок в месяц / 30 ~= 0.24 поездок в день
- 0.24 / 3 = 0.08 обращений за сервисом оплаты в день при допущении, что все 3 способа оплаты равноценно распределены по популярности
- 0,24 / 2 = 0.12 отзывов обратной связи в день при допущении, что каждый второй пользователь её оставляет

### Технические метрики

#### Размер хранения в разбивке по типам данных:
| Тип данных           | Оценочный размер      | Прирост в месяц     |
|----------------------|-----------------------|---------------------|
| Данные пользователей | Max 1 МБ 1.3 КБ       | Max 1 ТБ 173 ГБ     | 
| История поездок      | ~ 1 КБ                | ~ 300.5 ГБ          |
| CSAT                 | 300 Б                 | 45 ГБ               |

Расчёты:
- Данные пользователей: 1 МБ 1.3 КБ * (0.14[^4] коэф прироста в месяц * 100 млн всего / 12) = 1 ТБ 173 ГБ
- История поездок: 7.3 поездок/месяц \* 40.2 млн MAU \* 1 КБ = 300.5 ГБ
- Собранный CSAT: 7.3 поездок/месяц \* 40.2 млн MAU \* 300 Б * 0.5 коэф пользователей, оставляющих обратную связь = 45 ГБ

#### RPS по типам запросов:
| Тип запроса | Средний оценочный RPS |
|-------------|-----------------------|
| Заказ такси | 17 = 0.24 сред. заказов/день \* 6 млн DAU / (24 \* 3600) |
| Обращение к оплате | 5.66 = 17 / 3  |
| CSAT        | 8.5 = 17 / 2          |
| Авторизация | 1.4 = 0.02 \* 6 млн DAU / (24 \* 3600)                   |
| Регистрация | 0.45 = 0.14[^4] * 100 млн всего / (12 * 30 * 24 * 3600)  |
| Просмотр профиля | 70 = 6 млн DAU / (24 \* 3600)                       |

Суммарный RPS по основным бизнес-запросам: ~= 103

#### Сетевой трафик
Оценим размер пустого HTTP-запроса: 40 Б минимальный TCP-сегмент[^2] + 26 Б минимальный HTTP-request[^3] + ~100 Б оверхед на заголовки и длину хоста = 166 Б

**Пиковое потребление в течение суток (Гбит/с) по типам трафика:**
| Тип трафика        | Пиковое потребление, Гбит/с       | Суммарный суточный трафик, Гбит/сутки  |
|--------------------|-----------------------------------|----------------------------------------|
| Cтатические файлы  | 1.6 = 2 * (70 + 17 * 2) * 1 МБ    | 69200 = 138600 / 2                     |
| API                | 0.000012 = 206 * 1.2 КБ           | 10.7 = 21.4 / 2                        |

### Список источников:
[^1]: [Финансовый отчёт Яндекса за Q2'2023](https://yastatic.net/s3/ir-docs/events/2023/Supplementary_slides_2Q23_RUS.pdf)
[^2]: [Минимальный размер TCP-сегмента](https://superuser.com/questions/243008/whats-the-minimum-size-of-a-tcp-packet)
[^3]: [Минимальный размер HTTP-запроса](https://stackoverflow.com/questions/25047905/http-request-minimum-size-in-bytes)
[^4]: [Прирост количества пользователей Яндекс Go за год](https://tass.ru/ekonomika/17054865)
[^5]: [Дневная аудитория Яндекс.Такси в 2020](https://investim.guru/obzory/skolko-polzovateley-polzuetsya-yandeks-taksi-v-den-statistika-i-aktualnye-dannye)
