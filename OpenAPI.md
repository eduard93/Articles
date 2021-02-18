Продолжая [мою предыдущую статью](https://habr.com/ru/company/intersystems/blog/345532/) про разработку REST API на платформе InterSystems IRIS, в этой статье я хотел бы рассказать о подходе от спецификации (spec-first) при разработке REST API на платформе InterSystems IRIS.

В то время как традиционная разработка REST API (от кода, code-first) включает:

- Написание кода
- Написание REST брокера
- Документирование REST API

При написании REST API от спецификации вы будете делать все те же шаги, но в обратном порядке. Мы начинаем с написания спецификации, которая также является и документацией, генерируем из нее REST приложение и, наконец, пишем код бизнес-логики.
</break>
Это удобно, потому что:

- Теперь у вас всегда есть актуальная документация для разработчиков, использующих ваше REST API
- Спецификации Open API (Swagger), могут быть импортированы в различные инструменты, такие как: генераторы клиентов/серверов, управление API, юнит-тестирование и многое другое.
- Улучшается архитектура API. При написании API от кода разработчик может легко потерять контроль над общей архитектурой API, однако при разработке от спецификации разработчик вынужден взаимодействовать с API с позиции потребителя API, что обычно помогает в проектировании архитектуры.
- Скорость разработки - так как большая часть кода генерируется автоматически, вам не придется его писать, осталось только разработать бизнес-логику.
- Более быстрые циклы обратной связи - потребители API могут сразу же получить представление об API, и им будет проще вносить предложения, просто модифицируя спецификацию.

Давайте разработаем наше API начиная от спецификации!

**План**

1. Создание спецификации
    - В Docker
    - Локально
    - Онлайн
2. Загрузка спецификации в InterSystems IRIS
    - API Management REST API
    - Рутина `^REST`
    - Классы `%REST`
3. Что произошло со спецификацией?
4. Бизнес-логика
5. Дальнейшая разработка
6. Особенности
    - Параметры
    - CORS
7. Загрузка спецификации в InterSystems API Manager
8. Дополнительные инструменты

*Создание спецификации*

Первое что нужно сделать - это написать спецификацию. InterSystems IRIS поддерживает спецификацию Open API версии 2 (перевод [документации Swagger](https://swagger.io/docs/specification/about/)):


> **Спецификация OpenAPI** (ранее Swagger) - формат описания REST API. OpenAPI позволяет вам описать спецификацию API, включая:
> - Пути (`/users`) и операции над ними (`GET /users`, `POST /users`)
> - Входящие параметры операций
> - Результаты вызова операций
> - Методы аутентификации
> - Контактную информацию, лицензию, условия использования и прочую информацию.
>
> Описание API может быть представлено в форматах YAML и JSON. Формат легко изучить и он человеко- и машинно- читаем. Спецификация OpenAPI опубликована на GitHub: [OpenAPI 2.0 Specification](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md)

Мы будем использовать редактор Swagger для написания нашего API. Существует несколько способов использования Swagger:

- [Онлайн](https://editor.swagger.io/)
- В Docker: `docker run -d -p 8080:8080 swaggerapi/swagger-editor`
- [Локально](https://swagger.io/docs/open-source-tools/swagger-editor/)

После запуска Swagger откройте браузер:

![](https://community.intersystems.com/sites/default/files/inline/images/images/image-20191118205553-1.png)

В левой части Swagger вы редактируете описание API, а в правой сразу видите сгенерированную документацию и инструменты тестирования.

Загрузим нашу спецификацию (в формате [YAML](https://en.wikipedia.org/wiki/YAML)). Это простейшее API с одним GET запросом, возвращающим случайное число между `min` и `max`. 

Спецификация Math API:
```yaml
swagger: "2.0"
info:
  description: "Math"
  version: "1.0.0"
  title: "Math REST API"
host: "localhost:52773"
basePath: "/math"
schemes:
  - http
paths:
  /random/{min}/{max}:
    get:
      x-ISC_CORS: true
      summary: "Генерация случайного целого числа"
      description: "Генерация случайного целого числа между min и max"
      operationId: "getRandom"
      produces:
      - "application/json"
      parameters:
      - name: "min"
        in: "path"
        description: "Минимум"
        required: true
        type: "integer"
        format: "int32"
      - name: "max"
        in: "path"
        description: "Максимум"
        required: true
        type: "integer"
        format: "int32"
      responses:
        200:
          description: "OK"
```
Рассмотрим эту спецификацию поподробнее.

Начинается спецификация с базового описания нашего API и используемой версии Open API.

```yaml
swagger: "2.0"
info:
  description: "Math"
  version: "1.0.0"
  title: "Math REST API"
```

Далее идёт адрес API (сервер, порт), протокол доступа (http, https) и корневой путь:
```yaml
host: "localhost:52773"
basePath: "/math"
schemes:
  - http
```

Далее идут относительные пути и операции (клиент, таким образом, должен будет обращаться к нашему API по адресу http://localhost:52773/math/random/:min/:max):
```yaml
paths:
  /random/{min}/{max}:
```

Далее идёт информация о запросе:

```yaml
    get:
      x-ISC_CORS: true
      summary: "Генерация случайного целого числа"
      description: "Генерация случайного целого числа между min и max"
      operationId: "getRandom"
      produces:
      - "application/json"
      parameters:
      - name: "min"
        in: "path"
        description: "Минимум"
        required: true
        type: "integer"
        format: "int32"
      - name: "max"
        in: "path"
        description: "Максимум"
        required: true
        type: "integer"
        format: "int32"
      responses:
        200:
          description: "OK"
```

В спецификации представлена следующая информация о запросе:

- _x-ISC\_CORS_: включить CORS для данной операции (см. дальше)
- _summary_ and _description_: краткое и подробное описания
- _operationId_ - внутренний идентификатор операции, также это название метода с кодом в нашем классе имплементации
- _produces_ - формат ответа (text, xml, json)
- _parameters_ входные параметры (в URL  или теле запроса), в нашем случае мы описываем 2 параметра - минимальную и максимальную границы для генератора случайных чисел
- _responses_ - список возможных ответов сервера

Как вы видите, этот формат не является особенно сложным, хотя есть еще много возможностей, вот [спецификация](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md).

Следующий шаг - экспорт спецификации в формат JSON. Перейдите в меню `File` → `Convert and save as JSON`. Спецификация должна выглядеть примерно так:


```json
{
  "swagger": "2.0",
  "info": {
    "description": "Math",
    "version": "1.0.0",
    "title": "Math REST API"
  },
  "host": "localhost:52773",
  "basePath": "/math",
  "schemes": [
    "http"
  ],
  "paths": {
    "/random/{min}/{max}": {
      "get": {
        "x-ISC_CORS": true,
        "summary": "Генерация случайного целого числа",
        "description": "Генерация случайного целого числа между min и max",
        "operationId": "getRandom",
        "produces": [
          "application/json"
        ],
        "parameters": [
          {
            "name": "min",
            "in": "path",
            "description": "Минимум",
            "required": true,
            "type": "integer",
            "format": "int32"
          },
          {
            "name": "max",
            "in": "path",
            "description": "Максимум",
            "required": true,
            "type": "integer",
            "format": "int32"
          }
        ],
        "responses": {
          "200": {
            "description": "OK"
          }
        }
      }
    }
  }
}
```

*Загрузка спецификации в InterSystems IRIS*

Теперь, когда у нас есть спецификация, мы можем сгенерировать код этого REST API в InterSystems IRIS.

Для перехода к этому этапу, нам понадобятся:

- Имя REST-приложения: пакет для нашего сгенерированного кода (`math`)
- Спецификация Open API в формате JSON: мы только что создали ее на предыдущем шаге.
- Имя WEB-приложения: базовый путь для доступа к нашему API REST (`/math`)


Есть три способа использовать нашу спецификацию для генерации кода, они по сути одинаковы, но предлагают различные способы доступа к одной и той же функциональности:

1. Рутина `^%REST` (`Do ^%REST` в интерактивной терминальной сессии), [документация](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=GREST_routine).
2. Класс `%REST.API` (`Set sc = ##class(%REST.API).CreateApplication(applicationName, spec)`, в неинтерактивной терминальной сессии), [документация](https://docs.intersystems.com/irislatest/csp/docbook/Doc.View.cls?KEY=GREST_objectscriptapi).
3. API Management REST API, [документация](https://docs.intersystems.com/irislatest/csp/docbook/Doc.View.cls?KEY=GREST_apimgmnt).

Документация подробно описывает необходимые шаги, так что выберите любой из них. Добавлю только два примечания:

- В случаях (1) и (2) в качестве спецификации можно передать динамический объект, полное имя файла или URL
- В случаях (2) и (3) **необходимо** создать WEB-приложение: `set sc = ##class(%SYS.REST).DeployApplication(restApp, webApp, authenticationType)`, в нашем случае `set sc = ##class(%SYS.REST).DeployApplication("math", "/math")`, значения аргумента `authenticationType` доступны в `%sySecurity.inc`, см. макросы `$$$Authe*`, для анонимного доступа передайте значение `$$$AutheUnauthenticated` (32). Если значения нет то используется авторизация по паролю.

*Что произошло со спецификацией?*

После того как вы создали приложение, в новом пакете `math` должны быть сгенерированы 3 класса (для пользователей VSCode - по умолчанию VSCode не показывает сгенерированные классы, поэтому класс брокера не будет виден. [Включите](https://community.intersystems.com/post/developing-rest-api-spec-first-approach#comment-145666) отображение сгенерированных классов):

- _Spec_ - хранит спецификацию в формате JSON
- _Disp_ - вызывается при запросах к REST API. Брокер вызывает класс имплементации REST API.
- _Impl_ - содержит бизнес-логику REST API. Разработчик изменяет только этот класс.

[Документация](https://docs.intersystems.com/irislatest/csp/docbook/Doc.View.cls?KEY=GREST_intro#GREST_intro_classes) этих классов.

*Бизнес-логика*

Изначально класс имплементации `math.impl` содержит только один метод, соответствующий операции `GET /random/{min}/{max}`:

```
/// Get random integer between min and max<br/>
/// The method arguments hold values for:<br/>
///     min, Minimal Integer<br/>
///     max, Maximal Integer<br/>
ClassMethod getRandom(min As %Integer, max As %Integer) As %DynamicObject
{
    //(Place business logic here)
    //Do ..%SetStatusCode(<HTTP_status_code>)
    //Do ..%SetHeader(<name>,<value>)
    //Quit (Place response here) ; response may be a string, stream or dynamic object
}
```
Добавим бизнес-логику:
```
ClassMethod getRandom(min As %Integer, max As %Integer) As %DynamicObject
{
    quit {"value":($random(max-min)+min)}
}
```
И вызовем наше REST API перейдя по ссылке в браузере: http://localhost:52773/math/random/1/100

В результате с сервера должен прийти такой ответ (значение `value` может отличаться):
```json
{
    "value": 45
}
```
Также REST API можно вызвать прямо из Swagger нажав на кнопку `Try it out` и заполнив форму параметров:

![](https://community.intersystems.com/sites/default/files/inline/images/images/image-20191120141939-1.png)

Поздравляю! Наше первое REST API созданное от спецификации заработало!

*Дальнейшая разработка*

Конечно, новое REST API не статично, необходимо добавлять новые пути, операции и так далее. Сначала разрабатывается спецификация, затем обновляется REST-приложение (команды идентичны командам создания приложения) и, наконец, пишется код. Обратите внимание, что обновление спецификации безопасно: ваш код не будет удален, даже если путь был удален из спецификации, в классе имплементации метод не будет удален.

*Особенности*


**Параметры**

InterSystems IRIS также работает с некоторыми дополнительными параметрами спецификацию Open API:

| Название                   | Тип данных  | Значение по-умолчанию| Раздел | Описание                                                                                                                                                                       |
|------------------------|-----------|-----------|-----------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| x-ISC_DispatchParent   | classname | `%CSP.REST` | info      | Суперкласс для класса брокера. |
| x-ISC_CORS             | boolean   | `false`     | operation | Включение поддержки CORS запросов.  |
| x-ISC_RequiredResource | array     |           | operation | Разделенный запятыми список ресурсов и привилегий (ресурс:привелегия) необходимый для доступа к операции. Пример: ["%Development:USE"]|
| x-ISC_ServiceMethod    | string    |           | operation | Имя метода класса, вызываемого для исполнения операции, по умолчанию равен operationId.                                         |

**CORS**

Есть 3 способа добавления CORS:

1. На уровне операции, установив параметр `x-ISC_CORS` равным `true`. В Math REST API мы пошли именно этим путём.

2. На уровне API, установив параметр в классе имплементации:
```
Parameter HandleCorsRequest = 1;
```

3. (Рекомендованный)  На уровне API, через собственный класс брокера (наследующийся от `%CSP.REST`), в котором реализована логика обработки CORS запросов. Для этого используйте параметр `x-ISC_DispatchParent` в вашей спецификации.

**Загрузка спецификации в InterSystems API Manager**

Наконец, давайте добавим нашу спецификацию в InterSystems API Manager (IAM), чтобы она была доступна для других разработчиков, для контроля потребления API и аналитики по работе API.

Если вы еще не начали работать с IAM, прочитайте [эту статью](https://habr.com/ru/company/intersystems/blog/475670/) и посмотрите [вебинар](https://www.youtube.com/watch?v=ZmEGCjIVrKs). В статье описывается запуск IAM и добавление REST API в IAM, поэтому я пропущу эти шаги тут. Перед загрузкой спецификации API в IAM необходимо изменить параметры сервера и базового пути так, чтобы они указывали на IAM, а не на InterSystems IRIS.

Откройте портал администратора IAM и перейдите на вкладку Specs (спецификации) в соответствующем workspace.

![](https://community.intersystems.com/sites/default/files/inline/images/images/image-20191120150407-1.png)

Нажмите на кнопку `Add Spec` и введите название нового API (в нашем случае `math`). После создания новой Spec в IAM нажмите кнопку `Edit` и вставьте код спецификации (JSON или YAML - для IAM это не имеет значения):

![](https://community.intersystems.com/sites/default/files/inline/images/images/image-20191120151502-2.png)

Не забудьте нажать `Update File`.

Теперь наше API опубликовано для разработчиков. Откройте портал разработчика и нажмите кнопку `Documentation` в правом верхнем углу. В дополнение к трем стандартным API появился наш новый API Math:

![](https://community.intersystems.com/sites/default/files/inline/images/images/image-20191120151654-3.png)

Откройте его:

![](https://community.intersystems.com/sites/default/files/inline/images/images/image-20191120151813-4.png)

Теперь разработчики могут посмотреть документацию по нашему новому API и тут же протестировать его!

**Дополнительные инструменты**

В InterSystems IRIS доступны дополнительные инструменты работы с Open API спецификацией:

1. [Open API Client Gen](https://openexchange.intersystems.com/package/objectscript-openapi-definition) для генерации InterSystems ObjectScript клиента для работы со внешними REST API.
2. [Openapi-definition](https://openexchange.intersystems.com/package/objectscript-openapi-definition) для генерации модели данных из Open API спецификации.
2. [Генератор Swagger описания](https://community.intersystems.com/post/generate-swagger-spec-persistent-and-serial-classes) по модели данных.
3. [Парсер YAML](https://openexchange.intersystems.com/package/yaml-utils).

**Выводы**

InterSystems IRIS упрощает процесс разработки REST API, а подход, основанный на разработке от спецификации, позволяет быстрее и проще управлять жизненным циклом REST API. С помощью этого подхода можно использовать различные инструменты для решения различных смежных задач, таких как генерация клиентов, юнит тестирование, управление API и многое другое.

**Ссылки**

- [OpenAPI 2.0 Specification](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md)
- [Создание REST API](https://docs.intersystems.com/irislatest/csp/docbook/Doc.View.cls?KEY=GREST)
- [Статья по IAM](https://habr.com/ru/company/intersystems/blog/475670/)
- [Вебинар по IAM](https://www.youtube.com/watch?v=ZmEGCjIVrKs)
- [Документация IAM](https://docs.intersystems.com/irislatest/csp/docbook/apimgr/index.html)
