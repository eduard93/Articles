# Python в InterSystems IRIS 

Embedded Python (далее Python) позволяет использовать язык программирования Python вместе с InterSystems ObjectScript, языком программирования платформы данных InterSystems IRIS. Когда вы создаёте метод в классе InterSystems IRIS, используя Python, исходный код Python компилируется в объектный Python код, который запускается на сервере вместе с компилированным кодом ObjectScript. Это позволяет добиться более тесной интеграции, чем при использовании шлюза [Python Gateway](https://docs.intersystems.com/iris20212/csp/docbook/DocBook.UI.Page.cls?KEY=BEXTSERV) или [Native API для Python](https://docs.intersystems.com/iris20212/csp/docbook/DocBook.UI.Page.cls?KEY=BPYNAT_intro). Вы также можете импортировать пакеты Python, как собственные, так и общедоступные, и использовать их в коде ObjectScript. Объекты Python могут быть свободно использованы в ObjectScript и наоборот.

В этой статье мы рассмотрим следующие сценарии:

- Использование Python-библиотек из ObjectScript - этот сценарий предполагает, что вы являетесь разработчиком ObjectScript и хотите использовать возможности многочисленных библиотек Python, которые доступны сообществу разработчиков Python.
- Вызов API InterSystems IRIS из Python - этот сценарий предполагает, что вы являетесь разработчиком Python, который только начинает работать с InterSystems IRIS, и хотите узнать, как получить доступ к InterSystems IRIS.
- Совместное использование ObjectScript и Python - этот сценарий предполагает, что вы работаете в смешанной команде разработчиков на ObjectScript и Python и хотите узнать, как использовать эти два языка вместе.

Для воспроизведения примеров из статьи вам понадобится InterSystems IRIS версии 2021.2+. 

Некоторые примеры в этой статье используют классы из репозитория [Samples-Data](https://github.com/intersystems/Samples-Data). Рекомендуем создать область `SAMPLES` и загружать примеры туда. Если вы хотите просмотреть или изменить код примера, вам необходимо установить интегрированную среду разработки (IDE). Рекомендуется использовать Visual Studio Code.

# Использование библиотек Python из ObjectScript

Python дает разработчикам ObjectScript простой способ вызвать любую из многочисленных библиотек Python (обычно называемых "пакетами" или "модулями") прямо из InterSystems IRIS, устраняя необходимость разработки собственных библиотек для дублирования существующих функций. InterSystems IRIS ищет библиотеки Python в каталоге `<installdir>/mgr/python`.

Подготовка пакета Python для использования из ObjectScript состоит из двух этапов:
- Из командной строки установите нужный пакет из индекса пакетов Python [pypi](https://pypi.org), другого индекса или репозитория.
- В ObjectScript импортируйте установленный пакет, чтобы загрузить пакет и получить его в виде объекта. Затем вы можете использовать объект так же, как и инстанцированный объект ObjectScript.

## Установка пакета Python

Пакеты Python можно устанавливать из командной строки с помощью программы установки пакетов irispip: `<iris>\bin\irispip install --target <iris>/mgr/python <package>`. Например, вы можете установить пакет `numpy` на машину Windows следующим образом:

```
C:\InterSystems\IRIS\bin\irispip install --target C:\InterSystems\IRIS\mgr\python numpy
```

Важно! Обязательно использовать `irispip` от InterSystems. Использовать системный `pip` или `<iris>\bin\pip3` можно только если у вас установлен Python 3.9.5 (на момент выхода статьи с InterSystems IRIS 2021.2 поставляется именно эта версия Python).

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

В этом примере используется пакет [geopy](https://geopy.readthedocs.io/en/stable/) для геокодирования [Nominatim](https://geopy.readthedocs.io/en/stable/#nominatim) от OpenStreetMap. Геокодирование - это процесс преобразования текстового описания местоположения, например, адреса или названия места, в географические координаты (широту и долготу).

Сначала установите `geopy` из командной строки:

```
>C:\InterSystems\IRIS\bin\irispip install --target c:\Intersystems\IRIS\mgr\python geopy

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

Этот пример импортирует модуль `geopy` в процесс InterSystems IRIS, делая его доступным для вызова из ObjectScript. Затем мы используем метод Nominatim для создания объекта геолокатора. В примере используется метод `geocode` геолокатора для поиска координат. Затем вызывается метод `reverse`, чтобы наоборот найти адрес, по заданным широте и долготе.

Важно! `Nominatum` принимает только именованные (ключевые, keyword) аргументы, что напрямую не поддерживается в ObjectScript. Решением является создание динамического объекта, содержащего необходимые аргументы (который в данном случае устанавливает ключевое слово `user_agent` равным `Embedded Python`), а затем передача его в Python с помощью синтаксиса `args...` (что расширяет [существующий синтаксис args...](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=GCOS_usercode#GCOS_usercode_args_variable)).

В отличие от математического модуля, импортированного в предыдущем примере, вызов `zwrite` на объекте `geopy` является экземпляром пакета `geopy`, установленного в `C:\Intersystems\iris\mgr\python`:
```
>zwrite geopy
geopy=2@%SYS.Python ; <module 'geopy' from 'c:\\\intersystems\\iris\\mgr\\python\\\geopy\\\__init__.py'> ; <OREF>
```

## Доступ к InterSystems IRIS из Python

Если вы используете Embedded Python и вам необходимо взаимодействовать с InterSystems IRIS, вы можете использовать модуль `iris` из оболочки Python или из метода, написанного на Python в классе InterSystems IRIS. Чтобы выполнить примеры, приведенные в этом разделе, можно запустить оболочку Python из терминала с помощью команды ObjectScript: `do ##class(%SYS.Python).Shell()`.

Когда вы запускаете Python из терминала, Python наследует тот же контекст, что и терминал, например, текущую область и пользователя. Локальные переменные не наследуются.

### Классы

Чтобы получить доступ к классу InterSystems IRIS, импортируйте модуль `iris`, а затем инстанцируйте класс как объект. После этого вы можете использовать его свойства и методы так же, как и свойства и методы классов Python.

Вызовем метод `ManagerDirectory` системного класса [%Library.File](https://docs.intersystems.com/iris20212/csp/documatic/%25CSP.Documatic.cls?LIBRARY=%25SYS&CLASSNAME=%25Library.File#ManagerDirectory) для вывода пути к `mgr` каталогу InterSystems IRIS:

```
>>> import iris
>>> lf = iris.cls('%Library.File')
>>> print(lf.ManagerDirectory())
C:\InterSystems\IRIS\mgr\
```

В следующем примере используется класс [Sample.Company](https://github.com/intersystems/Samples-Data/blob/master/cls/Sample/Company.cls) из репозитория [Samples-Data](https://github.com/intersystems/Samples-Data). Хотя классы, начинающиеся со знака процента (%), такие как `%SYS.Python` или `%Library.File`, можно использовать из любого пространства имен, для доступа к классу `Sample.Company` необходимо находиться в пространстве имен `SAMPLES`, как упоминалось ранее.

Класс Sample.Company:

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

Этот класс наследуется от [%Library.Persistent](https://docs.intersystems.com/iris20212/csp/documatic/%25CSP.Documatic.cls?LIBRARY=%25SYS&CLASSNAME=%25Library.Persistent) (часто просто `%Persistent`), что означает, что объекты этого класса могут быть сохранены в базе данных InterSystems IRIS. Класс также имеет несколько свойств, включая `Name` и `TaxID`, оба из которых необходимы для сохранения объекта.

Хотя их не видно в определении класса, хранимые классы поставляются с рядом методов для работы с объектами этого класса, таких как `%New`, `%Save`, `%Id` и `%OpenId` ([документация](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=GOBJ_persobj_intro)). Однако знаки процента (`%`) не допускаются в именах методов Python, поэтому вместо них используется знак подчеркивания (`_`).

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

Приведенный выше код использует метод `_New` для создания экземпляра класса и `_Save` для сохранения экземпляра в базе данных. Метод `_Save` возвращает статус (подробнее см. раздел Статусы). В данном случае `1` означает, что сохранение прошло успешно. Когда вы сохраняете объект, InterSystems IRIS присваивает ему уникальный идентификатор, который вы можете использовать для извлечения объекта из базы данных. Метод `_Id` возвращает идентификатор объекта.

Для извлечения объекта из постоянного хранилища в память для обработки используйте метод `_OpenId` класса:

```
>>> yourCompany = iris.cls("Sample.Company")._OpenId(22)
>>> print(yourCompany.Name)
Acme Widgets, Inc.
```

Добавим метод `Print`, который печатает `Name` и `TaxID` компании. Установка ключевого слова `Language` равным `python` сообщает компилятору класса, что метод написан на языке Python.

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

Классы в InterSystems IRIS доступны как SQL таблицы (подробнее о взаимосвязи классов, таблиц и глобалов можно почитать [здесь](https://community.intersystems.com/post/classes-tables-and-globals-how-it-all-works)), позволяя вам получать доступ к данным с помощью SQL запросов, в дополнение к использованию методов классов или прямого доступа к глобалам. Модуль `iris` предоставляет возможность выполнять SQL запросы к InterSystems IRIS из Python.

В следующем примере используется `iris.sql.prepare` для запуска SQL запроса, возвращающего все компании, название которых начинается на `A` (используйте метод `dataframe` для получения pandas dataframe):

```
>>> stmt = iris.sql.prepare("SELECT Name, TaxID FROM Sample.Company WHERE Name %STARTSWITH ?")
>>> rs = stmt.execute("A")
>>> df = rs.dataframe()
>>> for idx, row in enumerate(rs):                                              
...     print(f"[{idx}]: {row}")                                                
...
[0]: [Acme Widgets, Inc.', 123456789]
```

### SQL процедуры, триггеры

Любой метод класса InterSystems IRIS может быть доступен из SQL. Для этого добавьте [SQLProc](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=ROBJ_method_sqlproc) в ключевые слова метода.

Вы также можете писать хранимые процедуры и триггеры на Python, указав аргумент LANGUAGE PYTHON в операторе CREATE, как показано ниже:

```sql
CREATE FUNCTION tzconvert(dt DATETIME, tzfrom VARCHAR, tzto VARCHAR)
    RETURNS DATETIME
    LANGUAGE PYTHON
{
    from datetime import datetime
    from dateutil import parser, tz
    d = parser.parse(dt)
    if (tzfrom is not None):
        tzf = tz.gettz(tzfrom)
        d = d.replace(tzinfo = tzf)
    return d.astimezone(tz.gettz(tzto)).strftime("%Y-%m-%d %H:%M:%S")
}
```

Вызовем эту SQL процедуру, преобразующую дату/время из московского времени во всемирное координированное время (UTC).

```sql
SELECT tzconvert(now(), 'Europe/Moscow', 'UTC')
```

### Глобалы

В InterSystems IRIS все данные хранятся в глобалах. Глобалы - это массивы, которые являются хранимыми (то есть хранятся на диске), многомерными (то есть могут иметь любое количество подзаписей) и разреженными (то есть подзаписи не обязательно должны быть смежными). Когда вы храните объекты класса или строки в таблице, эти данные хранятся в глобалах, хотя вы обычно обращаетесь к ним через объектную модель или SQL и не работаете с глобалами напрямую.

Иногда бывает полезно хранить данные в глобалах, не создавая класс или таблицу SQL. В InterSystems IRIS глобал выглядит так же, как и любая другая переменная, но перед ее именем ставится каретка (`^`). В следующем примере имена рабочих дней хранятся в глобале `^Workdays`.

```
>>> myGref = iris.gref('^Workdays')
>>> myGref[None] = 5
>>> myGref[1] = 'Понедельник'
>>> myGref[2] = 'Вторник'
>>> myGref[3] = 'Среда'
>>> myGref[4] = 'Четверг'
>>> myGref[5] = 'Пятница'
>>> print(myGref[3])
Wednesday
```

Первая строка кода, `myGref = iris.gref('^Workdays')`, получает ссылку на глобал `^Workdays`, который может уже существовать или нет.

Вторая строка, `myGref[None] = 5`, сохраняет количество рабочих дней в `^Workdays` без сабскрипта (также узла, ключа).

Третья строка, `myGref[1] = 'Monday'`, сохраняет строку `Понедельник` в узле глобала `^Workdays(1)`. Следующие четыре строки хранят остальные рабочие дни в `^Workdays(2)` - `^Workdays(5)`.

Последняя строка, `print(myGref[3])`, показывает, как можно получить доступ к значению, хранящемуся в глобале, по его ссылке.

## Совместное использование ObjectScript и Python

InterSystems IRIS облегчает совместную работу смешанных команд программистов на ObjectScript и Python. Например, некоторые методы в классе могут быть написаны на ObjectScript, а некоторые - на Python. Программисты могут выбрать язык, на котором им удобнее всего писать, или язык, который больше подходит для решения поставленной задачи.

### Создание смешанных классов InterSystems IRIS

Следующий класс имеет метод `Print`, написанный на Python, и метод `Write`, написанный на ObjectScript, но функционально они эквивалентны, и любой из методов может быть вызван из Python или ObjectScript.

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

Этот пример кода на Python показывает, как открыть объект Company с Id=2 и вызвать методы `Print` и `Write`.

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

Следующий пример ObjectScript создает два массива Python, `moscow` и `spb`, каждый из которых содержит широту и долготу города:
```
>set builtins = ##class(%SYS.Python).Builtins()
>set moscow = builtins.list()
>do moscow.append(55.7558)
>do moscow.append(37.6173)
>set spb = builtins.list()
>do spb.append(59.9311)
>do spb.append(30.3609)

>zwrite moscow
moscow=11@%SYS.Python  ; [(55.7558, 37.6173]  ; <OREF>

>zwrite spb
spb=11@%SYS.Python  ; [59.9311, 30.3609]  ; <OREF>
```

Приведенный ниже код использует пакет `geopy`, который вы видели в предыдущем примере, для расчета расстояния между Москвой и Санкт-Петербургом. Он вычисляет расстояние с помощью метода `geopy.distance.distance`, передавая массивы в качестве параметров, а затем печатает свойство km результирующей дистанции:

```
>set distance = $system.Python.Import("geopy.distance")
>set route = distance.distance(moscow, spb)
>write route.km

633.3167668109714
```

### Запуск произвольной команды Python

Когда вы разрабатываете или тестируете что-то, иногда бывает полезно запустить строку кода Python, чтобы посмотреть, что она делает. В таких случаях можно использовать метод `%SYS.Python:Run`, как показано в примере ниже:

```
>set rslt = ##class(%SYS.Python).Run("print('hello world')")
hello world
```

## Различия

Из-за различий между ObjectScript и Python необходимо InterSystems разработали несколько вспомогательных функций, которые помогут вам преодолеть различия между языками.

### Builtins

Пакет builtins загружается автоматически при запуске интерпретатора Python и содержит все [встроенные функции](https://docs.python.org/3.9/library/functions.html#built-in-funcs), [константы](https://docs.python.org/3/library/constants.html#built-in-consts) и [классы](https://docs.python.org/3.9/library/stdtypes.html), такие как базовый класс объекта, классы типов данных, классы исключений.

Вы можете импортировать этот пакет в ObjectScript следующим образом:

```
set builtins = ##class(%SYS.Python).Import("builtins")
set builtins = $system.Python.Builtins()
```

Например функция Python `print` является методом модуля `builtins`, поэтому вы можете использовать эту функцию из ObjectScript:

```
>do builtins.print("hello world!")
hello world!
```

Команда zwrite может быть использована для просмотра объекта builtins (как и любого другого объекта), и, поскольку это объект Python, он использует метод `str` пакета builtins для получения текстового представления объекта. Например:

```
>zwrite builtins
builtins=5@%SYS.Python  ; <module 'builtins' (built-in)>  ; <OREF>
```

Теперь создадим пустой список Python:

```
USER>set list = builtins.list()
 
USER>zwrite list
list=5@%SYS.Python  ; []  ; <OREF>
```

Используйте `type` чтобы узнать какого класса объект:

```
>zwrite builtins.type(list)
3@%SYS.Python  ; <class 'list'>  ; <OREF>
```

Посмотрим, какие методы есть у этого объекта, используя метод `dir`:

```
>zwrite builtins.dir(list)
3@%SYS.Python  ; ['__add__', '__class__', '__class_getitem__', '__contains__', '__delattr__', '__delitem__', '__dir__', '__doc__', 
'__eq__', '__format__', '__ge__', '__getattribute__', '__getitem__', '__gt__', '__hash__', '__iadd__', '__imul__', '__init__', 
'__init_subclass__', '__iter__', '__le__', '__len__', '__lt__', '__mul__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', 
'__repr__', '__reversed__', '__rmul__', '__setattr__', '__setitem__', '__sizeof__', '__str__', '__subclasshook__', 'append', 
'clear', 'copy', 'count', 'extend', 'index', 'insert', 'pop', 'remove', 'reverse', 'sort']  ; <OREF>
 ```

Метод `help` выводит подробную информацию. Ещё больше информации можно получить, используя библиотеку [rich](https://github.com/willmcgugan/rich#rich-inspect).

```
>do builtins.help(list)
Help on list object:
class list(object)
 |  list(iterable=(), /)
 |
 |  Built-in mutable sequence.
 |
 |  If no argument is given, the constructor creates a new empty list.
 |  The argument must be an iterable if specified.
 |
 |  Methods defined here:
 |
 |  __add__(self, value, /)
 |      Return self+value.
 |
 |  __contains__(self, key, /)
 |      Return key in self.
 |
 |  __delitem__(self, key, /)
 |      Delete self[key].
```

### Идентификаторы

Правила именования идентификаторов отличаются между ObjectScript и Python. Например, подчеркивание (`_`) разрешено в именах методов Python, и широко используется для так называемых методов и атрибутов "dunder" ("dunder" - сокращение от "double underscore"), таких как `__getitem__` или `__class__`. Чтобы использовать такие идентификаторы в ObjectScript, обрамляйте их двойными кавычками:

```
>set mylist = builtins.list()
>zwrite mylist."__class__"
2@%SYS.Python  ; <class list>  ; <OREF>
```

И наоборот, методы InterSystems IRIS часто начинаются со знака процента (`%`). например, `%New` или `%Save`. Чтобы использовать такие идентификаторы из Python, замените знак процента на знак подчеркивания. Если у вас есть класс `User.Person`, то следующая строка кода Python создает новый объект `Person`.

```python
import iris
p = iris.cls('User.Person')._New()
```

### Именованные и позиционные аргументы

В Python возможно использовать именованные (ключевые) аргументы в сигнатуре метода. Это позволяет легко пропускать аргументы, если они не нужны, или указывать аргументы в соответствии с их именами, а не позициями. В качестве примера возьмем следующий метод:

```python
def mymethod(foo=1, bar=2, baz="three"):
    print(f"foo={foo}, bar={bar}, baz={baz}")
```

Поскольку в InterSystems IRIS отсутствует концепция именованных аргументов, необходимо создать [динамический объект](https://docs.intersystems.com/iris20212/csp/docbook/DocBook.UI.Page.cls?KEY=GJSON_create) для хранения пар ключевое слово/значение, например:

```
set args={ "bar": 123, "foo": "foo"}
```

Если метод `mymethod`  определён в модуле `mymodule`, то его можно вызвать так:

```
>set obj = ##class(%SYS.Python).Import("mymodule")
>set args={ "bar": 123, "foo": "foo"}
>do obj.mymethod(args...)
foo=foo, bar=123, baz=three
```

Поскольку `baz` не был передан в метод, ему по умолчанию присваивается значение `three`.

### Ссылочные аргументы

Аргументы в методах ObjectScript, могут передаваться по значению или по ссылке. В приведенном ниже методе ключевое слово `ByRef` перед вторым и третьим аргументами в сигнатуре указывает, что они предназначены для передачи по ссылке:

```
ClassMethod SandwichSwitch(bread As %String, ByRef filling1 As %String, ByRef filling2 As %String)
{
    set bread = "whole wheat"
    set filling1 = "almond butter"
    set filling2 = "cherry preserves"
}
```

При вызове метода из ObjectScript поставьте точку перед аргументом, чтобы передать его по ссылке, как показано ниже:

```
>set arg1 = "white bread"
>set arg2 = "peanut butter"
>set arg3 = "grape jelly"
>do ##class(User.EmbeddedPython).SandwichSwitch(arg1, .arg2, .arg3)
>write arg1
white bread

>write arg2
almond butter

>write arg3
cherry preserves
```

Видно, что значение переменной `arg1` после вызова `SandwichSwitch` осталось прежним, а значения переменных `arg2` и `arg3` изменились.

Поскольку Python не поддерживает вызов по ссылке изначально, необходимо использовать метод `iris.ref` для создания ссылки для передачи в метод для каждого аргумента, передаваемого по ссылке:

```
>>> import iris
>>> arg1 = "white bread"
>>> arg2 = iris.ref("peanut butter")
>>> arg3 = iris.ref("grape jelly")
>>> iris.cls('User.EmbeddedPython').SandwichSwitch(arg1, arg2, arg3)
>>> arg1
'white bread'

>>> arg2.value
'almond butter'

>>> arg3.value
'cherry preserves'
```

Вы можете использовать свойство `value` для доступа к значениям `arg2` и `arg3` и увидеть, что они изменились после вызова метода.

Важно! ByRef как и Output являются лишь рекомендацией. Передача по значению или по ссылке всегда зависит от вызывающей стороны. Пользователь, вызывающий код, может как передавать по ссылке аргументы без модификаторов ByRef, Output так и передавать значение в аргумент, ожидающий передачу по ссылке.

Важно! Объекты передаются по ссылке по умолчанию. Передача через точку передаёт вместо ссылки на объект ссылку на ссылку на объект, позволяя заменять ссылку на объект внутри вызываемого метода. Это достаточно редко используемая возможность актуальная, как правило, при написании коллбэков.


### True, False, None

Класс `%SYS.Python` определяет методы `True`, `False` и `None`, которые возвращают соответствующие Python объекты:

```
>zwrite ##class(%SYS.Python).True()
2@%SYS.Python  ; True  ; <OREF>
```

Используйте их для вызова Python методов. В случае если метод возвращает одно из этих значений они будут автоматически сконвертированы в `1`, `0` и `""` соответственно.

### Словари Dict

Для работы со [словарями](https://docs.python.org/3/tutorial/datastructures.html#dictionaries) Python используйте метод `dict` модуля `builtins`:

```
>set mycar = $system.Python.Builtins().dict()
>do mycar.setdefault("make", "Toyota") 
>do mycar.setdefault("model", "RAV4")
>do mycar.setdefault("color", "blue")

>zwrite mycar
mycar=2@%SYS.Python  ; {'make': 'Toyota', 'model': 'RAV4', 'color': 'blue'}  ; <OREF>
 
>write mycar."__getitem__"("color")
blue
```

В примере выше используется метод `setdefault` для установки значения ключа и `__getitem__` для получения значения ключа.

### Списки Lists

В Python [списки](https://docs.python.org/3/tutorial/datastructures.html) хранят коллекции значений. Доступ к элементам списка осуществляется по их индексу.

```python
>>> fruits = ["apple", "banana", "cherry"]
>>> print(fruits)
['apple', 'banana', 'cherry']
>>> print(fruits[0])
apple
```

В ObjectScript вы можете работать со списками Python, используя метод `list` модуля `builtins`:

```
>set l = ##class(%SYS.Python).Builtins().list()
>do l.append("apple")
>do l.append("banana")
>do l.append("cherry")

>zwrite l
l=13@%SYS.Python  ; ['apple', 'banana', 'cherry']  ; <OREF>
 
>write l."__getitem__"(0)
apple
```

В приведенном выше примере используется метод `append` для добавления элемента в список и `__getitem__` для получения значения по индексу. (списки Python начинаются с нуля).

### Исключения

Исключения Python и ObjectScript преобразуются друг в друга в зависимости от контекста.

### Статусы (%Status)

Большое число методов InterSystems IRIS озвращают структуру %Status. Для работы со [статусами](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=ASTATUS) используйте методы класса [%SYSTEM.Status](https://docs.intersystems.com/irislatest/csp/documatic/%25CSP.Documatic.cls?&LIBRARY=%25SYS&CLASSNAME=%25SYSTEM.Status):

```
>>> systemStatus = iris.cls('%SYSTEM.Status')
>>> sc = systemStatus.Error(5001,'MyText')     # Создать статус
>>> systemStatus.IsError(sc)                   # Проверить статус: 1 - ошибка, 0 - ОК
1

>>> systemStatus.GetErrorText(sc)              # Получить текст ошибки
'ERROR #5001: MyText'
```

Полный список кодов статусов доступен в [документации](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=RERR_gen). Также можно определять [пользовательские коды статусов](https://www.sql.ru/blogs/servit/1279).

### Строки и байты

Python разделяет объекты byte, которые представляют собой последовательности 8-битных байтов, и строки string, которые являются последовательностями байтов UTF-8, представляющими строку. В Python объекты типа byte никак не преобразуются, но строки могут быть преобразованы в зависимости от набора символов, используемого операционной системой хоста, например, CP1251.

InterSystems IRIS не делает различий между байтами и строками. Хотя InterSystems IRIS поддерживает строки Unicode (USC-2/UTF-16), любая строка, в которой все значения символов менее 256, может быть либо строкой, либо массивом байтов. По этой причине при передаче строк и байтов в Python и из него действуют следующие правила:

- Строки InterSystems IRIS считаются строками и преобразуются в UTF-8 при передаче из ObjectScript в Python.
- Строки Python преобразуются из UTF-8 в строки InterSystems IRIS при передаче обратно в ObjectScript, что может привести к появлению широких символов.
- Объекты byte из Python возвращаются в ObjectScript как 8-битные строки. Если длина байтового объекта превышает максимальную длину строки, то возвращается байтовый объект Python.
- Чтобы передать байтовые объекты в Python из ObjectScript, используйте метод `%SYS.Python:Bytes`, который НЕ преобразует базовую строку InterSystems IRIS в UTF-8.

Следующий пример превращает строку InterSystems IRIS в объект Python типа bytes:

```
>set b = ##class(%SYS.Python).Bytes("Hello Bytes!")

>zwrite b
b=8@%SYS.Python  ; b'Hello Bytes!'  ; <OREF>
 
>zwrite builtins.type(b)
4@%SYS.Python  ; <class 'bytes'>  ; <OREF>
```

Для создания байтовых объектов Python, длина которых превышает максимальную длину строки 3,8 МБ в InterSystems IRIS, вы можете использовать объект bytearray и добавлять к нему меньшие фрагменты байтов с помощью метода `extend`. Наконец, передайте объект bytearray в метод `bytes`, чтобы получить байты:

```
>set ba = builtins.bytearray()
>do ba.extend(##class(%SYS.Python).Bytes("chunk 1"))
>do ba.extend(##class(%SYS.Python).Bytes("chunk 2"))
 >zwrite builtins.bytes(ba)
"chunk 1chunk 2"
```

### IO

При использовании Python стандартный вывод (stdout) перенаправляется в консоль InterSystems IRIS, что означает, что вывод `print` отправляется в терминал. Стандартная ошибка (stderr) перенаправляется в messages.log, расположенный в каталоге `<install-dir>/mgr`.

В качестве примера рассмотрим метод Python:

```
def divide(a, b):
    try:
        print(a/b)
    except ZeroDivisionError:
        print("Cannot divide by zero")
    except TypeError:
        import sys
        print("Bad argument type", file=sys.stderr)
    except:
        print("Something else went wrong")
```

Протестируем его в терминале:

```
>set obj = ##class(%SYS.Python).Import("mymodule")
 
>do obj.divide(5, 0)
Cannot divide by zero
 
>do obj.divide(5, "hello")
```

Если вы пытаетесь разделить на ноль, сообщение об ошибке направляется в терминал, но если вы пытаетесь разделить на строку, сообщение отправляется в `messages.log`:

```
11/19/21-15:49:33:248 (28804) 0 [Python] Bad argument type
```

В файл `messages.log` следует отправлять только важные сообщения.

### Сигналы

И IRIS, и Python регистрируют обработчики сигналов для обеспечения надлежащей интеграции с операционной системой, однако иногда они могут конфликтовать. В идеале, мы должны работать с обработчиками сигналов IRIS во время выполнения кода IRIS и переходить на обработчики сигналов Python во время выполнении кода Python. Бенчмаркинг показал, что вызовы `sigaction`/`signal` являются достаточно дорогими, поэтому по умолчанию мы оставляем обработчики сигналов IRIS. Это означает, что для долго работающего метода Python, Ctrl-C не будет передан.

Если вам необходимо, чтобы при вызове метода Python была включена обработка сигналов Python, вы можете использовать метод `$system.Python.ChangeSignalState`:

```
set oldstate = $system.Python.ChangeSignalState(0) # Включение обработки сигналов для Python
do obj."slow_python_method"()                      # Ctrl-C теперь работает и обрабатывается Python
do $system.Python.ChangeSignalState(oldstate)      # Отключение обработки сигналов для Python
```

### Отладка

Иногда необходимо отладить код Python, например, чтобы диагностировать исключение в коде Python. Можно использовать [отладчик Python](https://docs.python.org/3.9/library/pdb.html) внутри InterSystems IRIS. Однако вам необходимо включить его вызовом, чтобы предотвратить обработку и очистку ошибок при вызове Python кода. Это делается с помощью метода `$system.Python.Debugging`:

```
do $system.Debugging(1)                # InterSystems IRIS теперь НЕ обрабатывает исключения Python
set pdb = $system.Python.Import("pdb") # импорт дебаггера
do obj."erroneous_python_method"()
do pdb.pm()                            # вывод дебаг информации
do $system.Debugging(0)                # InterSystems IRIS вновь обрабатывает исключения Python
```

### Профилирование

Профилирование кода Python возможно с помощью пакетов [cProfile](https://docs.python.org/3.9/library/profile.html) и [pstats](https://docs.python.org/3.9/library/profile.html?highlight=pstats#module-pstats):

```
set cp = $system.Python.Import("cProfile")
set pstats = $system.Python.Import("pstats")
set profiler = cp.Profile()
do profiler.enable()
do obj."python_method_to_profile"()
do profiler.disable()
set ps = pstats.Stats(profiler)
do ps."sort_stats"(pstats.SortKey.CUMULATIVE)
do ps."print_stats"()
```

Этот пример создаст отчет о профилировании на основе времени, проведенного в функциях Python.

### Интеграционные решения

Если вы разрабатываете собственные классы бизнес-хостов или адаптеров для интеграционных решений InterSystems IRIS, любые коллбэк методы (такие как `OnProcessInput` или `OnInit`) должны быть написаны на ObjectScript. Код ObjectScript может, использовать библиотеки Python или вызывать другие методы, реализованные полностью на Python. 

Дополнительно, при разработке BP/BPL процессов помните что:
- Необходимо сохранять все нужные вам объекты между состояниями как свойства контекста.
- Необходимо инициализировать модули перед каждой работой с ними в новом состоянии процесса.
- Объекты Python не могут быть сохранены asis (используйте pickle/dill).


## Выводы

Embedded Python позволяет использовать язык программирования Python вместе с InterSystems ObjectScript. Если вы разработчик на ObjectScript теперь вы можете получить доступ ко всем библиотекам Python прямо в InterSystems IRIS. Если же вы разработчик на Python, теперь вы можете писать код в InterSystems IRIS на языке Python. 

## Ссылки

- Документация: [1](https://docs.intersystems.com/iris20212/csp/docbook/DocBook.UI.Page.cls?KEY=AFL_epython#AFL_epython_irisapi), [2](https://docs.intersystems.com/iris20212/csp/docbook/DocBook.UI.Page.cls?KEY=AEPYTHON)
- [Сессии Python на Virtual Summit 2021](https://www.intersystems.com/virtual-summit-2021/#python)
- [Репозиторий с примерами](https://github.com/intersystems-community/OpcUA-empy)
- [Вебинар]()
