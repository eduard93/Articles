# Шлюзы Java/.Net в интеграционных продукциях InterSystems IRIS

Шлюзы в InterSystems IRIS это механизм взаимодействия между ядром InterSystems IRIS и прикладным кодом на языках Java/.Net. С помощью шлюзов вы можете работать как с объектами Java/.NET из ObjectScript так и с объектами ObjectScript и глобалами из Java/.NET. Шлюзы могут быть запущены где угодно - локально, на удаленном сервере, в докере. 

В этой статье я покажу, как можно легко разработать и контейнеризовать интеграционную продукцию с .Net/Java кодом. А для взаимодействия с кодом на языках Java/.Net будем использовать [PEX](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=EPEX), предоставляющий возможность реализовать любой элемент интеграционной продукции на языках Java/.Net.

Для нашего примера мы разработаем интеграцию с [Apache Kafka](https://kafka.apache.org/).

# Архитектура

Apache Kafka – популярный брокер сообщений. В Kafka есть *тема* (topic) сообщения в которую *издатели* (publisher) пишут сообщения и есть *подписчики* (consumer) на темы, которые читают эти сообщения.

Сначала мы напишем Java бизнес-операцию которая будет пубиковать сообщения в Apache Kafka. Затем добавим бизнес-службу на языке C# которая будет сообщения читать, сохранять и передавать для дальнейшей обработки в InterSystems IRIS.

Наше решение будеть работать в докере и выглядит следующим образом:

![](https://raw.githubusercontent.com/intersystems-community/pex-demo/master/architecture.PNG)


# Java Gateway

Прежде всего, разработаем Бизнес-Операцию на Java для отправки сообщений в Apache Kafka. Код может быть написан в любой Java IDE и [выглядеть так](github.com/intersystems-community/pex-demo/blob/master/java/src/dc/rmq/KafkaOperation.java):
- Для разработки новой PEX бизнес-операции необходимо реализовать абстрактный класс `com.intersystems.enslib.pex.BusinessOperation`.
- Публичные свойства класса — это настройки нашего бизнес-хоста
- Метод `OnInit` используется для установления соединения с Apache Kafka и получения указателя на InterSystems IRIS
- `OnTearDown` используется для отключения от Apache Kafka (при остановке процесса).
- `OnMessage` получает сообщение [dc.KafkaRequest](https://github.com/intersystems-community/pex-demo/blob/master/iris/src/dc/KafkaRequest.cls) и отправляет его в Apache Kafka.

Теперь упакуем нашу бизнес-операцию в Docker контейнер.

Вот наш [докер-файл](https://github.com/intersystems-community/pex-demo/blob/master/java/Dockerfile):

```dockerfile
FROM openjdk:8 AS builder

ARG APP_HOME=/tmp/app

COPY src $APP_HOME/src

COPY --from=intersystemscommunity/jgw:latest /jgw/*.jar $APP_HOME/jgw/

WORKDIR $APP_HOME/jar/
ADD https://repo1.maven.org/maven2/org/apache/kafka/kafka-clients/2.5.0/kafka-clients-2.5.0.jar .
ADD https://repo1.maven.org/maven2/ch/qos/logback/logback-classic/1.2.3/logback-classic-1.2.3.jar .
ADD https://repo1.maven.org/maven2/ch/qos/logback/logback-core/1.2.3/logback-core-1.2.3.jar .
ADD https://repo1.maven.org/maven2/org/slf4j/slf4j-api/1.7.30/slf4j-api-1.7.30.jar .

WORKDIR $APP_HOME/src

RUN javac -classpath $APP_HOME/jar/*:$APP_HOME/jgw/* dc/rmq/KafkaOperation.java && \
    jar -cvf $APP_HOME/jar/KafkaOperation.jar dc/rmq/KafkaOperation.class

FROM intersystemscommunity/jgw:latest

COPY --from=builder /tmp/app/jar/*.jar $GWDIR/
```

Посмотрим, что здесь происходит (я предполагаю, что вы знакомы с [многоступенчатыми докер сборками](https://docs.docker.com/develop/develop-images/multistage-build/)):

```dockerfile
FROM openjdk:8 AS builder
```

JDK8 это базовый образ, в котором мы будем компилировать наше приложение. 

```dockerfile
ARG APP_HOME=/tmp/app
COPY src $APP_HOME/src
```

Копируем исходный код из папки `/src` в `/tmp/app`.

```dockerfile
COPY --from=intersystemscommunity/jgw:latest /jgw/*.jar $APP_HOME/jgw/
```
Копируем библиотеки Java Gateway в папку `/tmp/app/jgw`.

```dockerfile
WORKDIR $APP_HOME/jar/
ADD https://repo1.maven.org/maven2/org/apache/kafka/kafka-clients/2.5.0/kafka-clients-2.5.0.jar .
ADD https://repo1.maven.org/maven2/ch/qos/logback/logback-classic/1.2.3/logback-classic-1.2.3.jar .
ADD https://repo1.maven.org/maven2/ch/qos/logback/logback-core/1.2.3/logback-core-1.2.3.jar .
ADD https://repo1.maven.org/maven2/org/slf4j/slf4j-api/1.7.30/slf4j-api-1.7.30.jar .

WORKDIR $APP_HOME/src

RUN javac -classpath $APP_HOME/jar/*:$APP_HOME/jgw/* dc/rmq/KafkaOperation.java && \
    jar -cvf $APP_HOME/jar/KafkaOperation.jar dc/rmq/KafkaOperation.class
```

Все зависимости скачаны - вызываем `javac/jar` для компиляции jar файла. Для реальных проектов рекомендуется использовать полноценную систему сборки maven или gradle.

```dockerfile
FROM intersystemscommunity/jgw:latest

COPY --from=builder /tmp/app/jar/*.jar $GWDIR/
```

И, наконец, jar файлы копируются в базовый образ Java шлюза, который содержит все необходимые зависимости и обеспечивает запуск Java шлюза.

# .Net Gateway

Далее разработаем службу .Net, которая будет получать сообщения от Apache Kafka. Код может быть написан в любой .Net IDE и [выглядеть так](https://github.com/intersystems-community/pex-demo/blob/master/dotnet/KafkaConsumer.cs).

Особенности:
- Для разработки новой PEX бизнес-службы необходимо реализовать абстрактный класс `InterSystems.EnsLib.PEX.BusinessService`.
- Публичные свойства класса — это настройки нашего бизнес-хоста
- Метод `OnInit` используется для установления соединения с Apache Kafka, подписки на темы Apache Kafka и получения указателя на InterSystems IRIS
- `OnTearDown` используется для отключения от Apache Kafka (при остановке процесса)
- `OnMessage` получает сообщения из Apache Kafka и отправляет сообщение класса `Ens.StringContainer` в целевые бизнес-хосты продукции

Теперь упакуем нашу бизнес-операцию в Docker контейнер.

Вот наш [докер-файл](https://github.com/intersystems-community/pex-demo/blob/master/net/Dockerfile):

```dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:2.1 AS build

ENV ISC_PACKAGE_INSTALLDIR /usr/irissys
ENV GWLIBDIR lib
ENV ISC_LIBDIR ${ISC_PACKAGE_INSTALLDIR}/dev/dotnet/bin/Core21

WORKDIR /source
COPY --from=store/intersystems/iris-community:2020.2.0.211.0 $ISC_LIBDIR/*.nupkg $GWLIBDIR/

# copy csproj and restore as distinct layers
COPY *.csproj ./
RUN dotnet restore

# copy and publish app and libraries
COPY . .
RUN dotnet publish -c release -o /app

# final stage/image
FROM mcr.microsoft.com/dotnet/core/runtime:2.1
WORKDIR /app
COPY --from=build /app ./

# Configs to start the Gateway Server
RUN cp KafkaConsumer.runtimeconfig.json IRISGatewayCore21.runtimeconfig.json && \
    cp KafkaConsumer.deps.json IRISGatewayCore21.deps.json

ENV PORT 55556

CMD dotnet IRISGatewayCore21.dll $PORT 0.0.0.0
```

Посмотрим, что здесь происходит:
```dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:2.1 AS build
```
Используем образ .Net Core 2.1 SDK для сборки нашего приложения.
```dockerfile
ENV ISC_PACKAGE_INSTALLDIR /usr/irissys
ENV GWLIBDIR lib
ENV ISC_LIBDIR ${ISC_PACKAGE_INSTALLDIR}/dev/dotnet/bin/Core21

WORKDIR /source
COPY --from=store/intersystems/iris-community:2020.2.0.211.0 $ISC_LIBDIR/*.nupkg $GWLIBDIR/
```
Копируем библиотеки .Net Gateway из официального образа InterSystems IRIS:

```dockerfile
# copy csproj and restore as distinct layers
COPY *.csproj ./
RUN dotnet restore

# copy and publish app and libraries
COPY . .
RUN dotnet publish -c release -o /app
```
Компилируем нашу бизнес-операцию.

```dockerfile
FROM mcr.microsoft.com/dotnet/core/runtime:2.1
WORKDIR /app
COPY --from=build /app ./
```
Копируем библиотеки в финальный контейнер.

```dockerfile
RUN cp KafkaConsumer.runtimeconfig.json IRISGatewayCore21.runtimeconfig.json && \
    cp KafkaConsumer.deps.json IRISGatewayCore21.deps.json
```
В настоящее время шлюз .Net должен загружать все зависимости при запуске, поэтому мы информируем его обо всех возможных зависимостях.

```dockerfile
ENV PORT 55556

CMD dotnet IRISGatewayCore21.dll $PORT 0.0.0.0
```

Запускаем шлюз на порту `55556`, слушаем все сетевые интерфейсы.

Готово!

Вот полная конфигурация [docker-compose](https://github.com/intersystems-community/pex-demo/blob/master/docker-compose.yml), для запуска демо целиком (в том числе и UI для Apache Kafka, для просмотра сообщений).

# Запуск демо

Для запуска демо локально:

Установите:
- [docker](https://docs.docker.com/get-docker/)
- [docker-compose](https://docs.docker.com/compose/install/)
- [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)

Выполните:
```
git clone https://github.com/intersystems-community/pex-demo.git
cd pex-demo
docker-compose pull
docker-compose up -d
```

# Выводы

- В интеграционных продукциях InterSystems IRIS появилась возможность создавать любые элементы продукции на языках Java/.Net
- Код на Java/.Net возможно вызывать из InterSystems ObjectScript и наоборот, код на InterSystems ObjectScript из Java/.Net
- Генерация прокси классов больше не требуется
- Возможна как классическая поставка решения, так и поставка в Docker


# Ссылки

- Документация
    - [PEX](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=EPEX)
    - [Java](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=EJVG_using)
    - [.NET](https://docs.intersystems.com/irislatest/csp/docbook/Doc.View.cls?KEY=BGNT_using)
- [Репозиторий с демо](https://github.com/intersystems-community/pex-demo.git)
- [Вебинар про PEX](https://www.youtube.com/watch?v=ArSnkOthz8E) по материалам данной статьи
