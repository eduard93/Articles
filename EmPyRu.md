# Embedded Python 

Embedded Python позволяет использовать язык программирования Python вместе с InterSystems ObjectScript, языком программирования платформы данных InterSystems IRIS. Когда вы создаёте метод в классе InterSystems IRIS, используя Python, исходный код Python компилируется в объектный Python код, который запускается на сервере вместе с компилированным кодом ObjectScript. Это позволяет добиться более тесной интеграции, чем при использовании шлюза [Python Gateway](https://docs.intersystems.com/iris20212/csp/docbook/DocBook.UI.Page.cls?KEY=BEXTSERV) или [Native API для Python](https://docs.intersystems.com/iris20212/csp/docbook/DocBook.UI.Page.cls?KEY=BPYNAT_intro). Вы также можете импортировать пакеты Python, как собственные, так и общедоступные, и использовать их в коде ObjectScript. Объекты Python могут быть свободно использованы в ObjectScript и наоборот.

В этой статье мы рассматрим следующие сценарии:

- Использование Python-библиотек из ObjectScript - Этот сценарий предполагает, что вы являетесь разработчиком ObjectScript и хотите использовать возможности многочисленных библиотек Python, которые доступны сообществу разработчиков Python.
- Вызов API InterSystems IRIS из Python - Этот сценарий предполагает, что вы являетесь разработчиком Python, который только начинает работать с InterSystems IRIS, и хотите узнать, как получить доступ к InterSystems IRIS.
- Совместное использование ObjectScript и Python - этот сценарий предполагает, что вы работаете в смешанной команде разработчиков ObjectScript и Python и хотите узнать, как использовать эти два языка вместе.

Для воспроизведения примеров из статьи вам понадобится InterSystems IRIS версии 2021.2 или более поздней. 

Некоторые примеры в этой статье используют классы из репозитория [Samples-Data](https://github.com/intersystems/Samples-Data). Рекомендуем создать область `SAMPLES` и загружать примеры туда. Если вы хотите просмотреть или изменить код примера, вам необходимо установить интегрированную среду разработки (IDE). Рекомендуется использовать Visual Studio Code.

# Использование библиотек Python из ObjectScript

Python дает разработчикам ObjectScript простой способ вызвать любую из многочисленных библиотек Python (обычно называемых "пакетами" или "модулями") прямо из InterSystems IRIS, устраняя необходимость разработки собственных библиотек для дублирования существующих функций. InterSystems IRIS ищет библиотеки Python в каталоге `<installdir>/mgr/python`.

Подготовка пакета Python для использования из ObjectScript состоит из двух этапов:
- Из командной строки установите нужный пакет из индекса пакетов Python [pypi](https://pypi.org), другого индекса или репозитория.
- В ObjectScript импортируйте установленный пакет, чтобы загрузить пакет и получить его в виде объекта. Затем вы можете использовать объект так же, как и инстанцированный объект ObjectScript.

## Установка пакета Python

Пакеты Python можно устанавливать из командной строки с помощью программы установки пакетов pip3: `<iris>\bin\pip3 install --target <iris>/mgr/python <package>`. Например, вы можете установить пакет `numpy` на машину Windows следующим образом:

```
C:\InterSystems\IRIS\bin\pip3 install --target C:\InterSystems\IRIS\mgr\python numpy
```

Важно! Обязательно использовать `pip3` от InterSystems. Использовать системный pip можно только если у вас установлен Python = 3.9.5 (на момент выхода статьи и InterSystems IRIS 2021.2).

## Импорт пакета Python

Класс [%SYS.Python](https://docs.intersystems.com/iris20212/csp/documatic/%25CSP.Documatic.cls?&LIBRARY=%25SYS&CLASSNAME=%25SYS.Python) содержит методы, необходимую для использования Python из ObjectScript. Вы можете использовать `%SYS.Python` в любом контексте ObjectScript, например, в классах, интерактивных терминальных сессиях или SQL процедурах. 

Чтобы импортировать пакет или модуль Python из ObjectScript, используйте метод `%SYS.Python:Import()`.

Например, следующая команда импортирует модуль [math](https://docs.python.org/3/library/math.html):

```
set pymath = ##class(%SYS.Python).Import("math")
```

Модуль `math` поставляется в комплекте с Python, поэтому вам не нужно устанавливать его перед импортом. Вызвав команду [zwrite](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=RCOS_czwrite) для объекта `pymath`, вы можете увидеть, что он является экземпляром встроенного модуля math:

```
>zwrite pymath

pymath=1@%SYS.Python ; <module 'math' (built-in)> ; <OREF>
```

Теперь вы можете получить доступ к свойствам и методам модуля `math` точно так же, как и к любому объекту ObjectScript:

```
>write pymath.pi
3.141592653589793116
>write pymath.factorial(10)
3628800
```

## Пример

В этом примере используется пакет [geopy](https://geopy.readthedocs.io/en/stable/) для геокодирования [Nominatim](https://geopy.readthedocs.io/en/stable/#nominatim) от OpenStreetMap. Геокодирование - это процесс преобразования текстового описания местоположения, например, адреса или названия места, и возвращения географических координат, таких как широта и долгота.

Сначала установите `geopy` из командной строки:

```
>pip3 install --target c:\Intersystems\IRIS\mgr\python geopy

Collecting geopy
  Using cached geopy-2.2.0-py3-none-any.whl (118 kB)
Collecting geographiclib<2,>=1.49
  Using cached geographiclib-1.52-py3-none-any.whl (38 kB)
Installing collected packages: geographiclib, geopy
Successfully installed geographiclib-1.52 geopy-2.2.0
```

Затем выполните следующие команды в терминале, чтобы импортировать и использовать модуль `geopy`:

```
set geopy = ##class(%SYS.Python).Import("geopy")
set args = { "user_agent": "Embedded Python" }
set geolocator = geopy.Nominatim(args...)
set office = geolocator.geocode("Москва, Краснопресненская наб. 12")
```

Посмотрим на результаты:

```
>write office.address
UPS, 12, Краснопресненская набережная, Кудрино, Пресненский район, Москва, Центральный федеральный округ, 123610, Россия
>write office.latitude _ ", " _ office.longitude
55.7551364, 37.5567383
>set office2 = geolocator.reverse("55.7551364,37.5567383")
>write office.address
UPS, 12, Краснопресненская набережная, Кудрино, Пресненский район, Москва, Центральный федеральный округ, 123610, Россия
```

Этот пример импортирует модуль `geopy` в процесс InterSystems IRIS, делая его доступным для вызова из ObjectScript. Затем мы используем сервис Nominatim для создания объекта геолокатора. В примере используется метод `geocode` геолокатора, чтобы найти местоположение заданное строкой. Затем вызывается метод `reverse`, чтобы наоборот найти адрес, заданный широтой и долготой.

Важно отметить, что `Nominatum` принимает только именованные (ключевые, keyword) аргументы, что напрямую не поддерживается в ObjectScript. Решением является создание динамического объекта, содержащего необходимые аргументы (который в данном случае устанавливает ключевое слово `user_agent` равным `Embedded Python`), а затем передача его в Python с помощью синтаксиса `args...` (что расширяет [существующий синтаксис args...](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=GCOS_usercode#GCOS_usercode_args_variable)).

В отличие от математического модуля, импортированного в предыдущем примере, вызов `zwrite` на объекте `geopy` является экземпляром пакета `geopy`, установленного в `C:\Intersystems\iris\mgr\python`:
```
>zwrite geopy
geopy=2@%SYS.Python ; <module 'geopy' from 'c:\\\intersystems\\iris\\mgr\\python\\\geopy\\\__init__.py'> ; <OREF>
```

## Доступ к InterSystems IRIS из Python

Если вы используете Embedded Python и вам необходимо взаимодействовать с InterSystems IRIS, вы можете использовать модуль `iris` из оболочки Python или из метода, написанного на Python в классе InterSystems IRIS. Чтобы выполнить примеры, приведенные в этом разделе, можно запустить оболочку Python из терминала с помощью команды ObjectScript: `do ##class(%SYS.Python).Shell()`.

Когда вы запускаете Python из терминала, Python наследует тот же контекст, что и терминал, например, текущую область и пользователя. Локальные переменные не наследуются.

### Классы

Чтобы получить доступ к классу InterSystems IRIS, импортируйте модуль `iris`, а затем инстанцируйте класс как объект, который вы хотите использовать. Затем вы можете использовать его свойства и методы так же, как и свойства и методы классов Python.

Вызовем метод `ManagerDirectory` системного класса [%Library.File](https://docs.intersystems.com/iris20212/csp/documatic/%25CSP.Documatic.cls?LIBRARY=%25SYS&CLASSNAME=%25Library.File#ManagerDirectory) для вывода пути к mgr каталогу InterSystems IRIS:

```
>>> import iris
>>> lf = iris.cls('%Library.File')
>>> print(lf.ManagerDirectory())
C:\InterSystems\IRIS\mgr\
```

В следующем примере используется класс [Sample.Company](https://github.com/intersystems/Samples-Data/blob/master/cls/Sample/Company.cls) из репозитория [Samples-Data](https://github.com/intersystems/Samples-Data). Хотя классы, начинающиеся со знака процента (%), такие как `%SYS.Python` или `%Library.File`, можно использовать из любого пространства имен, для доступа к классу `Sample.Company` необходимо находиться в пространстве имен `SAMPLES`, как упоминалось ранее.

Определение класса Sample.Company выглядит следующим образом:

```
Class Sample.Company Extends (%Persistent, %Populate, %XML.Adaptor)
{

/// The company's name.
Property Name As %String(MAXLEN = 80, POPSPEC = "Company()") [ Required ];

/// The company's mission statement.
Property Mission As %String(MAXLEN = 200, POPSPEC = "Mission()");

/// The unique Tax ID number for the company.
Property TaxID As %String [ Required ];

/// The last reported revenue for the company.
Property Revenue As %Integer;

/// The Employee objects associated with this Company.
Relationship Employees As Employee [ Cardinality = many, Inverse = Company ];

}
```

Этот класс наследуется от [%Library.Persistent](https://docs.intersystems.com/iris20212/csp/documatic/%25CSP.Documatic.cls?LIBRARY=%25SYS&CLASSNAME=%25Library.Persistent) (часто сокращенно `%Persistent`), что означает, что объекты этого класса могут быть сохранены в базе данных InterSystems IRIS. Класс также имеет несколько свойств, включая `Name` и `TaxID`, оба из которых необходимы для сохранения объекта.

Хотя их не видно в определении класса, хранимые классы поставляются с рядом методов для работы с объектами этого класса, таких как `%New`, `%Save`, `%Id` и `%OpenId`. Однако знаки процента (`%`) не допускаются в именах методов Python, поэтому вместо них используется знак подчеркивания (`_`).

Приведенный ниже код создает новый объект Company, устанавливает обязательные свойства Name и TaxID, а затем сохраняет компанию в базе данных:

```
>>> import iris
>>> myCompany = iris.cls('Sample.Company')._New()
>>> myCompany.Name = 'Acme Widgets, Inc.'
>>> myCompany.TaxID = '123456789'
>>> status = myCompany._Save()
>>> print(status)
1
>>> print(myCompany._Id())
22
```

Приведенный выше код использует метод `_New` для создания экземпляра класса и `_Save` для сохранения экземпляра в базе данных. Метод `_Save` возвращает статус. В данном случае `1` означает, что сохранение прошло успешно. Когда вы сохраняете объект, InterSystems IRIS присваивает ему уникальный идентификатор, который вы можете использовать для извлечения объекта из базы данных. Метод `_Id` возвращает идентификатор объекта.

Для извлечения объекта из постоянного хранилища в память для обработки используйте метод `_OpenId` класса:

```
>>> yourCompany = iris.cls("Sample.Company")._OpenId(22)
>>> print(yourCompany.Name)
Acme Widgets, Inc.
```

Добавление следующего кода в определение класса создает метод `Print`, который печатает название и TaxID текущей компании. Установка ключевого слова `Language` равным `python` сообщает компилятору класса, что метод написан на языке Python.

```
Method Print() [ Language = python ]
{
    print('\nName: ' + self.Name + ' TaxID: ' + self.TaxID)
}
```

Вызовем метод `Print`:

```
>>> yourCompany.Print()
 
Name: Acme Widgets, Inc. TaxID: 123456789
```

### SQL

Классы в InterSystems IRIS доступны как SQL таблицы (подробнее о взаимосвязи классов, таблиц и глобалов можно почитать [здесь](https://community.intersystems.com/post/classes-tables-and-globals-how-it-all-works)), позволяя вам получать доступ к данным с помощью SQL запросов, в дополнение к использованию методов классов или прямого доступа к глобалам. Модуль `iris` предоставляет возможность выполнять SQL запросы к IRIS из Python.

В следующем примере используется `iris.sql.prepare` для запуска SQL запроса, возвращающего все компании, название которых начинается на `A`

```
>>> stmt = iris.sql.prepare("SELECT Name, TaxID FROM Sample.Company WHERE Name %STARTSWITH ?")
>>> rs = stmt.execute("A")
```

```
>>> for idx, row in enumerate(rs):                                              
...     print(f"[{idx}]: {row}")                                                
...
[0]: [Acme Widgets, Inc.', 123456789]
```

### Глобалы

В InterSystems IRIS все данные хранятся в глобалах. Глобалы - это массивы, которые являются хранимыми (то есть хранятся на диске), многомерными (то есть могут иметь любое количество подзаписей) и разреженными (то есть подзаписи не должны быть смежными). Когда вы храните объекты класса или строки в таблице, эти данные фактически хранятся в глобалах, хотя вы обычно обращаетесь к ним через методы или SQL и не касаетесь глобальных таблиц напрямую.

Иногда бывает полезно хранить постоянные данные в глобальных таблицах, не создавая класс или таблицу SQL. В InterSystems IRIS глобальная переменная выглядит так же, как и любая другая переменная, но перед ее именем ставится каретка (^). В следующем примере имена рабочих дней хранятся в глобальной переменной ^Workdays.

```
>>> myGref = iris.gref('^Workdays')
>>> myGref[None] = 5
>>> myGref[1] = 'Monday'
>>> myGref[2] = 'Tuesday'
>>> myGref[3] = 'Wednesday'
>>> myGref[4] = 'Thursday'
>>> myGref[5] = 'Friday'
>>> print(myGref[3])
Wednesday
```

Первая строка кода, `myGref = iris.gref('^Workdays')`, получает глобальную ссылку (или gref) на глобал с именем `^Workdays`, который может уже существовать или нет.

Вторая строка, `myGref[None] = 5`, сохраняет количество рабочих дней в `^Workdays` без сабскрипта.

Третья строка, `myGref[1] = 'Monday'`, сохраняет строку Monday в узле глобала ^Workdays(1). Следующие четыре строки хранят остальные рабочие дни в `^Workdays(2)` - `^Workdays(5)`.

Последняя строка, `print(myGref[3])`, показывает, как можно получить доступ к значению, хранящемуся в глобале, по его gref.

## Совместное использование ObjectScript и Python

InterSystems IRIS облегчает совместную работу смешанных команд программистов на ObjectScript и Python. Например, некоторые методы в классе могут быть написаны на ObjectScript, а некоторые - на Python. Программисты могут выбрать язык, на котором им удобнее всего писать, или язык, который больше подходит для решения поставленной задачи.

### Создание смешанных классов InterSystems IRIS

Следующий класс имеет метод `Print, написанный на Python, и метод `Write`, написанный на ObjectScript, но функционально они эквивалентны, и любой из методов может быть вызван из Python или ObjectScript.

```
Class Sample.Company Extends (%Persistent, %Populate, %XML.Adaptor)
{

/// The company's name.
Property Name As %String(MAXLEN = 80, POPSPEC = "Company()") [ Required ];

/// The company's mission statement.
Property Mission As %String(MAXLEN = 200, POPSPEC = "Mission()");

/// The unique Tax ID number for the company.
Property TaxID As %String [ Required ];

/// The last reported revenue for the company.
Property Revenue As %Integer;

/// The Employee objects associated with this Company.
Relationship Employees As Employee [ Cardinality = many, Inverse = Company ];

Method Print() [ Language = python ]
{
    print ('\nName: ' + self.Name + ' TaxID: ' + self.TaxID)
}

Method Write() [ Language = objectscript ]
{
    write !, "Name: ", ..Name, " TaxID: ", ..TaxID
}
}
```

Этот пример кода Python показывает, как открыть объект Company с Id=2 и вызвать методы `Print` и `Write`.

```
>>> import iris
>>> company = iris.cls("Sample.Company")._OpenId(2)
>>> company.Print()
 
Name: IntraData Group Ltd. TaxID: G468
>>> company.Write()
 
Name: IntraData Group Ltd. TaxID: G468
```

Этот пример кода ObjectScript показывает, как открыть тот же объект Company и вызвать оба метода.

```
>set company = ##class(Sample.Company).%OpenId(2)
 
>do company.Print()
 
Name: IntraData Group Ltd. TaxID: G468
 
>do company.Write()
 
Name: IntraData Group Ltd. TaxID: G468
```

### Передача данных между Python и ObjectScript

Хотя Python и ObjectScript во многом совместимы, у них ряд уникальных типов данных и конструкций, и иногда при передаче данных из одного языка в другой необходимо выполнить некоторые преобразования данных. Один пример вы видели ранее, в примере передачи именованных аргументов из ObjectScript в Python.

Метод `Builtins` класса `%SYS.Python` предоставляет удобный способ доступа к встроенным функциям Python, которые могут помочь вам создать объекты того типа, который ожидается в методе Python.

Следующий пример ObjectScript создает два массива Python, `newport` и `cleveland`, каждый из которых содержит широту и долготу города:
```
>set builtins = ##class(%SYS.Python).Builtins()
 
>set moscow = builtins.list()
 
>do moscow.append(55.7558)
 
>do moscow.append(37.6173)
 
>set spb = builtins.list()
 
>do spb.append(59.9311)
 
>do spb.append(30.3609)

>zwrite moscow
newport=11@%SYS.Python  ; [(55.7558, 37.6173]  ; <OREF>

>zwrite spb
cleveland=11@%SYS.Python  ; [59.9311, 30.3609]  ; <OREF>
```

Приведенный ниже код использует пакет `geopy`, который вы видели в предыдущем примере, для расчета расстояния между Москвой и Санкт-Петербургом. Он вычисляет расстояние с помощью метода `geopy.distance.distance`, передавая массивы в качестве параметров, а затем печатает свойство km результирующей дистанции:

```
>set distance = $system.Python.Import("geopy.distance")
>set route = distance.distance(moscow, spb)
>write route.km

633.3167668109714
```

# Запуск произвольной команды Python

Когда вы разрабатываете или тестируете что-то, иногда бывает полезно запустить строку кода Python, чтобы посмотреть, что она делает или работает ли. В таких случаях можно использовать метод %SYS.Python.Run(), как показано в примере ниже:

```
>set rslt = ##class(%SYS.Python).Run("print('hello world')")
hello world
```

Важно! Команда Run запускает отдельный интерпретатор Python, который уничтожается по её завершении. Поэтому вот так работать не будет:

```
set rslt = ##class(%SYS.Python).Run("x=1")
set rslt = ##class(%SYS.Python).Run("print(x)")
```
