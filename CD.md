# GitLab для Continuous Delivery проекта на технологиях InterSystems 

В данной статье хотелось бы рассказать про организацию процессов [Continuous Integration](https://ru.wikipedia.org/wiki/Непрерывная_интеграция) / Continuous Delivery, автоматизирующих сборку, тестирование и доставку приложений применимо к решениям на платформе InterSystems. 
Рассмотри такие темы как:

- Git 101
- Методологии разработки (Git flow)
  - GitHub flow
  - GitLab flow
- GitLab
- GitLab CI

## Git 101

Несмотря на то, что основная тема это CD, [Git](https://ru.wikipedia.org/wiki/Git#Преимущества_и_недостатки), а точнее ряд фундаментальных принципов лежащих в его основе оказали большое влияние на процессы разработки программного обеспечения, поэтому начнём с краткого описания основных терминов Git.

Git — система контроля и управления версиями, в основе которой такие идеи, как:

- Нелинейная разработка - несмотря на то, что версии приложения выходят последовательно, разработка каждой версии ведётся параллельно несколькими разработчиками, которые могут одновременно править одни и те же части приложения.
- Распределённая разработка - разработчик независим от центрального сервера и может разрабатывать в своём локальном окружении без каких-либо проблем.
- Слияние - реализация предыдущих двух пунктов приводит к тому, что существует несколько корректных версий проекта одновременно и часто нужно их объединять в единую версию правды.

Git не был первой системой контроля версий в которой данные идеи были реализованы, однако Git популяризовал данные идеи и существенно упростил их практическое применение.

### Репозиторий (Repository)

Место, где хранятся и поддерживаются какие-либо данные.

- “Физически” — папка в ОС
- Хранит файлы и папки
- Хранит историю их изменения

Локальный репозиторий — репозиторий, расположенный на локальном компьютере 
Удалённый репозиторий — репозиторий, находящийся на удалённом сервере

### Коммит (Commit)

Зафиксированное состояние репозитория.
Хранит отличие (Diff) от какого-либо другого коммита который называется родительским.

Родителей бывает:
- 0 – у первого коммита.
- 1 – как правило бывает именно так.
- 2 – для слияния изменений.
- 3+ - для слияния изменений (но так делать не надо).

### Ветка (Branch)

Указатель на коммит.
Всегда можно посмотреть на его историю до первого коммита. Например, ветка `master`:

![Ветка](https://community.intersystems.com/sites/default/files/inline/images/risunok1_0.png)


#### Дерево коммитов

Коммиты образуют деревья - графическое представление репозитория. Рассмотрим самый простой линейный вариант, когда к двум существующим коммитам с картинки выше добавили ещё три коммита:

![ветка](https://community.intersystems.com/sites/default/files/inline/images/risunok2.png)

Теперь пример дерева посложнее: два разработчика работают одновременно и, чтобы не мешать друг другу каждый из них работает в своей ветке. Это выглядит так:

![ветка](https://community.intersystems.com/sites/default/files/inline/images/risunok3.png)

Через некоторое время им надо объединить изменения, для этого существуют **запросы на слияние** - запросы на объединение двух состояний репозитория в одно новое состояние. В нашем случае запрос на слияние ветки `develop` в ветку `master`. После того как запрос был рассмотрен и одобрен и слияние произошло репозиторий выглядит следующим образом:

![ветка](https://community.intersystems.com/sites/default/files/inline/images/risunok4.png)

После чего разработка продолжается:

![ветка](https://community.intersystems.com/sites/default/files/inline/images/risunok5.png)

### Выводы - Git 101

- Git — система контроля и управления версиями.
- Репозиторий (Repository) — место, где хранятся и поддерживаются какие-либо данные.
- Коммит (Commit) — зафиксированное состояние репозитория.
- Ветка (Branch) — указатель на коммит.
- Запрос на слияние (Pull request или Merge request) – запрос на слияние двух состояний репозитория.


## Методологии разработки (Git flows)

Методология разработки на основе Git это серия подходов к разработке программного обеспечения использующих Git в качестве основы разработки. Существует множество методологий разработки, рассмотрим две из них:

- GitHub flow
- GitLab flow

### GitHub flow

GitHub flow является, наверное, одной из самых простых методологий разработки на основе Git. Вот она:

- Для каждой новой фичи создаём новую ветку, называемую веткой фичи (feature branch)
- Изменения коммитятся в новую ветку
- После того как изменения закомичены отправляется запрос на слияние
- Запрос на слияние обсуждается и дорабатывается
- Запрос на слияние одобряется

Кроме того существует ряд правил:

- Ветка `master` всегда находится в работоспособном состоянии
- В ветке `master` не идёт разработка
- Разработка ведётся в отдельных ветках
- Ветка `master` == промышленное окружение
- Промышленное окружение обновляется с каждым изменением ветки `master`

Окружение это сконфигурированный ресурс где исполняется код вашего приложения. Это может быть сервер, виртуальная машина или даже контейнер.

Вот как это выглядит:

![GitHub](https://community.intersystems.com/sites/default/files/inline/images/risunok6.png)

Подробно про GitHub flow на Хабре уже [писали](https://habrahabr.ru/post/346066/) и не [раз](https://habrahabr.ru/post/189046/). 

### GitLab flow

Если вы не готовы автоматически обновлять код на промышленном окружении, GitLab flow предоставляет единение GitHub flow с несколькими окружениями. Вот как это работает: разработка ведётся аналогично GitHub flow - в отдельных ветках, которые также сливаются в `master`, но содержимое ветки `master` развёрнуто на тестовом сервере. Дополнительно у вас есть ветки окружений содержимое которых соответствует содержимому ваших окружений. Обычно существует три окружения но их может быть больше или меньше в зависимости от ваших требований:

- Тестовое окружение == ветка `master`
- Опытное окружение  == ветка `preprod` 
- Промышленное окружение == ветка `prod`

Выглядит процесс разработки так:

![](https://community.intersystems.com/sites/default/files/inline/images/risunok7.png)

Подробно про GitLab flow на Хабре тоже [писали](https://habrahabr.ru/company/softmart/blog/316686/).

## Выводы - Методологии разработки (Git flows)

Существует ряд методологий разработки основанных на Git - от простых до сложных. Выберите ту методологию которая с одной стороны не переусложнена, с другой обеспечивает надлежащий уровень контроля за состоянием вашего проекта.

## GitLab Workflow

GitLab Workflow - методология разработки программного обеспечения, которая затрагивает не только этапы разработки но весь жизненный цикл продукта от идеи до отзывов пользователей. Вот как он выглядит:

- Идея: новая функциональность начинается с идеи.
- Задача (Issue): наиболее эффективным способом обсуждения идеи является создание для нее задачи. Другие разработчики помогут улучшить идею, предложить пути её реализации.
- План: как только обсуждение задачи придет к чему-либо, пришло время писать код. Но, во-первых, нам необходимо расставить приоритеты и организовать наш рабочий процесс. Для этого есть этапы, канбан доска, дата исполнения и ответственный.
- Код: теперь всё готово для написания кода.
- Коммит: как только мы довольны нашим кодом, можно перенести его коммитить. GitLab flow был подробно описан выше.
- Тест: запустите скрипты с помощью GitLab CI, чтобы скомпилировать и протестировать ваше приложение.
- Ревью: как только код работает, тесты и сборки проходят успешно, можно сделать ревью кода.
- Опытная эксплуатация: теперь пришло время развернуть новую версию приложения в опытном окружении, чтобы проверить, все ли работает, или нужны ещё правки.
- Промышленная эксплуатация: пришло время для развертывания в промышленном окружении
- Обратная связь: что ещё нуждается в улучшении.

![](https://community.intersystems.com/sites/default/files/inline/images/idea-to-production-10-steps.png)

Подробно каждый этап описан на [сайте GitLab](https://about.gitlab.com/2016/10/25/gitlab-workflow-an-overview/), я ограничусь описанием нескольких этапов.

### Задачи и планирование

Начальные этапы GitLab Workflow сосредоточены на задаче - новой функциональности, ошибке или каком-либо другом отдельном объёме работ. Задача имеет несколько целей, таких как:

- Управление: в задаче есть даты создания и исполнения, ответственный, временные затраты, приоритет и т. д., для отслеживания решения проблемы.
- Административный: проблема является частью этапа разработки, а также канбан доски, что позволяет формировать план работ и следить за его выполнением.
- Разработка: в задаче можно обсуждать пути её решения, коммиты, ветки, запросы на слияние и другие задачи также могут быть связаны с задачей.

Этап планирования позволяет группировать задачи по их приоритету, этапу, статусу.

![](https://community.intersystems.com/sites/default/files/inline/images/issue-board.gif)

Разработку мы обсудили выше, следуйте любой методологии разработки git. После того, как мы разработали нашу новую функциональность и слили её в ветку `master` - что дальше?

## Continuous Delivery (непрерывная доставка)

Непрерывная доставка - это подход к разработке программного обеспечения, в котором команды разрабатывают программное обеспечение в коротких спринтах, гарантируя, что новая версия приложения может быть выпущена в любое время. Данный подход автоматизирует сборку, тестирование и доставку программного обеспечения. Этот подход помогает снизить затраты и риски внесения изменений, позволяя получать быстрые инкрементные обновления для приложений в промышленной эксплуатации. Для непрерывной доставки важно настроить простой и воспроизводимый процесс доставки.

### Непрерывная доставка в GitLab

В GitLab конфигурации непрерывной доставки определяется отдельно для каждого репозитория и хранится в файле конфигурации YAML в корне репозитория.

- Конфигурация непрерывной доставки - это серия последовательных этапов.
- Каждый этап имеет один или несколько скриптов, которые выполняются параллельно.

#### Скрипт

Определяет одно действие и какие условия должны выполняться для его запуска:

- Что делать (выполнить команду ОС, запустить контейнер)?
- Когда запускать скрипт:
    - Триггер (например коммит в ветку `master`)?
    - Запускать ли скрипт, если предыдущие этапы не удались (по умолчанию нет)?
    - Запуск вручную или автоматически?
- В какой среде запускать скрипт?
- Какие артефакты сохраняются после выполнения (они загружаются из окружения в GitLab для быстрого доступа)?

**Окружение** - это настроенный сервер или контейнер, в котором вы можете запускать скрипты.

**Runner** - служба, выполняет скрипты в определенном окружении. Подключен к GitLab и выполняют скрипты по мере необходимости.
Runner может быть развернут на сервере, в контейнере или даже на локальном компьютере разработчика.

#### Как происходит непрерывная доставка?

- Новый коммит загружается в репозиторий.
- GitLab проверяет конфигурацию непрерывной доставки.
- Конфигурация непрерывной доставки содержит все возможные скрипты для всех случаев, поэтому они фильтруются до набора скриптов, которые должны выполняться для этого нового коммита (например, скрипт для ветки `master` срабатывает только в случае коммита в ветку `master`). Этот набор называется **запуском** (pipeline).
- Запуск выполняется в соответствующем окружении, результаты выполнения сохраняются и доступны на GitLab.

Вот пример запуска:

![](https://community.intersystems.com/sites/default/files/inline/images/risunok1_2.png)

Он состоит из четырех этапов, выполняемых последовательно

- Load загружает код на сервер
- Test запускает юнит тесты
- Package состоит из двух скриптов, выполняемых параллельно:
  - Сборка клиента
  - Экспорт серверного кода в один "xml" (в основном для информационных целей)
- Deploy перемещает собранный клиент в каталог веб-сервера.

Как мы видим, каждый скрипт выполнился успешно, если бы один из скриптов завершился неудачно, следующие скрипты не будут выполняться:

![](https://community.intersystems.com/sites/default/files/inline/images/snimok_33.png)

Если открыть скрипт, можно увидеть почему он закончился с ошибкой:

```
Running with gitlab-runner 10.4.0 (857480b6)
 on test runner (ab34a8c5)
Using Shell executor...
Running on gitlab-test...
Fetching changes...
Removing diff.xml
Removing full.xml
Removing index.html
Removing tests.html
HEAD is now at a5bf3e8 Merge branch '4-versiya-1-0' into 'master'
From http://gitlab.eduard.win/test/testProject
 * [new branch] 5-versiya-1-1 -> origin/5-versiya-1-1
 a5bf3e8..442a4db master -> origin/master
 d28295a..42a10aa preprod -> origin/preprod
 3ac4b21..7edf7f4 prod -> origin/prod
Checking out 442a4db1 as master...
Skipping Git submodules setup
$ csession ensemble "##class(isc.git.GitLab).loadDiff()"

[2018-03-06 13:58:19.188] Importing dir /home/gitlab-runner/builds/ab34a8c5/0/test/testProject/

[2018-03-06 13:58:19.188] Loading diff between a5bf3e8596d842c5cc3da7819409ed81e62c31e3 and 442a4db170aa58f2129e5889a4bb79261aa0cad0

[2018-03-06 13:58:19.192] Variable modified
var=$lb("MyApp/Info.cls")

Load started on 03/06/2018 13:58:19
Loading file /home/gitlab-runner/builds/ab34a8c5/0/test/testProject/MyApp/Info.cls as udl
Load finished successfully.

[2018-03-06 13:58:19.241] Variable items
var="MyApp.Info.cls"
var("MyApp.Info.cls")=""

Compilation started on 03/06/2018 13:58:19 with qualifiers 'cuk /checkuptodate=expandedonly'
Compiling class MyApp.Info
Compiling routine MyApp.Info.1
ERROR: MyApp.Info.cls(version+2) #1003: Expected space : '}' : Offset:14 [zversion+1^MyApp.Info.1]
 TEXT: 	quit, "1.0" }
Detected 1 errors during compilation in 0.010s.

[2018-03-06 13:58:19.252] ERROR #5475: Error compiling routine: MyApp.Info.1. Errors: ERROR: MyApp.Info.cls(version+2) #1003: Expected space : '}' : Offset:14 [zversion+1^MyApp.Info.1]
 > ERROR #5030: An error occurred while compiling class 'MyApp.Info'
ERROR: Job failed: exit status 1
```

Ошибка компиляции стала причиной неудачного выполнения скрипта.

Перейдём от теории к практике.

## Установка GitLab

Будем устанавливать GitLab на нашем собственном сервере. Впрочем можно пользоваться GitLab.com. Существует много способов установки GitLab: из исходников, из пакета, в контейнере. Выберите тот способ который вам нравится и следуйте [инструкции по установке](https://about.gitlab.com/installation/).

Требования:

- Отдельный сервер - GitLab достаточно ресурсоёмкое веб-приложение, поэтому лучше держать его на отдельном сервере (4 Gb RAM, 2 CPU).
- ОС Linux.
- (Опционально но рекомендуется) Домен - необходим для обеспечения безопасности и запуска pages.

### Конфигурация

Прежде всего настройте [почтовые уведомления](https://docs.gitlab.com/omnibus/settings/smtp.html).

Далее, рекомендую установить Pages. Как было сказано выше, артефакты от выполнения скриптов могут быть загружены на GitLab. Пользователи могут загрузить их, но часто бывает полезно открыть их напрямую в браузере и для этого нужно установить pages. 

Зачем нужны pages:

- Для вики или набора статичных страниц связанных с проектом.
- Для просмотра html артефактов.
- [И ряд других причин](https://docs.gitlab.com/ce/user/project/pages/#explore-gitlab-pages).

Так как в HTML может быть добавлено автоматическое перенаправление при загрузке страницы, то можно направлять пользователя на страницу с результатами юнит-тестов.

```
ClassMethod writeTestHTML()
{
  set text = ##class(%Dictionary.XDataDefinition).IDKEYOpen($classname(), "html").Data.Read()
  set text = $replace(text, "!!!", ..getURL())
  
  set file = ##class(%Stream.FileCharacter).%New()
  set name = "tests.html"
  do file.LinkToFile(name)
  do file.Write(text)
  quit file.%Save()
}

ClassMethod getURL()
{
  set url = "http://host:57772"
  set url = url _ $system.CSP.GetDefaultApp("%SYS")
  set url = url_"/%25UnitTest.Portal.Indices.cls?Index="_ $g(^UnitTest.Result, 1) _ "&$NAMESPACE=" _ $zconvert($namespace,"O","URL")
  quit url
}

XData html
{
<html lang="en-US">
  <head>
    <meta charset="UTF-8"/>
    <meta http-equiv="refresh" content="0; url=!!!"/>
    <script type="text/javascript">window.location.href = "!!!"</script>
  </head>
  <body>
    If you are not redirected automatically, follow this <a href='!!!'>link to tests</a>.
  </body>
</html>
}
```

Кстати, я столкнулся с багом при использовании pages (Ошибка 502 при просмотре артефактов) - вот [решение](https://gitlab.com/gitlab-com/infrastructure/issues/3064).

### Подключение окружений к GitLab

Для запуска CD скриптов необходимы окружения - настроенные серверы для запуска вашего приложения. Для начала предположим, что у вас установлен Linux-сервер с платформой InterSystems (скажем, InterSystems IRIS, но всё будет работать с Caché или Ensemble). Для соединения окружения с GitLab нужно:

1. [Установить GitLab runner](https://docs.gitlab.com/runner/).
2. [Зарегистрировать](https://docs.gitlab.com/runner/register/index.html) GitLab runner в GitLab.
3. Разрешить пользователю `gitlab-runner` вызывать InterSystems IRIS.

Важное замечание по установке GitLab runner - НЕ клонируйте сервер после установки GitLab runner. Результаты непредсказуемы и нежелательны.
Поподробнее о шагах 2, 3.

#### Зарегистрировать GitLab runner в GitLab.

После запуска команды: `sudo gitlab-runner register`

Будет предложено несколько опций на выбор, и хотя большинство шагов довольно просто, некоторые из них стоит прокомментировать:

- Please enter the gitlab-ci token for this runner

Доступно несколько токенов: для всей системы (доступен в настройках администрирования) либо для одного проекта (доступен в настройках проекта).
Когда вы подключаете runner для CD конкретного проекта, нужно указать токен именно этого проекта.

- Please enter the gitlab-ci tags for this runner (comma separated)

В CD конфигурации можно фильтровать, какие скрипты выполняются окружениях с определёнными тегами. Поэтому в самом простом случае укажите один тег, который будет названием окружения.

- Please enter the executor: ssh, docker+machine, docker-ssh+machine, kubernetes, docker, parallels, virtualbox, docker-ssh, shell

Вне зависимости от того используете ли вы docker, выбирайте shell для запуска скриптов.

#### Разрешить пользователю `gitlab-runner` вызывать InterSystems IRIS.

После подключения к GitLab нужно разрешить юзеру `gitlab-runner` вызывать InterSystems IRIS. Для этого:

1. Пользователь `gitlab-runner` должен иметь права для вызова `irissession` либо `csession`. Для этого его нужно добавить в группу `irisusr` либо `cacheusr` командой: `usermod -a -G cacheusr gitlab-runner`
2. В InterSystems IRIS [создайте пользователя](http://docs.intersystems.com/latest/csp/docbook/DocBook.UI.Page.cls?KEY=GSQL_privileges) `gitlab-runner` и предоставьте ему права для выполнения скриптов CD (запись в БД и т.д.)
3. [Включите ОС аутентификацию](http://docs.intersystems.com/latest/csp/docbook/DocBook.UI.Page.cls?KEY=GCAS_intro_authe_os).

Вместо пунктов 2 и 3 можно использовать другие подходы, например передачу пользователя/пароля, но вариант с ОС аутентификацией представляется мне более предпочтительным.

## Конфигурация CD

Итак, приступим к написанию конфигурации непрерывной доставки. Для начала опишем окружения и план.

### Окружения

У нас есть несколько окружений и соответствующих им веток:

| Окружени     | Ветка   | Доставка          | Кто может коммитить    | Кто может сливать      |
|--------------|---------|-------------------|------------------------|------------------------|
| Тестовое     | master  | Автоматическая    | Разработчики, Владельцы| Разработчики, Владельцы|
| Опытное      | preprod | Автоматическая    | Никто                  | Владельцы              |
| Промышленное | prod    | По нажатию кнопки | Никто                  | Владельцы              |


### План работ

1. Постановка задачи, разработка и автоматическое тестирование
   - Владелец ставит задачу
   - Разработчик создаёт feature-ветку и коммитит туда код решающий задачу
   - Разработчик сливает feature-ветку в master
   - Новый код автоматически доставляется в тестовое окружение и тестируется
2. Доставка в опытное окружение
   - Разработчик создаёт запрос на слияние кода master в preprod
   - Владелец одобряет слияние кода из master в preprod
   - Новый код автоматически доставляется в опытное окружение
3. Доставка в промышленное окружение
   - Разработчик (или владелец) создаёт запрос на слияние кода из preprod в prod
   - Владелец одобряет слияние кода preprod в prod
   - Владелец нажимает кнопку «Развенуть»
   - Новая версия приложения разворачивается в автоматическом окружении

Тоже самое в графическом виде:

![](https://community.intersystems.com/sites/default/files/inline/images/risunok3_0.png)

### Приложение

Наше тестовое приложение состоит из двух частей:

- REST API на платформе InterSystems
- Клиентское веб приложение (HTML + JS + CSS)

### Этапы (Stages)

Из приведенного выше плана мы можем выделить этапы, которые нам необходимо определить в нашей конфигурации непрерывной доставки:

- Load загружает код на сервер
- Test запускает юнит тесты
- Package собирает клиент
- Deploy перемещает собранный клиент в каталог веб-сервера.

Начнём составлять конфигурацию в файле `.gitlab-ci.yml`:

```
stages:
  - load
  - test
  - package
  - deploy
```

### Скрипты

Следующая часть конфигурации - скрипты. [Документация](https://docs.gitlab.com/ee/ci/yaml/README.html).

#### Load

Начнём со скрипта `load server` который загружает серверный код.

```
load server:
  environment:
    name: test
    url: http://test.hostname.com
  only:
    - master
  tags:
    - test
  stage: load
```

Что здесь происходит?

- `load server` это название скрипта.
- Далее описывается окружение в котором скрипт запускается.
- `only: master` - запускает скрипт только если новый коммит в ветке `master`.
- `tags: test` - отправляет скрипт на запуск в runner, у которого есть такой тэг.
- `stage` - определяет к какому этапу принадлежит скрипт.
- `script` - определяет какую команду выполнять. В данном случае вызывается метод `load` класса `isc.git.GitLab`.

Теперь надо создать класс `isc.git.GitLab`. Все точки входа в него должны выглядеть так:

```
ClassMethod method()
{
    try {
        // code
        halt
    } catch ex {
        write !,$System.Status.GetErrorText(ex.AsStatus()),!
        do $system.Process.Terminate(, 1)
    }
}
```

Такой метод может завершится только двумя способами
- Остановкой текущего процесса командой `halt`, что считается успешным завершением скрипта.
- Вызовом `$system.Process.Terminate` - и выходом из процесса с ошибкой, что приводит к ошибке при выполнении скрипта.

Вот код загрузки:

```
/// Do a full load
/// do ##class(isc.git.GitLab).load()
ClassMethod load()
{
    try {
        set dir = ..getDir()
        do ..log("Importing dir " _ dir)
        do $system.OBJ.ImportDir(dir, ..getExtWildcard(), "c", .errors, 1)
        throw:$get(errors,0)'=0 ##class(%Exception.General).%New("Load error")
        
        halt
    } catch ex {
        write !,$System.Status.GetErrorText(ex.AsStatus()),!
        do $system.Process.Terminate(, 1)
    }
}
```

Этот метод вызывает два других метода:

- getExtWildcard - метод получения списка расширений которые нужно загружать.
- getDir - метод получения папки с репозиторием.

Но как мы можем получить директорию с репозиторием?

Когда GitLab выполняет скрипт, он определяет ряд [переменных окружения](https://docs.gitlab.com/ce/ci/variables/README.html). Одна из них `CI_PROJECT_DIR` - полный путь до корня репозитория. Так его можно получить в методе `getDir`:

```
ClassMethod getDir() [ CodeMode = expression ]
{
##class(%File).NormalizeDirectory($system.Util.GetEnviron("CI_PROJECT_DIR"))
}
```

#### Test

Вот скрипт запуска тестов:

```
tests:
  environment:
    name: test
    url: http://test.hostname.com
  only:
    - master
  tags:
    - test
  stage: test
  script: csession IRIS "##class(isc.git.GitLab).test()"
  artifacts:
    paths:
      - tests.html
```

Что изменилось? Конечно, название и код скрипта, но ещё добавился `artifacts`. Артефакт - это набор файлов и папок, которые прикрепляются к скрипту после его успешного завершения. В нашем случае после завершения тестов мы можем сгенерировать  HTML-страницу перенаправляющую пользователя на результаты тестов.

Обратите внимание на схожесть скриптов тестирования и загрузки. Части скриптов, такие как окружения, одинаковые во всех скриптах могут быть выделены в отдельные блоки. Определим тестовое окружение:

```
.env_test: &env_test
  environment:
    name: test
    url: http://test.hostname.com
  only:
    - master
  tags:
    - test
 ```
Теперь наш скрипт `tests` выглядит так:

```
tests:
  <<: *env_test
  script: csession IRIS "##class(isc.git.GitLab).test()"
  artifacts:
    paths:
      - tests.html
 ```

Теперь напишем соответствующий серверный код, вызывающий [юнит тесты](http://docs.intersystems.com/latest/csp/docbook/DocBook.UI.Page.cls?KEY=TUNT) ([статья на хабре](https://habrahabr.ru/company/intersystems/blog/263677/)):

```
/// do ##class(isc.git.GitLab).test()
ClassMethod test()
{
    try {
        set tests = ##class(isc.git.Settings).getSetting("tests")
        if (tests'="") {
            set dir = ..getDir()
            set ^UnitTestRoot = dir
            
            $$$TOE(sc, ##class(%UnitTest.Manager).RunTest(tests, "/nodelete"))
            $$$TOE(sc, ..writeTestHTML())
            throw:'..isLastTestOk() ##class(%Exception.General).%New("Tests error")
        }
        halt
    } catch ex {
        do ..logException(ex)
        do $system.Process.Terminate(, 1)
    }
}
```

Настройка `tests` - путь относительно корня репозитория до папки с тестами. Если он не задан, то пропускаем юнит тесты.

Метод `writeTestHTML` используется для генерации страницы-ссылки на результаты юнит-тестов.

```
ClassMethod writeTestHTML()
{
    set text = ##class(%Dictionary.XDataDefinition).IDKEYOpen($classname(), "html").Data.Read()
    set text = $replace(text, "!!!", ..getURL())
    
    set file = ##class(%Stream.FileCharacter).%New()
    set name = ..getDir() _  "tests.html"
    do file.LinkToFile(name)
    do file.Write(text)
    quit file.%Save()
}

ClassMethod getURL()
{
    set url = ##class(isc.git.Settings).getSetting("url")
    set url = url _ $system.CSP.GetDefaultApp("%SYS")
    set url = url_"/%25UnitTest.Portal.Indices.cls?Index="_ $g(^UnitTest.Result, 1) _ "&$NAMESPACE=" _ $zconvert($namespace,"O","URL")
    quit url
}

ClassMethod isLastTestOk() As %Boolean
{
    set in = ##class(%UnitTest.Result.TestInstance).%OpenId(^UnitTest.Result)
    for i=1:1:in.TestSuites.Count() {
        #dim suite As %UnitTest.Result.TestSuite
        set suite = in.TestSuites.GetAt(i)
        return:suite.Status=0 $$$NO
    }
    quit $$$YES
}

XData html
{
<html lang="en-US">
<head>
<meta charset="UTF-8"/>
<meta http-equiv="refresh" content="0; url=!!!"/>
<script type="text/javascript">
window.location.href = "!!!"
</script>
</head>
<body>
If you are not redirected automatically, follow this <a href='!!!'>link to tests</a>.
</body>
</html>
}
```

### Package

Весь клиент это одна веб-страница, обращающаяся к REST API:

```
<html>
<head>
<script type="text/javascript">
function initializePage() {
  var xhr = new XMLHttpRequest();
  var url = "${CI_ENVIRONMENT_URL}:57772/MyApp/version";
  xhr.open("GET", url, true);
  xhr.send();
  xhr.onloadend = function (data) {
    document.getElementById("version").innerHTML = "Version: " + this.response;
  };
  
  var xhr = new XMLHttpRequest();
  var url = "${CI_ENVIRONMENT_URL}:57772/MyApp/author";
  xhr.open("GET", url, true);
  xhr.send();
  xhr.onloadend = function (data) {
    document.getElementById("author").innerHTML = "Author: " + this.response;
  };
}
</script>
</head>
<body  onload="initializePage()">
<div id = "version"></div>
<div id = "author"></div>
</body>
</html>
```

И для того чтобы её "собрать", нужно заменить переменную `${CI_ENVIRONMENT_URL}` на её значение. В реальных приложениях всё будет намного сложнее, но это просто пример. Вот скрипт:

```
package client:
  <<: *env_test
  stage: package
  script: envsubst < client/index.html > index.html
  artifacts:
    paths:
      - index.html
 ```
 
### Deploy

Наконец, развернём клиент переместив `index.html` в корень веб-сервера.

```
deploy client:
  <<: *env_test
  stage: deploy
  script: cp -f index.html /var/www/html/index.html
```

Вот и всё!

### Несколько окружений

Что делать если нужно выполнить один и тот же скрипт в нескольких окружениях? Части скриптов также могут быть именованы, поэтому их тоже можно выделять в отдельные блоки. Вот пример конфигурации загружающей код в тестовое и опытное окружения:

```
stages:
  - load
  - test

.env_test: &env_test
  environment:
    name: test
    url: http://test.hostname.com
  only:
    - master
  tags:
    - test
    
.env_preprod: &env_preprod
  environment:
    name: preprod
    url: http://preprod.hostname.com
  only:
    - preprod
  tags:
    - preprod

.script_load: &script_load
  stage: load
  script: csession IRIS "##class(isc.git.GitLab).loadDiff()"

load test:
  <<: *env_test
  <<: *script_load

load preprod:
  <<: *env_preprod
  <<: *script_load
```

Так мы можем избавиться от дублирования кода.

Полная конфигурация CD доступна [здесь](https://github.com/intersystems-ru/GitLab/blob/master/.gitlab-ci.yml) и она соответствует первоначальному плану перемещения кода между тестовым, опытным и промышленным окружениями.

## Выводы

Непрерывная доставка - это подход к разработке программного обеспечения, в котором команды разрабатывают программное обеспечение в коротких спринтах, гарантируя, что новая версия приложения может быть выпущена в любое время. Данный подход автоматизирует сборку, тестирование и доставку программного обеспечения. Этот подход помогает снизить затраты и риски внесения изменений, позволяя получать быстрые инкрементные обновления для приложений в промышленной эксплуатации.

Ваши приложения на платформе InterSytems легко могут быть адаптированы к процессам непрерывной доставки, увеличивая скорость разработки, улучшая качество кода и устраняя ошибки вызванные ручным управлением процессами сборки, тестирования и доставки. 

## Ссылки

- [Тестовый репозиторий](http://gitlab.eduard.win/test/testProject)
- [Репозиторий с хуками GitLab](https://github.com/intersystems-ru/GitLab)
- [GitLab workflow](https://about.gitlab.com/2016/10/25/gitlab-workflow-an-overview/)
- [Инструкции по установке GitLab](https://about.gitlab.com/installation/)
- [Документация скриптов](https://docs.gitlab.com/ee/ci/yaml/README.html)
- [Переменные окружения](https://docs.gitlab.com/ce/ci/variables/README.html)
- [Документация по CD](https://about.gitlab.com/features/gitlab-ci-cd/)
