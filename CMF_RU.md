Деплоим шард-кластер InterSystems IRIS с Configuration Merge File

# План

- Configuration Merge File
- Actions
- Пример: Шард—кластер
- Выводы

# Configuration Merge File

Конфигурация платформы данных InterSystems IRIS во многом определяется файлом `iris.cpf`. При каждом запуске InterSystems IRIS считывает этот конфигурационный файл (далее CPF) для получения значений параметров работы инстанса. Это позволяет в любой момент изменить конфигурацию инстанса, изменив его CPF и перезапустив InterSystems IRIS. 

В этой статье мы запустим кластер InterSystems IRIS с помощью docker и [файлов Merge CPF](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=ACMF) (CMF) — новой функции, позволяющей легко конфигурировать серверы InterSystems IRIS. В UNIX® и Linux вы можете изменить стандартный iris.cpf с помощью декларативного файла Merge CPF. Файл Merge CPF — это частичный CPF, который устанавливает нужные значения для любых параметров при запуске InterSystems IRIS. 

Merge CPF использует те же параметры, содержащиеся в файле CPF. Новые элементы или значения этих элементов мёржатся в файл CPF при старте инстанса. Это позволяет обновлять любую часть CPF файла во время инициализации инстанса. Если переменная окружения `ISC_CPF_MERGE_FILE` установлена и содержит путь до файла Merge CPF, то содержимое этого файла мёржится в активный CPF файл.

Вот минимальный пример — этот файл Merge CPF устанавливает буфер глобалов равным 800 Мб:

```
[config]
globals=0,0,800,0,0,0
```

# Actions

Описанная выше возможность конфигурации инстанса ограничена более простыми параметрами конфигурации, как и показано в примере. Есть несколько типов системных настроек (связанных с базами данных и безопасностью), которые требуют большего обобщения. Эти параметры включают настройки для новых баз данных, областей, отображений глобалов и классов, а также всей информации, связанной с безопасностью. Для таких случаев в файле CMF есть специальная секция Actions.

Секция Actions позволяет выполнять набор действий при старте InterSystems IRIS. Набор действий основан на существующих API с классами `Security` и `Config`. Действия, которые могут быть выполнены, могут быть "Создать", "Изменить" или "Удалить". Например, для управления пользователями Actions позволят выполнять следующие действия: CreateUser, ModifyUser и DeleteUser. Вот пример:


```
[Actions]
DeleteUser:Name=Admin
CreateUser:Name=User1,Password=SYS,ChangePassword=1,Roles=%All,%Developer,%Manager
CreateUser:Name=User2,Password=SYS,ChangePassword=1,Roles=%Developer
ModifyUser:Name=_SYSTEM,ChangePassword=1
```

Параметры, передаваемые вызовам, являются именами свойств для соответствующего класса, в данном случае `Security.Users`. В приведенном выше сценарии `Security.Users:Delete()` вызывается для удаления пользователя Admin, `Security.Users:Create()` вызывается для создания пользователей User1 и User2, а `Security.Users:Modify()` вызывается для изменения пользователя `_SYSTEM`.

При выполнении слияния будут выполнены все действия, указанные в разделе Action. Обратите внимание, что действия выполняются в заранее определенном порядке, а не в том, в котором они указаны в файле CMF. Порядок в примере ниже будет таким: ModifySystem, CreateResource, CreateRole, CreateUser.

```
[Actions]
CreateUser:Name=User1,Password=SYS,ChangePassword=1,Roles=%All,%Developer,%Manager
CreateUser:Name=User2,Password=SYS,ChangePassword=1,Roles=%Developer,NewRole1
CreateUser:Name=User3,Password=SYS,ChangePassword=1,Roles=%Manager,NewRole2
CreateUser:Name=User4,Password=SYS,ChangePassword=1
CreateRole:Name=NewRole1,Description=NewRole1 Description,Resources=Resource1:READ,W
CreateRole:Name=NewRole2,Description=NewRole2 Description,Resources=Resource2:RW
CreateResource:Name=Resource1,Description=Resource1 Description,PublicPermission=R
CreateResource:Name=Resource2,Description=Resource2 Description
ModifySystem:AuditEnabled=1,SecurityDomains=1
ActivateCPF
```

Поведение операций Create / Modify / Delete следующее:
- Create - Если создаваемый элемент уже существует, то операция игнорируется, и возвращается 1.
- Modify - Если изменяемый элемент не существует, то операция игнорируется, и возвращается 1.
- Удалить - Если удаляемый элемент не существует, то операция игнорируется, и возвращается 1.

Вот все возможные Actions:

| Название                                                   | Класс                 |
| ---------------------------------------------------------- | --------------------- |
| ModifySystem                                               | Security.System       |
| ModifyService                                              | Security.Services     |
| CreateResource, ModifyResource, DeleteResource             | Security.Resources    |
| CreateRole, ModifyRole, DeleteRole                         | Security.Roles        |
| CreateUser, ModifyUser, DeleteUser                         | Security.Users        |
| CreateApplication, ModifyApplication, DeleteApplication    | Security.Applications |
| CreateLDAPConfig, ModifyLDAPConfig, DeleteLDAPConfig       | Security.LDAPConfigs  |
| CreateEvent, ModifyEvent, DeleteEvent                      | Security.Events       |
| CreateSSLConfig, ModifySSLConfig, DeleteSSLConfig          | Security.SSLConfigs   |
| CreateKMIPServer, ModifyKMIPServer, DeleteKMIPServer       | Security.KMIPServers  |
| CreateDocDB, ModifyDocDB, DeleteDocDB                      | Security.DocDBs       |
| CreateDatabaseFile, ModifyDatabaseFile, DeleteDatabaseFile | SYS.Database          |
| CreateDatabase, ModifyDatabase, DeleteDatabase             | SYS.Database          |
| CreateNamespace, ModifyNamespace, DeleteNamespace          | Config.Namespaces     |
| CreateGlobalMap, ModifyGlobalMap, DeleteGlobalMap          | Config.MapGlobals     |
| CreateRoutineMap, ModifyRoutineMap, DeleteRoutineMap       | Config.MapRoutines    |
| CreatePackageMap, ModifyPackageMap, DeletePackageMap       | Config.MapPackages    |

# Пример: Шард-кластер

[Шардирование](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=GSCALE_sharding) является важной функцией горизонтального масштабирования платформы данных InterSystems IRIS. Кластер InterSystems IRIS разделяет хранение и кэширование данных между несколькими серверами, обеспечивая гибкое и недорогое масштабирование производительности запросов чтения и записи данных при максимальном уменьшении стоимости инфраструктуры за счет высокоэффективного использования ресурсов. Шардинг легко сочетается со значительными возможностями вертикального масштабирования InterSystems IRIS, что значительно расширяет спектр рабочих нагрузок, поддерживаемых InterSystems IRIS.

Наш кластер будет состоять из одного Node1 (главный узел) и двух Data Nodes (вот все [возможные роли](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=RACS_ShardRole)). К сожалению, `docker-compose` не может развертываться на нескольких серверах (хотя он может развертываться на удаленных узлах), поэтому такой шард-кластер можно использовать для локальной разработки моделей данных с поддержкой шардинга, тестов и т.д. Для развертывания кластера InterSystems IRIS на нескольких серверах следует использовать либо [ICM](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=GICM_using), либо [IKO](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=AIKO).

## Docker-compose.yml

Начнём с конфигурации docker-compose:

```
version: '3.7'
services:
  iris1:
    image: containers.intersystems.com/intersystems/iris:2020.3.0.221.0
    init: true
    command: --key /ISC/iris.key
    hostname: iris1
    environment:
     - ISC_DATA_DIRECTORY=/ISC/iris.sys.d/sys1
     - ISC_CPF_MERGE_FILE=/ISC/CPF2merge-master-instance.conf
    volumes:
     - ./:/ISC:delegated
    ports:
      - 9011:1972
      - 9012:52773

  iris2:
    image: containers.intersystems.com/intersystems/iris:2020.3.0.221.0
    command: --key /ISC/iris.key --before 'sleep 60'
    init: true
    hostname: iris2
    environment:
     - ISC_DATA_DIRECTORY=/ISC/iris.sys.d/sys2
     - ISC_CPF_MERGE_FILE=/ISC/CPF2merge-data-instance.conf
    volumes:
     - ./:/ISC:delegated
    depends_on:
      - iris1
    ports:
      - 9021:1972
      - 9022:52773

  iris3:
    image: containers.intersystems.com/intersystems/iris:2020.3.0.221.0
    command: --key /ISC/iris.key --before 'sleep 60'
    init: true
    hostname: iris3
    environment:
     - ISC_DATA_DIRECTORY=/ISC/iris.sys.d/sys3
     - ISC_CPF_MERGE_FILE=/ISC/CPF2merge-data-instance.conf
    volumes:
     - ./:/ISC:delegated
    depends_on:
      - iris1
    ports:
      - 9031:1972
      - 9032:52773
```

Мы запускаем стандартный образ `intersystems/iris:2020.3.0.221.0` с лицензионным ключом с поддержкой шардинга, сохраняем данные с помощью [Durable %SYS](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=ADOCK#ADOCK_iris_durable) и предоставляем `ISC_CPF_MERGE_FILE`, указывающий на наши файлы Merge CPF (которые отличаются для Node1 и Data Nodes). Кроме того, Data Nodes запускаются на минуту позже, чем Node1, но это крайне консервативная оценка, на большинстве серверов время запуска занимает максимум несколько секунд.

Конфигурация кластера происходит в файлах Merge CPF.

## CPF2merge-data-instance.conf

```
[Startup]
PasswordHash=FBFE8593AEFA510C27FD184738D6E865A441DE98,u4ocm4qh
ShardRole=node1


[config]
MaxServerConn=64
MaxServers=64
globals=0,0,400,0,0,0
errlog=1000
routines=32
gmheap=256000
locksiz=1179648
```
Что здесь происходит?

В части [Startup] мы включаем шардинг, назначая роль `Node1` этому инстансу и устанавливаем пароль `SYS` для всех пользователей. В секции [config] мы немного расширяем наш сервер. Вот и всё!

## CPF2merge-data-instance.conf

```
[Startup]

ShardClusterURL=IRIS://iris1:1972/IRISCLUSTER
ShardRole=DATA
```

Для нод данных достаточно установить адрес Node1 и роль ноды в кластере.

## Репозиторий

Оба примера доступны в репозитории:
```
git clone https://github.com/intersystems-ru/iris-container-recipes.git
cd iris-container-recipes
cd cluster
// copy iris.key in cluster folder
docker-compose up -d
```

После запуска кластера Intersystems IRIS можно получить к нему доступ из браузера. Логин/пароль: `_SYSTEM`/`SYS`.
Также в репозитории, в папке `mirror` представлен пример запуска зеркалированной конфигурации из двух нод — основной и резервной и арбитра.

# Ссылки

- [Merge CPF](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=ACMF)
- [Репозиторий](https://github.com/intersystems-ru/iris-container-recipes)
- [Шардинг](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=GSCALE_sharding)

# Выводы

Merge CPF files — это отличный и простой инструмент, позволяющий легко конфигурировать InterSystems IRIS.
