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

Пакеты Python можно устанавливать из командной строки с помощью программы установки пакетов pip3: `pip3 install --target <installdir>/mgr/python <package>`. Например, вы можете установить пакет `numpy` на машину Windows следующим образом:

```
pip3 install --target C:\InterSystems\IRIS\mgr\python numpy
```

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

Важно отметить, что `Nominatum` принимает только именованные (ключевые, keyword) аргументы, что напрямую не поддерживается в ObjectScript. Решением является создание динамического объекта, содержащего необходимые аргументы (который в данном случае устанавливает ключевое слово `user_agent` равным `Embedded Python`), а затем передача его в Python с помощью синтаксиса `args...` (то расширяет [существующий синтаксис args...](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=GCOS_usercode#GCOS_usercode_args_variable)).

В отличие от математического модуля, импортированного в предыдущем примере, вызов `zwrite` на объекте `geopy` является экземпляром пакета `geopy`, установленного в `C:\Intersystems\iris\mgr\python`:
```
>zwrite geopy
geopy=2@%SYS.Python ; <module 'geopy' from 'c:\\\intersystems\\iris\\mgr\\python\\\geopy\\\__init__.py'> ; <OREF>
```
