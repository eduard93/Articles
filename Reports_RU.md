# InterSystems Reports

InterSystems Reports – модуль InterSystems IRIS и InterSystems IRIS for Health. Это современное решение для создания и публикации отчетов, которое включает в себя:
- Встроенную оперативную отчетность, которая может быть настроена как разработчиками отчетов, так и конечными пользователями.
- Точное форматирование, позволяющее создавать специализированные формы, например, макеты для счетов, документов и т.д.
- Макеты, обеспечивающие структуру для отображения как агрегированных, так и транзакционных данных.
- Позиционирование заголовков, колонтитулов, агрегированных и подробных данных, изображений и вложенных отчетов.
- Разнообразные типы отчетов.
- Публикация и распространение отчетов, включая экспорт в PDF, XLS, HTML, XML и другие форматы файлов, печать и архивирование для соблюдения нормативных требований.


InterSystems Reports состоит из:
- Дизайнера отчетов, в котором разработчики создают отчёты.
- Сервера отчетов, который предоставляет  пользователям доступ через браузер для запуска, планирования, фильтрации и изменения отчетов.

Эта статья посвящена серверной части InterSystems Reports и содержит руководство по запуску Report Server в контейнерах с сохранением конфигурации.

## Подготовка

Прежде чем мы начнем, для работы InterSystems Reports должно быть доступно следующее программное обеспечение:

- [Docker](https://docs.docker.com/engine/install/) - хотя InterSystems Reports может работать и без Docker (в операционных системах Windows, Mac, Linux), эта статья посвящена запуску Reports Server в Docker.
- (Опционально) [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git) - для клонирования репозитория, можно также загрузить его в виде [архива](https://github.com/eduard93/reports/archive/refs/heads/master.zip).
- (Опционально) [InterSystems Reports Designer](https://wrc.intersystems.com/) - для создания новых отчетов.

Дополнительно вам потребуется:
- [Авторизоваться](https://docs.intersystems.com/components/csp/docbook/DocBook.UI.Page.cls?KEY=PAGE_CONTAINERREGISTRY) в Docker registry `containers.intersystems.com`
- Лицензия InterSystems Reports

## План

Вот что необходимо сделать для запуска Reports Server:

1. Запустить Reports и InterSystems IRIS в режиме настройки, чтобы установить IRIS в качестве базы данных (не источника данных!) для Reports.
2. Настроить Reports и сохранить эту конфигурацию на хосте.
3. Запустить Reports с конфигурацией сохраненными данными.

# Первый запуск

Шаги 1-8 используют `docker-compose_setup.yml` в качестве конфигурационного файла docker-compose. Любые дополнительные команды docker-compose во время первого старта должны быть выполнены с аргументом `-f docker-compose_setup.yml`.

1. Склонируйте это репозиторий: `git clone https://github.com/eduard93/reports.git` или скачайте [архив](https://github.com/eduard93/reports/archive/refs/heads/master.zip).

2. Отредактируйте `config.properties` и укажите информацию о лицензии InterSystems Reports Server (User и Key). Если у вас их нет - обратитесь в InterSystems. Существует ряд [других свойств](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=GISR_server), описанных в документации. Обратите внимание, что InterSystems IRIS в данном случае является базой данных для InterSystems Reports, а не источником данных для отчетов (который мы добавим позже).

3. Запустите InterSystems Reports Server: `docker-compose -f docker-compose_setup.yml up -d`

4. Подождите, пока InterSystems Reports Server запустится (проверьте статус `docker-compose -f docker-compose_setup.yml logs reports`). Это может занять 5-10 минут. Сервер отчетов готов к работе, если в логах отображается: `reports_1 | Logi Report Server is ready for service.`.

5. Откройте сервер отчетов. (Пользователь/пароль: `admin`/`admin`). В случае, если отображается предупреждение об истекшем сроке действия лицензии, введите информацию о лицензии заново. В результате должен открыться портал InterSystems Reports:

![](https://user-images.githubusercontent.com/5127457/120627117-04bded00-c46c-11eb-99ad-5bc89e3bfb76.png)

# Сохранение конфигурации

Теперь, когда Reports запущен, нам нужно немного поправить конфигурацию и сохранить ее на хосте (обратите внимание, что конфигурация InterSystems IRIS сохраняется с помощью [Durable %SYS](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=ADOCK#ADOCK_iris_durable)).

6. Отметьте опцию `Enable Resources from Real Paths` в разделе `Administration` > `Configuration` > `Advanced` ([документация](https://devnet.logianalytics.com/hc/en-us/articles/1500009750141-Getting-and-Using-Resources-from-a-Real-Path)). Это позволит нам публиковать отчеты копируя их в папку `reports` в репозитории.

![](https://user-images.githubusercontent.com/5127457/120627668-90377e00-c46c-11eb-87c4-9745665c1857.png)

 7. Скопируйте конфигурацию InterSystems Reports на хост:
```bash
docker cp reports_reports_1:/opt/LogiReport/Server/bin .
docker cp reports_reports_1:/opt/LogiReport/Server/derby .
docker cp reports_reports_1:/opt/LogiReport/Server/font .
docker cp reports_reports_1:/opt/LogiReport/Server/history .
docker cp reports_reports_1:/opt/LogiReport/Server/style .
```
8. Остановите InterSystems Reports Server: `docker-compose -f docker-compose_setup.yml down`

# Второй запуск

Теперь мы готовы запустить Reports с хранимой конфигурацией - именно так он и будет работать.

9. Запустите InterSystems Reports Server без инициализации: `docker-compose up -d`

10. Создайте новую папку в `Public Reports` с `Real Path`: `/reports` ([документация](https://devnet.logianalytics.com/hc/en-us/articles/1500009750141-Getting-and-Using-Resources-from-a-Real-Path)). Для этого откройте `Public Reports` и выберите `Publish` > `From Server Machine`:

![](https://user-images.githubusercontent.com/5127457/120638494-da266100-c478-11eb-80a0-b89ba4e345db.png)

Создайте новую папку `/reports`:

![](https://user-images.githubusercontent.com/5127457/120638907-6fc1f080-c479-11eb-965d-8346d603ada2.png)

Откройте её:

![](https://user-images.githubusercontent.com/5127457/120638753-34bfbd00-c479-11eb-893f-0980ebe4f569.png)

В папке должен быть `catalog` (в котором настроено соединение с InterSystems IRIS) и два отчета (`reportset1` и `reportset2`). Запустите их (используйте кнопку `Run` для просмотра в браузере и `Advanced Run` для выбора между форматами HTML, PDF, Excel, Text, RTF, XML и PostScript). Вот как выглядят отчеты:

![](https://user-images.githubusercontent.com/5127457/120632926-333ec680-c472-11eb-8367-c1769fc344ce.png)

![](https://user-images.githubusercontent.com/5127457/120632973-42257900-c472-11eb-870f-9af4f77a5161.png)

InterSystems Reports поддерживает Unicode из коробки. В этом примере я использую один и тот же инстанс InterSystems IRIS в качестве источника данных, но в общем случае это может быть любой другой сервер InterSystems IRIS - подключение определяется в `catalog`. В этом демо используется набор данных HoleFoods (установлен с помощью `zpm "install samples-bi"`). Чтобы добавить новые источники данных InterSystems IRIS, создайте новый `catalog` в Designer. После этого создайте новые отчеты и экспортируйте все в новую подпапку в папке отчетов. Также можно указать адрес InterSystems Server, тогда InterSystems Designer загрузит отчёт напрямую на сервер. Контейнер InterSystems Server должен иметь сетевой доступ ко всем серверам IRIS - источникам данных.

Вот и все! Теперь, если вы хотите остановить Reports, выполните: `docker-compose stop`. А чтобы снова запустить Reports, выполните: `docker-compose up -d`. После перезапуска все отчеты по-прежнему доступны.

## Отладка

Все журналы хранятся в папке `/opt/LogiReport/Server/logs`. В случае возникновения ошибок добавьте ее в `volumes`, перезапустите Reports и воспроизведите ошибку.

В документации описано, как настроить [уровень журналирования](https://documentation.logianalytics.com/rsg17u1/content/html/config/config_log.htm?Highlight=logging). Если загрузка Reports прерывается до запуска пользовательского интерфейса, измените файл `LogConfig.properties`, расположенный в папке `bin`:

```
logger.Engine.level = TRIVIAL
logger.DHTML.level = TRIVIAL
logger.Designer.level = TRIVIAL
logger.Event.level = TRIVIAL
logger.Error.level = TRIVIAL
logger.Access.level = TRIVIAL
logger.Manage.level = TRIVIAL
logger.Debug.level = TRIVIAL
logger.Performance.level = TRIVIAL
logger.Dump.level = TRIVIAL
```

# API

Чтобы встроить отчеты в ваше веб-приложение, используйте [Embedded API](https://documentation.logianalytics.com/logiinfov12/content/embedded-reports-api.htm).

Другие [доступные API](https://documentation.logianalytics.com/logireportserverguidev17/content/html/api/wkapi_srv.htm).

# Заключение

InterSystems Reports представляет собой надежное современное решение для создания отчетов. InterSystems Reports Server предоставляет конечным пользователям доступ через браузер для запуска, планирования, фильтрации и изменения отчетов. InterSystems Reports Server может быть запущен как на хосте, так и в среде Docker.

# Ссылки

- [Документация](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=GISR_intro)
- [Репозиторий](https://github.com/eduard93/reports)
- [Логирование](https://documentation.logianalytics.com/rsg17u1/content/html/config/config_log.htm?Highlight=logging)


