# Представляем InterSystems API Manager

Недавно мы выпустили InterSystems API Manager (IAM) - новый компонент InterSystems IRIS Data Platform, обеспечивающий наблюдение, контроль и управление трафиком в/из web API в рамках IT-инфраструктуры.

В этой статье я покажу как настраивать IAM и продемонстрирую некоторые из многочисленных возможностей, которые доступны вам с IAM. InterSystems API Manager позволяет вам:

- Наблюдать за API, понимать кто использует API, какие API наиболее популярны, а какие требуют доработки.
- Контролировать кто использует API и ограничивать использование API от простого ограничения доступа до ограничений в зависимости от запроса - у вас есть настраиваемый контроль и вы можете быстро реагировать на изменения паттернов потребления API.
- Защищать API с помощью централизованных механизмов безопасности, таких как OAuth2.0, LDAP или Key Token Authentication.
- Упростить работу сторонних разработчиков и предоставить им превосходный опыт работы с API, открыв специальный портал для разработчиков.
- Масштабировать API и обеспечить минимальную задержку при ответе.

Управление API является необходимым для перехода к микросервисной архитектуре либо SOA, упрощая интеграцию между отдельными (микро)сервисами, делая их доступными для всех внешних и внутренних потребителей. В итоге новые API становится проще создавать, поддерживать и потреблять.

Если вы уже используете InterSystems IRIS, вы можете добавить к вашей лицензии опцию IAM. Опция IAM бесплатна для заказчиков InterSystems IRIS, но для начала использования IAM необходимо запросить в InterSystems новый лицензионый ключ.

Если вы еще не используете InterSystems IRIS и только планируете попробовать InterSystems API Manager, обратитесь в InterSystems.

# Начало работы и установка

Заказчики InterSystems дистрибутив IAM могут загрузить с сайта [WRC](https://wrc.intersystems.com/wrc/)  раздел "Software Distribution" и запустить как Docker контейнер. Минимальные системные требования:

- [Docker](https://docs.docker.com/install/) 17.04.0+.
- [Docker Compose](https://docs.docker.com/compose/install/) 1.12.0+.
- InterSystems IRIS 2019.1.1+.

Первоначально требуется загрузить Docker образ (Важно! Архив с WRC не является Docker образом, нужно его распаковать, внутри Docker образ):

```
docker load -i iam_image.tar
```

Эта команда сделает образ IAM доступным для последующего использования на вашем сервере. IAM работает как отдельный контейнер, поэтому вы можете масштабировать его независимо от InterSystems IRIS. Для запуска IAM требуется доступ к InterSystems IRIS для загрузки лицензии.

Настройте InterSystems IRIS:

- Включите веб-приложение `/api/IAM`
- Включите пользователя `IAM`
- Измените пароль пользователя `IAM`

Теперь запустим контейнер IAM. В архиве вы найдете скрипты `iam-setup` для Windows и Unix (и Mac). Эти скрипты помогут вам правильно настроить переменные окружения, позволяя контейнеру IAM установить соединение с InterSystems IRIS. Вот пример работы скрипта на Mac:

```
source ./iam-setup.sh 
Welcome to the InterSystems IRIS and InterSystems API Manager (IAM) setup script.
This script sets the ISC_IRIS_URL environment variable that is used by the IAM container to get the IAM license key from InterSystems IRIS.
Enter the full image repository, name and tag for your IAM docker image: intersystems/iam:0.34-1-1
Enter the IP address for your InterSystems IRIS instance. The IP address has to be accessible from within the IAM container, therefore, do not use "localhost" or "127.0.0.1" if IRIS is running on your local machine. Instead use the public IP address of your local machine. If IRIS is running in a container, use the public IP address of the host environment, not the IP address of the IRIS container. xxx.xxx.xxx.xxx               
Enter the web server port for your InterSystems IRIS instance: 52773
Enter the password for the IAM user for your InterSystems IRIS instance: 
Re-enter your password: 
Your inputs are:
Full image repository, name and tag for your IAM docker image: intersystems/iam:0.34-1-1
IP address for your InterSystems IRIS instance: xxx.xxx.xxx.xxx
Web server port for your InterSystems IRIS instance: 52773
Would you like to continue with these inputs (y/n)? y
Getting IAM license using your inputs...
Successfully got IAM license!
The ISC_IRIS_URL environment variable was set to: http://IAM:****************@xxx.xxx.xxx.xxx:52773/api/iam/license
WARNING: The environment variable is set for this shell only!
To start the services, run the following command in the top level directory: docker-compose up -d
To stop the services, run the following command in the top level directory: docker-compose down
URL for the IAM Manager portal: http://localhost:8002
```

Как видите полное имя образа, IP-адрес, порт InterSystems IRIS и пароль для пользователя IAM - все, что нужно для начала работы.

Вместо запуска скрипта можно установить переменные окружения вручную:
```
ISC_IAM_IMAGE=intersystems/iam:0.34-1-1
ISC_IRIS_URL=http://IAM:<PASS>@<IP>:<PORT>/api/iam/license
```

# Запуск

Теперь запустим IAM, выполнив команду:
```
docker-compose up -d
```

Эта команда упорядочивает контейнеры IAM и гарантирует, что все запущено в корректно. Состояние контейнеров проверяется с помощью  команды:
```
docker ps
```

Откройте в браузере интерфейс администратора `localhost:8002`.

![](https://community.intersystems.com/sites/default/files/inline/images/images/1%20-%20Overview.png)

Пока что тут пусто, потому что это совершенно новый узел. Давайте это изменим. IAM поддерживает концепцию рабочих мест (workspaces) для разделения API на модули и/или команды. Перейдите на рабочую область "default" которую мы будем использовать для наших экспериментов.

![Панель инструментов рабочей области по умолчанию](https://community.intersystems.com/sites/default/files/inline/images/images/2%20-%20default%20Dashboard.png)

Количество запросов для этого рабочего пространства по-прежнему равно нулю, но вы получите представление об основных концепциях IAM в меню слева. Первые два элемента: Сервисы (Services) и Маршруты (Routes) наиболее важны:
- Сервис (Service) - API, к которому мы хотим предоставить доступ потребителям. Таким образом, REST API в InterSystems IRIS является Сервисом, как и, например, Google API, если вы захотите его использовать. 
- Маршрут (Route) решает, в какой Сервис должны быть перенаправлены входящие запросы. Каждый Маршрут имеет определенный набор условий, и если они выполнены, запрос направляется в соответствующий Сервис. Например, Маршрут может совпадать с IP, доменом отправителя, HTTP методами, частями URI или комбинацией указанных примеров.

## Сервис

Давайте создадим Сервис InterSystems IRIS, со следующими значениями:

| Поле     | Значение     | Описание                             |
|----------|--------------|--------------------------------------|
| name     | iris         | Название Сервиса                     |
| host     | IP           | Хост или ip сервера InterSystems IRIS|
| port     | 52773        | Веб-порт сервера InterSystems IRIS   |
| path     | /api/atelier | Корневой путь                        |
| protocol | http         | Протокол                             |

Остальные значения оставьте по умолчанию. Нажмите кнопку `Create` и запишите ID созданного Сервиса.

## Маршрут

Теперь давайте создадим маршрут:

| Поле       | Значение     | Описание                         |
|------------|--------------|----------------------------------|
| path       | /api/atelier | Корневой путь                    |
| protocol   | http         | Протокол                         |
| service.id | guid from 3  | Сервис (ID из предыдущего шага)  |

Остальные значения оставьте по умолчанию.  Нажмите кнопку `Create` и запишите ID созданного Маршрута. По умолчанию IAM слушает входящие запросы на порту 8000. Теперь запросы, отправляемые на `http://localhost:8000` и начинающиеся с `/api/atelier` перенаправляются в InterSystems IRIS. 

## Тестирование

Давайте попробуем создать запрос в REST клиенте (я использую [Postman](https://www.getpostman.com/downloads/)).

![REST запрос в Postman](https://community.intersystems.com/sites/default/files/inline/images/images/3%20-%20first%20Postman%20request.png)

Отправим GET запрос на `http://localhost:8000/api/atelier/` (не забудьте `/` на конце) и получим ответ от InterSystems IRIS. Каждый запрос проходит через IAM который собирает метрики:
- Код состояния HTTP.
- Задержка.
- Мониторинг (если настроен).

Я сделал еще несколько запросов (включая два запроса на несуществующие конечные точки, такие как /api/atelier/est/), результаты сразу видны на панели мониторинга:

![Приборная панель с некоторыми метриками](https://community.intersystems.com/sites/default/files/inline/images/images/4%20-%20dashboard%20with%20some%20metrics.png)

# Работа с плагинами

Теперь, когда у нас настроен Маршрут, мы можем управлять нашим API. Мы можем добавлять характеристики, которое дополнят наш сервис.

Наиболее распространенным способом изменения поведения API является добавление плагина. Плагины изолируют отдельные функциональные возможности и могут быть подключены к IAM как глобально, так и только к отдельным сущностям, таким как Пользователь (группа пользователей), Сервис или Маршрут. Мы начнем с добавления плагина "Rate Limiting" (Ограничение количества запросов) в Маршрут. Для установления связи между подключаемым плагином и маршрутом нам нужен уникальный идентификатор (ID) маршрута.

## Ограничение количества запросов

Нажмите Plugins (Подключаемые модули) в меню левой боковой панели. Вы видите все активные плагины на этом экране, но так как этот сервер IAM новый, активных плагинов пока нет. Поэтому переходите к следующему шагу, нажав "New Plugin".

Плагин, который нам нужен, находится в категории "Traffic Control" (Управление трафиком) и называется "Rate Limiting" (Ограничение количества запросов). Выберите его. Есть довольно много настроек, которые вы можете здесь задать, но нас волнуют только два поля:


| Поле          | Значение  | Описание                         |
|---------------|-----------|----------------------------------|
| route_id      | ID        | ID Маршрута                      |
| config.minute | 5         | Количество запросов в минуту     |

Вот и всё. Плагин настроен и активен. Замечу, что мы можем выбирать различные временные интервалы, такие как минута, час или день. Настройки можно комбинировать (например, 1000 запросов в час и при этом 100 запросов в минуту). Я выбрал минуты, так как это позволяет  легко проверить работу плагина. 

Если вы снова отправите тот же самый запрос в Postman, вы увидите, что ответ возвращается с 2 дополнительными заголовками:
- XRateLimit-Limit-minute: 5
- XRateLimit-Remaining-minute: 4 

Это говорит клиенту, что он может совершать до 5 запросов в минуту и в текущем временном интервале может совершить ещё 4 запроса.

![Ограничение скорости](https://community.intersystems.com/sites/default/files/inline/images/images/6%20-%20rate%20limiting%20headers.png)

Если вы будете делать один и тот же запрос снова и снова, то в конце концов у вас закончится доступная квота и вместо этого вы получите код статуса HTTP 429 со следующим телом ответа:

![API Превышение](https://community.intersystems.com/sites/default/files/inline/images/images/7%20-%20API%20rate%20limit%20exceeded.png)

Подождите минуту, и вы вновь сможете отправлять запросы. 

Это удобный механизм, позволяющий:
- Защитить бэкенд от скачков нагрузки.
- Сообщать клиентам, сколько запросов они могут совершить.
- Монетизировать API.

Вы можете задавать значения для различных временных интервалов и таким образом сглаживать API трафик за определенный период времени. Допустим, вы разрешаете 600 запросов в час по определенному Маршруту. В среднем выходит 10 запросов в минуту. Но ничто не мешает клиенту выполнить все 600 запросов в первую минуту часа. Может быть, это то, что нужно. Возможно, вы захотите достичь более равномерной нагрузки в течение часа. Установив значение поля `config.minute` равным 20, вы гарантируете, что ваши пользователи совершают не более 20 запросов в минуту и 600 запросов в час. Это допускает небольшие скачки на минутном интервале по сравнению с полностью усреднённым потоком в 10 запросов в минуту, но пользователи не могут использовать часовую квоту в течение одной минуты. Теперь им понадобится не менее 30 минут, для использования всех своих запросов. Клиенты получат дополнительные заголовки для каждого заданного временного интервала, например:

| HTTP Заголовок               |Значение|
|------------------------------|--------|
| X-RateLimit-Limit-hour       | 600    |
| X-RateLimit-Remaining-hour   | 595    |
| X-RateLimit-Limit-minute     | 20     |
| X-RateLimit-Remaining-minute | 16     |

Конечно, существует множество различных способов настройки ограничений запросов в зависимости от того, чего вы хотите достичь.

# Выводы

На этом я и закончу, думаю материала вполне достаточно для первой статьи об InterSystems API Manager. Мы использовали только один из более чем 40 плагинов. Вы можете сделать с IAM еще много чего интересного:

- Добавьте центральный механизм аутентификации для всех ваших API.
- Масштабируйте нагрузку с помощью балансировщика к нескольким Сервисам.
- Добавляйте новую функциональность и исправления ошибок для тестовой аудитории перед полным обновлением.
- Предоставляйте внутренним и внешним разработчикам специализированный веб портал, документирующий все API.
- Кэшируйте запросы для уменьшения времени ожидания ответа и снижения нагрузки на бэкэнд системы.

# Ссылки

- [Пресс-релиз](https://www.intersystems.com/news-events/news/news-item/intersystems-iris-data-platform-2019-2-introduces-api-management-capabilities/)
- [Вводное видео](https://www.youtube.com/watch?v=XI1oqKEwLL0)
- [Более подробное видео](https://www.youtube.com/watch?v=htfs8lesAEU)
- [Документация](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=AIAM)
- [Настройка балансировщика нагрузки](https://community.intersystems.com/post/using-intersystems-api-management-load-balance-api)
