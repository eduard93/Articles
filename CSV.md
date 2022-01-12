# Загрузка CSV файлов в InterSystems IRIS

Не секрет, что несмотря на большое количество различных форматов хранения данных, CSV остаётся если не лидирующим, то, по крайней мере крупным игроком на этом рынке. Человекочитаемость (по сравнению с AVRO и Apache Parquet) и компактность (по сравнению с JSON и XML) а также простота генерации делают его популярным выбором как формата обмена данных для огромного количества информационных систем. InterSystems IRIS не является исключением, предоставляя ряд возможностей экспорта/импорта в CSV. Но в версии 2021.2 появилась новая SQL-функция [LOAD DATA](https://docs.intersystems.com/iris20212/csp/docbook/DocBook.UI.Page.cls?KEY=RSQL_loaddata) позволяющая легко загружать CSV файлы (и не только) в хранимые таблицы InterSystems IRIS. Об этом новом методе (с небольшим обзором других подходов) и будет эта статья.

# LOAD DATA 

Команда ```LOAD DATA``` загружает данные из источника данных в таблицу InterSystems IRIS. Источником может быть CSV файл или таблица во внешней СУБД, доступ к которой осуществляется по протоколу JDBC.

Эта команда предназначена для быстрого заполнения таблиц. Когда вы загружаете данные, %ROWCOUNT показывает количество успешно загруженных записей. Ошибка во входных данных приводит к тому, что эта запись не загружается, и загрузка переходит к следующей записи. SQLCODE не сообщает об этом как об ошибке; в журнале ```%SQL_Diag.Result``` указывается, сколько записей не удалось загрузить.

Если таблица в которую загружаются данные пуста, LOAD DATA заполняет таблицу строками исходных данных. Если таблица уже содержит данные, LOAD DATA добавляет строки исходных данных к существующим данным.

Примечание: Команда LOAD DATA использует Java шлюз. Перед выполнением команды LOAD DATA на сервере должна быть установлена виртуальная машина Java (JVM). Вы также должны установить соединение ([External Language Server](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=BJAVNAT_gateway#BJAVNAT_gateway_intro)) с сервером  Java. Это соединение запускается автоматически при первом использовании командой LOAD DATA.

Примечание: в превью версии 2021.2.0.617 необходимо отредактировать ```%Java Server``` и добавить аргумент JVM: `-Dfile.encoding=UTF-8`. В новых версиях этого делать не нужно.

## Определение команды:

```
LOAD DATA FROM FILE filepath 
    [ COLUMNS (fieldname datatype, fieldname2 datatype2, ...) ] 
INTO table [ (fieldname, fieldname2, ...) 
    [ VALUES (headeritem,headeritem2, ...) ] ]
    [ USING {json_object} ]

LOAD DATA FROM JDBC connection TABLE jtable
INTO table [ (fieldname, fieldname2, ...) 
    [ VALUES (jfieldname,jfieldname2, ...) ] ]
```

| Аргумент                        | Описание                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| filepath                        | (Только для файлов) Путь к файлу на сервере в кавычках.                                                                                                                                                                                                                                                                                                                                                                                                    |
| COLUMNS (fieldname datatype)    | (Опционально, только для файлов) Последовательность колонок в файле и их типы данных                                                                                                                                                                                                                                                                                                                                                                        |
| INTO table                      | Таблица, в которую будут загружены данные. Имя таблицы может быть квалифицированным (schema.tablename) или неквалифицированным (tablename). Неквалифицированное имя таблицы принимает имя схемы по умолчанию. Можно указать представление для загрузки данных в таблицу, доступ к которой осуществляется через представление.                                                                                                                               |
| (fieldname,fieldname)           | (Опционально) Поля таблицы для загрузки данных файла, указанные в порядке следования полей данных файла. Этот список имен полей позволяет указать выбранные поля таблицы и согласовать порядок элементов файла данных с полями в таблице. Поле, которое не указано, принимает значение по умолчанию, если оно указано в определении таблицы. Если этот пункт опущен, все поля таблицы, определенные пользователем, должны быть представлены в файле данных. |
| VALUES (headeritem,headeritem2) | (Опционально) Для источника файла данных - имена заголовков файла данных (headeritem) или имена COLUMNS. Для источника данных JDBC - имена столбцов таблицы JDBC (jfieldname). Элементы должны позиционно соответствовать именам полей в INTO.                                                                                                                                                                                                              |
| USING {json\_object}            | (Опционально) Параметры загрузки для входного файла данных, используя синтаксис вложенной пары ключ:значение объекта JSON. Например, USING {"from":{"file":{"columnseparator":"^"}}}. Используется для указания символа-разделителя столбцов файла исходных данных, наличия строки заголовка файла исходных данных и других параметров. Если пункт USING не указан, используются параметры нагрузки по умолчанию.                                           |
| connection                      | (Только для JDBC) Имя SQL Gateway Connection                                                                                                                                                                                                                                                                                                                                                                                                                |
| TABLE jtable                    | (Только для JDBC) Таблица SQL Gateway Connection                                                                                                                                                                                                                                                                                                                                                                                                            |

Вот пример команды LOAD DATA:

```
LOAD DATA FROM FILE 'C://TEMP/mydata.txt' 
INTO MyTable
```

В данном случае колонки таблицы сортируются по [SqlColumnNumber](https://docs.intersystems.com/iris20212/csp/docbook/DocBook.UI.Page.cls?KEY=ROBJ_property_sqlcolumnnumber) а колонки файла по алфавиту.
Поэтому в большинстве случаев следует явно указывать колонки, например:

```
LOAD DATA FROM FILE 'C://TEMP/mydata.txt' 
COLUMNS (head1 INT,head2 VARCHAR(20),head3 INT,head4 VARCHAR(20),head5 INT)
INTO MyTable(field1,field3) VALUES (head2,head5)
```

## Даты (и любые другие функциональные преобразования)

В InterSystems IRIS даты хранятся в формате [$horolog](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=RCOS_vhorolog). LOAD DATA ожидает даты в ODBC формате (yyyy-mm-dd).
Если даты в другом формате (или необходимо выполнить поизвольное функциональное преобразование), можно использовать 2 подхода:

1. Временное и вычисляемое свойства. Допустим у нас даты в формате `12/01/2021` (`dd/mm/yyyy`). Создадим 2 свойства:

```
Property InputDate As %String;

Property MyDate As %Date [ SqlComputeCode = { set {*} = $ZDH({InputDate},4)}, SqlComputed, SqlComputeOnChange = InputDate ];
```

`InputDate` будет заполняться функцией LOAD DATA (это строка и соответственно никаких проблем с форматом не будет), а свойство `MyDate` будет автоматически вычисляться при каждом изменении `InputDate`.
Недостатком данного подхода является необходимость хранить оба свойства.

2. Классы типов данных. В InterSystems IRIS можно создавать свои [типы данных](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=RSQL_datatype#RSQL_datatype_userdef):

```objectscript
Class User.MyDate Extends %Date
{

ClassMethod OdbcToLogical(%val As %String = "") As %Date [ CodeMode = generator, ServerOnly = 1 ]
{
	$$$GENERATE(" quit:%val=""""||($zu(115,13)&&(%val=$c(0))) """" quit:$isvalidnum(%val,0,-672045,2980013) %val set %val=$zdateh(%val,4,,,,,-672045,,""Error: '""_%val_""' is an invalid dd/mm/yyyy Date value"") q:%val||(%val=0) %val s %msg=%val ZTRAP ""ODAT""")
}

}
```

После компиляции класса и регистрации соответствующего ему SQL типа данных его можно исползовать при вызове LOAD DATA.

## Логирование и ошибки

Успешный вызов LOAD DATA создает запись в таблице ```%SQL_Diag.Result``` и таблице ```%SQL_Diag.Message```. 

```sql
SELECT * FROM %SQL_Diag.Result
```

В столбце ```errorCount``` таблицы ```%SQL_Diag.Result```  указано количество записей, которые не удалось загрузить.

В таблице ```%SQL_Diag.Message``` содержится подробная информация о каждой записи, которую не удалось загрузить.

```sql
SELECT * FROM %SQL_Diag.Message WHERE severity = 'error'
```

Обратите внимание, что метка времени в этих таблицах хранится в формате UTC, а не местном времени.

# %SQL_Util.CSV

Удобен для 

```
CALL %SQL_Util.CSV(,'ROW(MYID VARCHAR(10000), FIXED VARCHAR(10000))','C:\new.csv',';',,'CP1251')
```

# %SQL_Util.CSVTOCLASS

https://docs.intersystems.com/irislatest/csp/documatic/%25CSP.Documatic.cls?LIBRARY=%25SYS&CLASSNAME=%25SQL.Util.Procedures#CSVTOCLASS

# Глобалы

# Record Mapper
