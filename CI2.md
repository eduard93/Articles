# GitLab для Continuous Delivery проекта на технологиях InterSystems: Контейнеры


Эта статья - продолжение [статьи](https://habr.com/company/intersystems/blog/354158/) про организацию процессов [Continuous Integration](https://ru.wikipedia.org/wiki/Непрерывная_интеграция) / Continuous Delivery, автоматизирующих сборку, тестирование и доставку приложений применимо к решениям на платформе InterSystems. 
Рассмотрим такие темы как:

- Контейнеры 101
- Контейнеры на разных этапах цикла разработки ПО
- Continuous Delivery с контейнерами

# Контейнеры 101

Про контейнеры и контейнеризацию написано много статей и книг, поэтому тут я сделаю небольшое вступление, которое, впрочем, не претендует на какую-либо окончательность. Итак, начнём. 

Контейнеры, технически, это метод виртуализации, при котором ядро операционной системы поддерживает несколько изолированных экземпляров пространства пользователя (контейнеров), вместо одного. Наглядно это выглядит так:


![Docker vs VM](http://www.settlersoman.com/wp-content/uploads/2016/10/Docker_vs_VM.jpg)

При этом важно отметить что контейнеры - это не виртуальные машины, вот [хорошая статья](https://blog.docker.com/2016/03/containers-are-not-vms/) об их различиях.

## Преимущества контейнеров

Существует ряд преимуществ использования контейнеров:

- Портативность
- Эффективность
- Изоляция
- Лёгкость
- Неизменность (Immutability)

### Портативность

Контейнер содержит в себе приложение вместе со всеми зависимостями. Это позволяет легко запускать приложения в различных окружениях, таких как физические серверы, виртуальные машины, окружения для тестирования и продуктовые окружения, облака.

Также портативность состоит в том, что после того, как Docker образ собран и он работает правильно, он будет работать где угодно, если там работает Docker т.е. на серверах Windows, Linux и MacOS.

### Эффективность

При работе с приложениями виртуальных машин действительно ли нужны процессы ОС, системные программы и т.п.? Как правило нет, интересен только процесс вашего приложения. Контейнеры обеспечивают именно это: в контейнере запускаются только те процессы, которые  явно нужны, и ничего более. Поскольку контейнеры не требуют отдельной операционной системы, они используют меньше ресурсов. Виртуальная машина часто занимает несколько гигабайт, контейнер же может быть размером всего в несколько мегабайт, что позволяет запускать гораздо больше контейнеров, чем виртуальных машин на одном сервере. 
Поскольку контейнеры имеют более высокий уровень утилизации серверов, требуется меньше аппаратного обеспечения, что приводит к сокращению затрат.

### Изоляция

Контейнеры изолируют приложение от всех остальных процессов, и, хотя несколько контейнеров могут работать на одном сервере, они могут быть полностью независимы друг от друга. Любое взаимодействие между контейнерами должно быть явно объявлено. Если один контейнер выходит из строя, он не влияет на другие контейнеры и может быть быстро перезапущен. Безопасность также повышается благодаря такой изоляции. Например, использование уязвимости веб-сервера на хосте может дать злоумышленнику доступ ко всему серверу, но в случае контейнера злоумышленник получит доступ только к контейнеру веб-сервера.

### Лёгкость

Поскольку для контейнеров не требуется отдельная ОС, их можно запустить, остановить или перезагрузить в считанные секунды, что ускорит все связанные процессы, в том числе и процессы Continuous Integration. Вы можете начать разрабатывать быстрее и не тратить время на настройку окружения.

### Неизменность (Immutability)

Неизменяемая инфраструктура состоит из неизменяемых компонентов, которые заменяются для каждого развертывания, а не обновляются. Неизменность уменьшает несогласованность и позволяет легко и быстро реплицировать и перемещаться между различными состояниями вашего приложения. [Подробнее о неизменности](https://blog.codeship.com/immutable-infrastructure/).

## Новые возможности

Все эти преимущества позволяют управлять инфраструктурой и приложениями по-новому.

### Оркестрация

Виртуальные машины и сервера с течением времени часто обретают "индивидуальность", что приводит ко множеству как правило неприятных сюрпризов в будущем. Одним из вариантов  решения данной проблемы является [Инфраструктура как код](https://en.wikipedia.org/wiki/Infrastructure_as_Code) (IoC) - управление инфраструктурой с помощью описательной модели, используя систему контроля версий.

При использовании IoC команда развертывания окружения всегда приводит целевую среду в одну и ту же конфигурацию, независимо от исходного состояния среды. Это достигается путем автоматической настройки существующей среды или путем пересоздания среды с нуля.

С помощью IoC разработчики вносят изменения в описание среды. Впоследствии целевые среды модифицируются до нового состояния. Если в среду необходимо внести изменения, редактируется её описание.

Все это гораздо проще делать с контейнерами. Выключение контейнера и запуск нового занимает несколько секунд, а выделение новой виртуальной машины занимает несколько минут.

### Масштабирование

Инструменты оркестрации также могут обеспечивать горизонтальное масштабирование на основе текущей нагрузки. Возможно запускать столько контейнеров, сколько требуется в настоящее время требуется, и масштабировать приложение соответствующе. Всё это также  снижает затраты на работу приложения.

# Контейнеры на разных этапах жизненного цикла ПО

Рассмотрим преимущества контейнеров на различных этапах жизненного цикла ПО.

![Жизненный цикл ПО](http://inf-w.ru/wp-content/uploads/2015/07/7.png)

## Разработка

Самое главное преимущество - это простота начала разработки. После [установки Docker](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=ADOCK#ADOCK_additional_driver), достаточно выполнить две команды: `docker pull` для загрузки образа и `docker run` для его запуска. Все зависимости уже разрешены на этапе сборки приложения.

## Отладка

Все окружения консистентны и существуют их определения, кроме того легко развернуть необходимое окружение. Достаточно сделать  `docker pull` интересующего контейнера и запустить его.

## Тестирование / QA

В случае возникновения ошибки, проблемное окружения и условия воспроизведения ошибки можно передать с контейнером. Все изменения инфраструктуры «задокументированы». Уменьшается число переменных – версий библиотек, фреймворков, ОС… Возможен запуск нескольких контейнеров для параллелизации тестов.

## Доставка

Использование контейнеров позволяет проводить сборку один раз, кроме того для использования контейнеров необходим высокий уровень автоматизации процессов сборки и развёртывания приложения. Доставка приложения с помощью контейнеров может быть более безопасна за счёт дополнительной изоляции.

# Continuous Delivery

Перейдём от теории к практике. Вот общий вид нашего решения по автоматизации сборки и доставки:

![CD](https://habrastorage.org/webt/yv/dn/qc/yvdnqcgyrlswgft4-zhnm-2mabc.png)

Можно выделить три основных этапа:
- Сборка
- Доставка
- Запуск

## Сборка

В предыдущей статье сборка была инкрементальной - мы считали разницу между текущей средой и новой кодовой базой и изменяли нашу среду так, чтобы она соответствовала новой кодовой базе. С контейнерами каждая сборка является полной. Результатом сборки является образ Docker Image, который можно запускать где угодно.

## Доставка

После того, как наш образ собран и протестирован, он загружается в Docker Registry - специализированное приложение для размещения Docker Image. Там он может заменить предыдущий образ с тем же именем (тегом). Например, из-за нового коммита в ветку master мы собрали новый образ (`MyProject/MyApp:master`), и если тесты пройдены, мы можем обновить образ в Docker Registry и все, кто скачивает `MyProject/MyApp:master` получат новую версию.

## Запуск

Наконец, образ необходимо запустить. Система CD, такая как GitLab, может этим управлять как напрямую, так и с помощью специализированного оркестратора, но процесс, в общем, одинаков - некоторые образы запускаются, периодически проверяются на работоспособность и обновляются, если новая версия становится доступной.

Посмотрите [вебинар](https://blog.docker.com/2018/02/ci-cd-with-docker-ee/), объясняющий эти этапы.

Альтернативно, с точки зрения коммита:

![](https://community.intersystems.com/sites/default/files/inline/images/asset-3.png)

В нашей конфигурации непрерывной доставки мы:
- Коммитим код в репозиторий GitLab
- Собераем образ
- Тестируем его
- Публикуем новый образ в нашем Docker Registry
- Обновим старый контейнером на новую версию из Docker Registry

Для этого нам понадобятся:

- Docker
- Docker Registry
- Зарегистрированный домен (необязательно, но желательно)
- GUI инструменты (по желанию)

 
### Docker

Прежде всего, нам нужно запустить Docker. Я бы посоветовал начать с одного сервера с распространенной версией Linux, такой как Ubuntu, RHEL или Suse. Не рекомендую начинать с таких дистрибутивов как CoreOS, RancherOS и т. д. - они нацелены не на начинающих. [Не забудьте переключить драйвер хранилища на devicemapper](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=ADOCK#ADOCK_additional_driver).

Если же говорить о широкомасштабных развертываниях, то с помощью инструментов оркестрации, таких как Kubernetes, Rancher или Swarm, можно автоматизировать большинство задач, но мы не будем обсуждать их (по крайней мере, в рамках этой статьи).

### Docker Registry

Это первый контейнер, который нам необходимо запустить, это автономное приложение, которое позволяет хранить и распространять образы Docker. Нужно использовать Docker Registry, если вы хотите:
- Контролировать, где хранятся ваши образы
- Владеть сервером распространения образов
- Интегрировать хранение и распространение образов в процесс разработки

Вот [документация](https://docs.docker.com/registry/) по запуску и настройке Docker Registry.

### Подключение Docker Registry и GitLab

Чтобы подключить Docker Registry к GitLab, необходимо запустить Docker Registry с [поддержкой HTTPS](https://docs.docker.com/registry/configuration/). Я использую Let's Encrypt для получения сертификатов, и я следовал [этой инструкции](https://gist.github.com/PieterScheffers/63e4c2fd5553af8a35101b5e868a811e), для получения сертификата. Убедившись, что Docker Registry доступен по HTTPS (вы можете проверить это в браузере), следуйте [этим инструкциям](https://docs.gitlab.com/ee/administration/container_registry.html) по подключению Docker Registry к GitLab. Эти инструкции отличаются в зависимости от вашей установки GitLab и необходимой конфигурации. В моем случае настройка заключалась в добавлении сертификата Docker Registry и ключа в `/etc/gitlab/ssl`, а этих строк в `/etc/gitlab/gitlab.rb`:

```
registry_external_url 'https://docker.domain.com'
gitlab_rails ['registry_api_url'] = "https://docker.domain.com"
```

После [переконфигурации GitLab](https://docs.gitlab.com/ee/administration/restart_gitlab.html) появилась новая вкладка Registry, на которой предоставлена информация о том, как правильно назвать созданные образы, чтобы они тут появились.

![](https://community.intersystems.com/sites/default/files/inline/images/snimok_35.png)


 
### Домен

В нашей конфигурации непрерывной доставки мы будем автоматически создавать образ на каждую ветку, и если образ проходит тесты, то он публикуется в Docker Registry и запускается автоматически, поэтому наше приложение будет автоматически развёрнуто из всех ветках, например:

- Несколько веток фич по адресу `<featureName>.docker.domain.com`
- Тестовая версия на `master.docker.domain.com`
- Опытная версия на `preprod.docker.domain.com`
- Продуктовая версия на `prod.docker.domain.com`

Для этого нам нужно доменное имя и wildcard DNS-запись, которая перенаправляет запросы к `* .docker.domain.com` на IP-адрес `docker.domain.com`. Как вариант можно использовать различные порты.

### Nginx

Поскольку у нас есть несколько окружений, нам необходимо автоматически перенаправлять запросы к поддоменам в правильный контейнер. Для этого мы можем использовать Nginx в качестве обратного прокси. Вот [руководство](https://cloud.google.com/community/tutorials/nginx-reverse-proxy-docker).


### GUI Инструменты

Чтобы начать работу с контейнерами, вы можете использовать либо командную строку, либо один из графических интерфейсов. Есть много доступных, например:

- Rancher
- MicroBadger
- Portainer
- Simple Docker UI
- ...

Они позволяют создавать контейнеры и управлять ими из GUI вместо CLI. Вот как выглядит Rancher:

![Rancher](https://community.intersystems.com/sites/default/files/inline/images/snimok_36.png)

 
### GitLab runner 

Как и раньше, для выполнения скриптов на других серверах нам нужно установить GitLab runner. Подробно этот вопрос описан в [предыдущей статье](https://habr.com/company/intersystems/blog/354158/).

Обратите внимание, что нужно использовать executor Shell, а не Docker. Executor Docker используется, когда нужно что-то изнутри образа, например, при создании приложение для Android в java контейнере, и нужен только apk. В нашем случае артефакт - контейнер целиком, и для этого нужен executor Shell.


# Конфигурация Continuous Delivery

Теперь, когда все необходимые компоненты настроены, можно приступать к созданию конфигурации непрерывной доставки.

## Сборка

Во-первых, нам нужно собрать образ.

Наш код, как и всегда, хранится в репозитории, конфигурация CD в `gitlab-ci.yml`, но кроме того (для повышения безопасности) мы будем хранить несколько файлов, относящихся к образу, на сервере сборки.

### GitLab.xml

Содержит код хуков для CD. Он был разработан в [предыдущей статье](https://habr.com/company/intersystems/blog/354158/) и доступен на [GitHub](https://github.com/intersystems-ru/GitLab). Это небольшая библиотека для загрузки кода, запуска различных хуков и тестового кода. Предпочтительней использовать подмодули git, чтобы включить этот проект или что-то подобное в ваш репозиторий. Подмодули лучше, потому что легче поддерживать их в актуальном состоянии. Еще одна альтернатива - создать релиз на GitLab и загрузить его с помощью команды ADD уже при сборке.

### iris.key

Лицензионный ключ. Он может быть загружен во время сборки контейнера, а не храниться на сервере. Небезопасно хранить ключ в репозитории.

### pwd.txt

Файл, содержащий пароль по умолчанию. Опять же, хранить пароль в репозитории довольно небезопасно.

### load_ci.script

Скрипт, который:
- Включает ОС авторизацию в InterSystems IRIS
- Загружает GitLab.xml
- Инициализирует настройки GitLab хуков
- Загружает код


```

set sc = ##Class(Security.System).Get("SYSTEM",.Properties)
write:('sc) $System.Status.GetErrorText(sc)
set AutheEnabled = Properties("AutheEnabled")
set AutheEnabled = $zb(+AutheEnabled,16,7)
set Properties("AutheEnabled") = AutheEnabled
set sc = ##Class(Security.System).Modify("SYSTEM",.Properties)
write:('sc) $System.Status.GetErrorText(sc)
zn "USER"
do ##class(%SYSTEM.OBJ).Load(##class(%File).ManagerDirectory() _ "GitLab.xml","cdk")
do ##class(isc.git.Settings).setSetting("hooks", "MyApp/Hooks/")
do ##class(isc.git.Settings).setSetting("tests", "MyApp/Tests/")
do ##class(isc.git.GitLab).load()
halt
```

Обратите внимание, что первая строка намеренно оставлена пустой. Если этот начальный скрипт всегда один и тот же, вы можете просто сохранить его в репозитории.

## gitlab-ci.yml

Теперь, перейдём к конфигурации непрерывной доставки:

```
build image:
  stage: build
  tags:
    - test
  script:
    - cp -r /InterSystems/mount ci
    - cd ci
    - echo 'SuperUser' | cat - pwd.txt load_ci.script > temp.txt
    - mv temp.txt load_ci.script
    - cd ..
    - docker build --build-arg CI_PROJECT_DIR=$CI_PROJECT_DIR -t docker.domain.com/test/docker:$CI_COMMIT_REF_NAME .
```

Что здесь происходит?

Прежде всего, поскольку [процесс сборки образа](https://docs.docker.com/engine/reference/commandline/build/) может получить доступ только к подкаталогам базового каталога - в нашем случае корневого каталога репозитория, нужно скопировать «секретный» каталог (в котором есть `GitLab.xml`, `iris.key`, `pwd.txt` и `load_ci.skript`) в клонированный репозиторий.

Далее, для доступа к терминалу требуется пользователь/пароль, поэтому мы добавим их к `load_ci.script` (для этого нам и нужна  пустая строка в начале `load_ci.script`).

Наконец, мы создаем Docker Image и называем его: `docker.domain.com/test/docker:$CI_COMMIT_REF_NAME`

где `$CI_COMMIT_REF_NAME` - это имя ветви. Обратите внимание: первая часть тега образа должна совпадать с именем репозитория в GitLab, чтобы ее можно было увидеть на вкладке Registry (более полные инструкции по корректному тегированию доступны там же).

### Dockerfile

Docker Image создаётся с помощью [Dockerfile](https://docs.docker.com/engine/reference/builder/), вот он:

```
FROM docker.intersystems.com/intersystems/iris:2018.1.1.611.0

ENV SRC_DIR=/tmp/src
ENV CI_DIR=$SRC_DIR/ci
ENV CI_PROJECT_DIR=$SRC_DIR

COPY ./ $SRC_DIR

RUN cp $CI_DIR/iris.key $ISC_PACKAGE_INSTALLDIR/mgr/ \
 && cp $CI_DIR/GitLab.xml $ISC_PACKAGE_INSTALLDIR/mgr/ \
 && $ISC_PACKAGE_INSTALLDIR/dev/Cloud/ICM/changePassword.sh $CI_DIR/pwd.txt \
 && iris start $ISC_PACKAGE_INSTANCENAME \
 && irissession $ISC_PACKAGE_INSTANCENAME -U%SYS < $CI_DIR/load_ci.script \
 && iris stop $ISC_PACKAGE_INSTANCENAME quietly
```

Выполняются следующие действия:
- За основу берём контейнер InterSystems IRIS.
- Прежде всего, копируем наш репозиторий (и «секретный» каталог) внутрь контейнера.
- Копируем лицензионный ключ и `GitLab.xml` в каталог `mgr`.
- Меняем пароль на значение из `pwd.txt`. Обратите внимание, что `pwd.txt` удаляется при этой операции.
- Запускается InterSystems IRIS и выполняется `load_ci.script`.
- InterSystems IRIS останавливается.

Вот частичный лог сборки:

```
Running with gitlab-runner 10.6.0 (a3543a27)
  on docker 7b21e0c4
Using Shell executor...
Running on docker...
Fetching changes...
Removing ci/
Removing temp.txt
HEAD is now at 5ef9904 Build load_ci.script
From http://gitlab.eduard.win/test/docker
   5ef9904..9753a8d  master     -> origin/master
Checking out 9753a8db as master...
Skipping Git submodules setup
$ cp -r /InterSystems/mount ci
$ cd ci
$ echo 'SuperUser' | cat - pwd.txt load_ci.script > temp.txt
$ mv temp.txt load_ci.script
$ cd ..
$ docker build --build-arg CI_PROJECT_DIR=$CI_PROJECT_DIR -t docker.eduard.win/test/docker:$CI_COMMIT_REF_NAME .
Sending build context to Docker daemon  401.4kB

Step 1/6 : FROM docker.intersystems.com/intersystems/iris:2018.1.1.611.0
 ---> cd2e53e7f850
Step 2/6 : ENV SRC_DIR=/tmp/src
 ---> Using cache
 ---> 68ba1cb00aff
Step 3/6 : ENV CI_DIR=$SRC_DIR/ci
 ---> Using cache
 ---> 6784c34a9ee6
Step 4/6 : ENV CI_PROJECT_DIR=$SRC_DIR
 ---> Using cache
 ---> 3757fa88a28a
Step 5/6 : COPY ./ $SRC_DIR
 ---> 5515e13741b0
Step 6/6 : RUN cp $CI_DIR/iris.key $ISC_PACKAGE_INSTALLDIR/mgr/  && cp $CI_DIR/GitLab.xml $ISC_PACKAGE_INSTALLDIR/mgr/  && $ISC_PACKAGE_INSTALLDIR/dev/Cloud/ICM/changePassword.sh $CI_DIR/pwd.txt  && iris start $ISC_PACKAGE_INSTANCENAME  && irissession $ISC_PACKAGE_INSTANCENAME -U%SYS < $CI_DIR/load_ci.script  && iris stop $ISC_PACKAGE_INSTANCENAME quietly
 ---> Running in 86526183cf7c
.
Waited 1 seconds for InterSystems IRIS to start
This copy of InterSystems IRIS has been licensed for use exclusively by:
ISC Internal Container Sharding
Copyright (c) 1986-2018 by InterSystems Corporation
Any other use is a violation of your license agreement

%SYS>
1

%SYS>
Using 'iris.cpf' configuration file

This copy of InterSystems IRIS has been licensed for use exclusively by:
ISC Internal Container Sharding
Copyright (c) 1986-2018 by InterSystems Corporation
Any other use is a violation of your license agreement

1 alert(s) during startup. See messages.log for details.
Starting IRIS

Node: 39702b122ab6, Instance: IRIS

Username:
Password:

Load started on 04/06/2018 17:38:21
Loading file /usr/irissys/mgr/GitLab.xml as xml
Load finished successfully.

USER>

USER>

[2018-04-06 17:38:22.017] Running init hooks: before

[2018-04-06 17:38:22.017] Importing hooks dir /tmp/src/MyApp/Hooks/

[2018-04-06 17:38:22.374] Executing hook class: MyApp.Hooks.Global

[2018-04-06 17:38:22.375] Executing hook class: MyApp.Hooks.Local

[2018-04-06 17:38:22.375] Importing dir /tmp/src/

Loading file /tmp/src/MyApp/Tests/TestSuite.cls as udl

Compilation started on 04/06/2018 17:38:22 with qualifiers 'c'
Compilation finished successfully in 0.194s.

Load finished successfully.

[2018-04-06 17:38:22.876] Running init hooks: after

[2018-04-06 17:38:22.878] Executing hook class: MyApp.Hooks.Local

[2018-04-06 17:38:22.921] Executing hook class: MyApp.Hooks.Global
Removing intermediate container 39702b122ab6
 ---> dea6b2123165
[Warning] One or more build-args [CI_PROJECT_DIR] were not consumed
Successfully built dea6b2123165
Successfully tagged docker.domain.com/test/docker:master
Job succeeded
```

## Запуск

У нас есть образ, запустим его. В случае веток фич можно просто уничтожить старый контейнер и запустить новый. В случае продуктовой среды мы можем запустить сначала временный контейнер и заменить контейнер среды в случае успешного прохождения тестов.

Сначала скрипт удаления старого контейнера.

```
destroy old:
  stage: destroy
  tags:
    - test
  script:
    - docker stop iris-$CI_COMMIT_REF_NAME || true
    - docker rm -f iris-$CI_COMMIT_REF_NAME || true
```

Этот скрипт уничтожает запущенный контейнер и всегда завершается успешно (по умолчанию Docker возвращает ошибку, при попытке  остановить/удалить несуществующий контейнер).

После этого мы запускаем новый контейнер и регистрируем его как окружение.

```
run image:
  stage: run
  environment:
    name: $CI_COMMIT_REF_NAME
    url: http://$CI_COMMIT_REF_SLUG.docker.eduard.win/index.html
  tags:
    - test
  script:
    - docker run -d
      --expose 52773
      --volume /InterSystems/durable/$CI_COMMIT_REF_SLUG:/data
      --env ISC_DATA_DIRECTORY=/data/sys
      --env VIRTUAL_HOST=$CI_COMMIT_REF_SLUG.docker.eduard.win
      --name iris-$CI_COMMIT_REF_NAME
      docker.eduard.win/test/docker:$CI_COMMIT_REF_NAME
      --log $ISC_PACKAGE_INSTALLDIR/mgr/messages.log
 ```
 
[Контейнер Nginx](https://cloud.google.com/community/tutorials/nginx-reverse-proxy-docker) автоматически перенаправляет запросы с использованием переменной среды `VIRTUAL_HOST` на указанный порт - в данном случае 52773. 

Так как необходимо хранить некоторые данные (пароли, %SYS, данные приложения) на хосте в InterSystems IRIS существует функциональность [Durable %SYS](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=ADOCK#ADOCK_iris_durable), позволяющая хранить  на хосте такие данные, как:

- `iris.cpf` - основной файл конфигурации.
- Директорию `/csp` с файлами веб-приложений.
- `/httpd/httpd.conf` с конфигурацией приватного сервера Apache.
- Директорию `/mgr`, в которой хранятся:
  - Базы данных `IRISSYS`, `IRISTEMP`, `IRISAUDIT`, `IRIS`, `USER`.
  - `IRIS.WIJ`.
  - Директорию `/journal` хранящую журналы.
  - Директорию `/temp` для временных файлов.
  - Логи `messages.log`, `journal.log`, `SystemMonitor.log`.

Для включения Durable %SYS указывается аргумент `volume` монтирующий директорию хоста и переменная `ISC_DATA_DIRECTORY` устанавливающая директорию для хранения файлов Durable %SYS. Эта директория не должна существовать, она создастся автоматически.

Таким образом архитектура нашего контейнеризированного приложения следующая:

![архитектура контейнеризированного приложения](https://habrastorage.org/webt/61/0q/ze/610qzezlqdilphfkapj5vac9t84.png)

Чтобы собрать такое приложение, мы, как минимум, должны создать одну дополнительную базу данных (чтобы сохранить код приложения) и создать её маппинг в область приложения. Я использовал область `USER` для хранения данных приложения, поскольку эта область по умолчанию добавлена в Durable %SYS. Код приложения хранится в контейнере чтобы можно было его обновлять.

Исходя из вышесказанного, [%Installer](https://habr.com/company/intersystems/blog/268767/) должен:

- Создать область/базу данных `APP`
- Загрузить код в область `APP`
- Создать маппинг классов нашего приложения в область `USER`
- Выполнить прочую настройку (я создал CSP веб-приложение и REST веб-приложение)

Код %Installer

```
Class MyApp.Hooks.Local
{

Parameter Namespace = "APP";

/// See generated code in zsetup+1^MyApp.Hooks.Local.1
XData Install [ XMLNamespace = INSTALLER ]
{
<Manifest>

<Log Text="Creating namespace ${Namespace}" Level="0"/>
<Namespace Name="${Namespace}" Create="yes" Code="${Namespace}" Ensemble="" Data="IRISTEMP">
<Configuration>
<Database Name="${Namespace}" Dir="/usr/irissys/mgr/${Namespace}" Create="yes" MountRequired="true" Resource="%DB_${Namespace}" PublicPermissions="RW" MountAtStartup="true"/>
</Configuration>

<Import File="${Dir}Form" Recurse="1" Flags="cdk" IgnoreErrors="1" />
</Namespace>
<Log Text="End Creating namespace ${Namespace}" Level="0"/>

 
<Log Text="Mapping to USER" Level="0"/>
<Namespace Name="USER" Create="no" Code="USER" Data="USER" Ensemble="0">
<Configuration>
<Log Text="Mapping Form package to USER namespace" Level="0"/>
<ClassMapping From="${Namespace}" Package="Form"/>
<RoutineMapping From="${Namespace}" Routines="Form" />
</Configuration>

<CSPApplication  Url="/" Directory="${Dir}client" AuthenticationMethods="64" IsNamespaceDefault="false" Grant="%ALL" Recurse="1" />
</Namespace>

</Manifest>
}

/// This is a method generator whose code is generated by XGL.
/// Main setup method
/// set vars("Namespace")="TEMP3"
/// do ##class(MyApp.Hooks.Global).setup(.vars)
ClassMethod setup(ByRef pVars, pLogLevel As %Integer = 0, pInstaller As %Installer.Installer) As %Status [ CodeMode = objectgenerator, Internal ]
{
     Quit ##class(%Installer.Manifest).%Generate(%compiledclass, %code, "Install")
}

/// Entry point
ClassMethod onAfter() As %Status
{
    try {
        write "START INSTALLER",!
        set vars("Namespace") = ..#Namespace
        set vars("Dir") = ..getDir()
        set sc = ..setup(.vars)
        write !,$System.Status.GetErrorText(sc),!
        
        set sc = ..createWebApp()
    } catch ex {
        set sc = ex.AsStatus()
        write !,$System.Status.GetErrorText(sc),!
    }
    quit sc
}

/// Modify web app REST
ClassMethod createWebApp(appName As %String = "/forms") As %Status
{
    set:$e(appName)'="/" appName = "/" _ appName
    #dim sc As %Status = $$$OK
    new $namespace
    set $namespace = "%SYS"
    if '##class(Security.Applications).Exists(appName) {
        set props("AutheEnabled") = $$$AutheUnauthenticated
        set props("NameSpace") = "USER"
        set props("IsNameSpaceDefault") = $$$NO
        set props("DispatchClass") = "Form.REST.Main"
        set props("MatchRoles")=":" _ $$$AllRoleName
        set sc = ##class(Security.Applications).Create(appName, .props)
    }
    quit sc
}

ClassMethod getDir() [ CodeMode = expression ]
{
##class(%File).NormalizeDirectory($system.Util.GetEnviron("CI_PROJECT_DIR"))
}

}
```

Отмечу, что для создания базы данных не на хосте я использую директорию `/usr/irissys/mgr`, так как вызов `##class(%File).ManagerDirectory()` возвращает путь к директории для Durable %SYS.

## Тесты

Теперь запустим тесты.

```
test image:
  stage: test
  tags:
    - test
  script:
    - docker exec iris-$CI_COMMIT_REF_NAME irissession iris -U USER "##class(isc.git.GitLab).test()"
```

## Доставка

Тесты пройдены, опубликуем наш образ в Docker Registry.

```
publish image:
  stage: publish
  tags:
    - test
  script:
    - docker login docker.domain.com -u user -p pass
    - docker push docker.domain.com/test/docker:$CI_COMMIT_REF_NAME
```

Логин/пароль можно передать с использованием [секретных переменных](https://docs.gitlab.com/ee/ci/variables/#secret-variables).

Теперь образ отображается на GitLab.

![](https://community.intersystems.com/sites/default/files/inline/images/snimok_37.png)

И другие разработчики могут скачать его из Docker Registry. На вкладке Environments все наши окружения доступны для просмотра:
     
# Выводы

В этой серии статей рассмотрены общие подходы к непрерывной интеграции. Автоматизация сборки, тестирования и доставки вашего приложения на платформах InterSystems возможна и легко реализуема. 

Применение технологий контейнеризации поможет оптимизировать процессы разработки и развёртывания приложений. Устранение несоответствий между окружениями позволяет упростить тестирование и отладку. Оркестрация позволяет создавать масштабируемые приложения.

# Ссылки

- [Предыдущая статья](https://habr.com/company/intersystems/blog/354158/)
- [Серия статей на InterSystems Community](https://community.intersystems.com/post/continuous-delivery-your-intersystems-solution-using-gitlab-index)
- [Код](http://gitlab.eduard.win/test/docker)
- [Код на GitHub](https://github.com/intersystems-ru/GitLab)
