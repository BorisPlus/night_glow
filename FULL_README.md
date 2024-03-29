# Фреймворк "NiGHT GLOW"

## Что сподвигло к разработке фреймворка "NiGHT GLOW"

Рассмотрим специально подготовленную мной реализацию демонстрационного Python-скрипта ("ничего особо важного не делающего"), использующего базовый подход к разработке методик на базе Python-API ПО "Лампир":

```python
# Импорт базового Python-API ПО "Лампир"
import user_task_base as lmp

# Табличный формат вывода данных
class TestHeader(metaclass=lmp.Header):
    # Имя в интерфейсе пользователся
    display_name = "Тест1. Результат"
    # Строковый столбец таблицы
    value = lmp.Field('значение', lmp.ValueType.String)

# Методика
class TestTask(lmp.UserTaskBase):
    def __init__(self):
        super().__init__()

    # У методики должен быть уникальный идентификатор
    def get_id(self):
        return '18303851-0000-0000-0000-000000000000'

    # Методика должна быть отнесена к какой-то категории
    # что отобразиться в виде каталога методик
    # в интерфейсе пользователя ПО "Лампир"
    def get_category(self):
        return "Тестирование"

    # Методика должна иметь имя
    def get_display_name(self):
        return "Тест1. Напечатаю от 1 до ..."

    # Методика должна иметь список входных параметров
    def get_enter_params(self):
        ep_coll = lmp.EnterParamCollection()
        ep_coll.add_enter_param(
            "end_value",
            "печатать от 1 до ...",
            lmp.ValueType.Integer,
            required=True,
            default_value=10
        )
        return ep_coll

    # Методика должна иметь закрепленный список выходных форматов данных
    # которые отобразятся в интерфейсе пользователя ПО "Лампир"
    # в виде таблиц с результатами методики
    def get_headers(self):
        return lmp.HeaderCollection(
            TestHeader
        )

    # Реализация получения и экспорта данных
    def execute(self, enter_params, result_writer, log_writer, tmp_dir):
        log_writer.info('START execute')

        # получение внешних данных из БД, файлов или через какое-либо API
        some_calculated_data = [
            {'value': '№. %s' % value}
            for value in range(1, enter_params.end_value)
        ]

        # экспорта данных в заранее определенный формат
        fields_list = TestHeader.get_fields()
        for data_row in some_calculated_data:
            result_data_row = TestHeader.create_empty()
            for field_name in fields_list:
                if field_name in data_row:
                    result_data_row[fields_list[field_name]] = data_row[field_name]
            result_writer.write_line(result_data_row, header_class=TestHeader)

        log_writer.info('END execute')

```

### Рефакторинг

Не пугайтесь этого слова, если вы видите его впервые. Он по сути обозначает заявленный авторами ПО "Лампир" подход реализации "онтологии проектирования" методик. По рефакторингу много книг, но достойная - Фаулер.

С целью упрощения восприятия я буду утрировать и в рассмотренном Python-коде опускать некоторые его блоки, не затрагивающие суть изменений, указываемых в контексте того или иного рассуждения . Будет немного объектно-ориентированного программирования. Кто в курсе ООП, тот поймет (и простит), кто нет - то "Пора, мой друг, пора...".

Итак, что мне здесь НЕ нравится.

#### Объявление категории и экранного имени методики

```bash
def get_category(self):
    return "Тестирование"
    
def get_display_name(self):
    return "Тест1. Напечатаю от 1 до ..."
```

По сути эти методы возвращают одно и тоже для всех экземпляров одного класса - одни и те же значения категории методики и экранного имени. Сейчас это методы объектов класса, а должны быть методы всего класса (не путайте со статическими методами, в Python есть круче фишка - см. штатный декоратор "@classmethod").

Мое мнение, вот такая конструкция была б более изящна и код локоничнее (возможность писать так в фреймворке "NiGHT GLOW" уже реализована).

```python

сategory = "Тестирование"
display_name = "Тест2. Напечатаю от 1 до ..."
    
```

Теперь задумайтейсь, у вас есть несколько таких тестовых процедур и каждая отнесена к данной категории, и вдруг вы передумали и решили поменять название категории "Тестирование" на "Тестовая категория". Придется менять в каждой. Для 1-2 методики это нормально, но для 10 это долго. 

Выносим название категорий в отдельный модуль констрант и называем этот блок констант `task_category` класса (в фреймворке "NiGHT GLOW" реализовано). Вынесем ее в ядро, и теперь любой желающий сможет дать название категории своей методики.

```python
from night_glow.constants import task_categories

сategory = task_categories.TEST
    
```

А вам не лень вообще писать для целого блока методик вышеуказанную строку. Мне - лень.

Делаем базовый класс тестовых методик и наследуем от него наши остальные

```python
from night_glow.constants import task_categories
...
class BaseTestTask(...):
    category = task_categories.TEST
...

class TestTwoTask(BaseTestTask):
    pass
...
        
class TestThreeTask(BaseTestTask):
    pass
...

```

Ура, мы явно не писали для `TestTwoTask` и `TestThreeTask` методик их категории, но они унаследовались от родительской `BaseTestTask`. Если же Вам будет жизненно необходимо изменить категорию для вашей методики, пусть и у унаследованной от какой-то родительской, у которой уже категория задана (или то же унаследована), то вы всегда можете явно переопределить категорию.
 
```
class TestFourTask(BaseTestTask):
    category = task_categories.SOME_OTHER_CATEGORY
```

#### Уникальный идентификатор методики

Имеет место быть такая конструкция

```python

def get_id(self):
    return '00183038-0000-0000-0000-000000000000'

```

Проворачивать с ней все тоже самое, что с категорией и экранным именем, просто не имеет смыла.

Во-первых, избыточно иметь идентификаторы всех методик в отдельном файле констант, как мы это видели для названий категорий. Почему?

Потому, что с высокой долей вероятности вы не станене присваивать двум не уникальным процедурам уникальные идентификаторы, так как ПО "Лампир" не позволяет импорт процедур с одинаковыми идентификаторами. Ну может быть в единичных случаях для отладки отдельных процедур вам это понадобится, но ради этого заводить файл констант и наследование - нерационально.

Во-вторых, в пользовательском интерфейсе ПО "Лампир", вы сами запутаетесь, если у вас будет две методики с одинаковым экранным именем в одной категории. Нет, конечно это возможно в ПО "Лампир", чтоб имелось несколько методик с одинаковым экранным именем в одной категории. Сделайте, чтоб `get_id(self)` возвращал разные идентификаторы, но скорее всего вы в итоге при выборе сами запутаетесь и будете пересчелкивать. Это все равно, что было у вас три одноклассника, все Алексеи по паспорту, в итоге только один останется "Лехой", "Лешой" или "Алексеем", другие - получат свои погоняла. Так вот.

Итак, уникальность методики в интерфейсе ПО "Лампир" будет определяться все теми же категорией и экранным именем. Так пусть тогда мы и будем получать id нужного формата, но сгенерированный на основе категории и экранного именем.

```python
import uuid
...
uuid.uuid3(uuid.NAMESPACE_OID, '%s%s' % (get_category(), get_display_name()))
```

Мы не забываем, что методы опять же возвращают одно и тоже для всех экземпляров одного класса - одни и теже значения и что нам лень каждый раз писать

```python
    def get_id():
        return ...
```

У нас уже есть абстрактная с точки зрения функционала работы методика, от которой можно наследовать другие методики. Так пусть в ней и будет этот метод вычисления уникального идентифиактора.

```python
@classmethod
def get_id(cls):
    return str(uuid.uuid3(uuid.NAMESPACE_OID, '%s%s' % (
        cls.get_category(), cls.get_display_name()
    )))
```

#### Набор входных параметров и типы выходных данных

Что мне не нравится при ежечастном использовании методов `get_enter_params`  и `get_headers` ?
Да все тоже, что и для `get_category` и `get_display_name` - они опираются на то, что принадлежит всему классу.

Но у них есть еще один недостаток - они на самом деле содержат функциональный код установки свойств объектов, а не их получения. То есть они содержать в названии "get", а на самом деле делают "set".

```python
 def get_enter_params(self):
    ep_coll = lampyre.EnterParamCollection()
    ep_coll.add_enter_param(...)  # это set внутри get
    return ep_coll
    
def get_headers(self):
    return lampyre.HeaderCollection(TestHeader) # это set внутри get
    )
```

Я опущу подробности реализации базового класса, но как итог, при использовании фреймворка "NiGHT GLOW" в итоговом классе методики нужно лишь задать указанные поля

```python
class TestTwoTask(BaseTestTask):
    display_name = "Тест2"
    enter_params = (
        lampyre.EnterParamField(
            "end_value",
            "печатать от 2 до ...",
            lmp.ValueType.Integer,
            required=True,
            default_value=10
        ),
    )
    headers = (TestHeader,)

```
Опять - это чуть-чуть меньше изначального и это локоничнее, читабельнее и яснее.

#### Пухлый метод испонения

Иногда метод `execute` весьма прост для восприятия, но иногда, особенно, когда отлаживаешь логику чужой методики, которая почему-то не работает в твоем случае, достаточно сложно разобраться в витееватости кода.

Итак, я рекомендую вам выносить из execute методы, отвечающие за получение данных. Как это сделать лучше, чтоб не отдельным методом методики? Напишу чуть ниже. Сейчас о другом.

##### Экспорт полученных данных

Взгляните на конструкцию

```python
fields_list = TestHeader.get_fields()
for data_row in some_calculated_data:
    result_data_row = TestHeader.create_empty()
    for field_name in fields_list:
        if field_name in data_row:
            result_data_row[fields_list[field_name]] = data_row[field_name]
    result_writer.write_line(result_data_row, header_class=TestHeader)
```

Лично я и именно ее увидел в абсолютно разных методиках разных разработчиков. Она копируется от методики к методике. Есть некоторые причесывания и подгонки под определенный `header_class`, но суть одна:

```text
"пробежать по полученным данным, пробежаться по выходному типу данных, как-то сопоставить ячейки входных и выходных данных (тут по простому совпадению ключа), и экспортировать соответствующие данные (возмжно проведя над ними какие-нибудь манипуляции)".
```

Еще внутри это блока иногда встречвется вывод в консоль отладки ПО "Лампир". Смотрите, еще тут `TestHeader` присутствует три раза. Вы скопируете и в другом экспортере у вас будет `SomeOtherHeader`, и опять три раза. Не забудьте поменять везде, а тож искать будите долго. Вынесем эту конструкцию в отдельный модуль отчетов `reports`.

Как итог (не углубляясь в реализацию) вышеприведенный блок кода можно сделать таким

```python
# Отчет
AsIsReport(
    result_writer
).make(
    # Входные данные
    from_data=some_calculated_data,
    # Выходной формат
    to_header_format=TestHeader
)
```

Прочтем код. Достовно.

```text
Сделай отчет "как есть" (AsIsReport) по полученным данным (from_data) в формате выходного типа данных (to_header_format).
```

Можно передаль указатель на логгирование в процессе отчета, если это необходимо (в фреймворке "NiGHT GLOW" реализовано).

##### Отчеты с "причесыванием"

В одной из методик я встретил варианты конвертирования данных ячеек и занесения значений столбцов с определенным именованием из исходных данных в столбцы итоговых данных с другим именованием. Что-то вроде "значение из столбца NICKNAME перенести в столбец USERNAME". Таких соответствий может быть несколько, при этом возможно над значениями нужно будет провести какие-то преобразования, например, "номер из столбца id перенести в столбец url, дописав к id префикс 'https://site.com/user/'".

Все это можно формализовать. Просто надо "сказать" классу составителей отчетов, что нужно  над сопоставляемыми исходными и итоговыми столбцами данных произвести преобрпзование в соответствии с указателем на функцию преобразования.

Это уже реализовано в фреймворке "NiGHT GLOW".

```python
ReportWithFieldProcess(
    result_writer
).make(
    from_data=some_calculated_data,
    to_header_format=Identifiers,
    processor=Substitution(dict(identifier='value'))
)
```

Прочтем код. Достовно.

```text
Сделай отчет "с преобразованием полей" (ReportWithFieldProcess) по полученным данным (from_data=) в формате выходного типа данных (to_header_format=) и обработчиком (processor=) сопоставления имен исходных и итоговых столбцов(Substitution(dict(identifier='value')))'.
```

Можно указать несколько таких "соответствий". 

Обработчик полей может быть любым. Например, 

```python
    processor=SubstitutionWithNoneReplace(
        dict(identifier='value'),
        none_replace='нет данных'
    )
```

Этот обработчик дополнительно к замене полей вместо None значений поставит "нет данных". В рамках реализации класс SubstitutionWithNoneReplace наследуется от Substitution. 

Это гибко. 

И это уже реализовано в фреймворке "NiGHT GLOW".

Можно сделать более комплексный составитель отчетов, который опирается не только на значения полей, но и на весь массив исходных данных, например, для группировки или иного полного обсчета. Была б задача, это реализуемо. Это просто новый класс отчета и новый класс обработчика.


##### Получение данных

Вы уверены, что у вас не возникнет необходимости в получении данных из одного источника в разных методиках? Лично у меня нет такой уверенности.

В фреймворке "NiGHT GLOW" введено понятие "источника" данных и реализован соответствующий модуль. Источники можно группировать по видам, например, по названиям программных средств или информационных ресурсов, откуда берутся данные. Понятно, что конкретный способ выборки данных (SQL запрос, REST API обращение, чтение из файла и т.п.) будет инкапсулировано в базовом классе источника. Критерии выборки данных будут реализованы в наследниках базового. А вот сами значения параметров запроса - это значения входных параметров методик.

Тогда выше указанная конструкция примет вид
 
```python
AsIsReport(
    result_writer
).make(
    from_data=TestSource().get_data(**entered_kwargs_params),
    to_header_format=TestHeader
)
```

Конечно, для наложения критерия на отбираемые данные могут понадобится параметры, не передаваемые из пользовательского интерфеса ПО "Лампир", и простого `enter_params` станет не достаточно. Но тогда можно переопределить вызов `get_data`, передавая, например, некоторые вычисляемые аргументы или же вычислять их уже внутри метода `get_data`.

Истоники данных в большинстве случаев должны иметь настраиваемые параметры подключения к ресурсам, откуда и берутся данные. Так вот. Сами параметры подключения целесообразно передавать в конструкторе при инициализации объекта, например, так

```python
.make(
    from_data=TestSource(ip='127.0.0.1', port='8086', login='login').get_data(enter_params),
    to_header_format=TestHeader
)

```

Но вспомним, что мы же - **лентяи**, зачем нам каждый раз при подключению к одному и тому же источнику с выборкой данных по конкретному критерию заботиться о передаче этих параметров. Проще сделать один раз и забыть.

Заведем в модуле констант параметры подключения к источнику `connection_params`.

```
from night_glow.constants.some_service import connections_params

class SomeServiceTestSource(SomeServiceSource):
    def __init__(self):
        super().__init__(**connections_params.SOME_SERVICE_CONNECTION_PARAMS)
```

## Как итог

Я не уверен, что сейчас кто-то кинется переписывать методики, используя фреймворк "NiGHT GLOW". Но мне кажется, что уровень вхождения для использования ваших наработок будет ниже, так как писать меньше.

А точнее достаточно при аболютно рабочей методике сделать следующее.

Если для методики нужны "персональные" параметры подключения к истонику данных, например, "персональные" логин и пароль или токен доступа к API какого-то Интернет-рпесурса, то нужно:
1. Создать пользовательский файл с "персональными" параметрами подключения к источнику.
2. Унаследовать свой источник от используемого в методике источника, но в конструкторе передать "персональные" параметры подключения к источнику - свои из п.1.
3. Унаследовать свою методику от указанной методики, объявив в качестве класса источника - свой из п.2.

[см. конкретный пример](EXAMPLE.md)

Если же методика не содержит "персональности", то просто подключите ее в ПО "Лампир".


## Резюмирую

Я вижу, что часто при разработке процедур для ПО "Лампир" разрабтчиками используется понятие "онтология".

Читаем:

_**Онтоло́гия (в информатике)** — это попытка всеобъемлющей и детальной формализации некоторой области знаний с помощью концептуальной схемы. Обычно такая схема состоит из структуры данных, содержащей все релевантные классы объектов, их связи и правила(теоремы, ограничения), принятые в этой области._

Теперь при использовании фреймворка "NiGHT GLOW" ваша конкретная реализация действительно будет опираться на "структуру данных, содержащую все релевантные классы объектов, их связи и правила".

Сравните код с написанным в самом начале

```python
import sys
import user_task_base as lmp
# Все потому, что рабочая директория находится выше,
# так как определяется каталогом, где сам Lampyre.exe
sys.path.append('./user_tasks')
from night_glow.reports.common import AsIsReport
from ontology_test.headers.common.test import TestHeader
from ontology_test.tasks.test import base_test_task
from ontology_test.sources.test import TestSource

class TestTwoTask(base_test_task.BaseTestTask):
    display_name = "Тест2. Напечатаю от 1 до ..."
    enter_params = (
        lmp.EnterParamField(
            "end_value",
            "печатать от 1 до ...",
            lmp.ValueType.Integer,
            required=True,
            default_value=10
        ),
    )
    headers = (TestHeader,)

    def execute(self, enter_params, result_writer, log_writer, *args, **kwargs):
        AsIsReport(
            result_writer,
            log_writer=log_writer
        ).make(
            from_data=TestSource().get_data(enter_params),
            to_header_format=TestHeader
        )
```

## Чего не хватает мне прямо сейчас в Python-API ПО "Лампир"

* Штатно не возможно сделать в процедуре вывод разных данных в две таблицы одного типа, все однотипные данные будут залиты в соответствующую этому типу единственную таблицу. Выходит "типы выходных данных" - это не типы, а сами данные. Хотя я соглашусь с тем, что иметь строгое соответствие того, что "такие-то" данные на входе, а "такие-то" наборы данных на выходе - это очень удобно на перспективу построения цепочки методик. 
* Штатной возможности чтения данных из результатов других методик.
* Нельзя дописывать в результат прошлой методики (обогощать или просто дописывать)
* Exception('is_array option available only for string enter parameters') - чЁ?
* Методике может быть необходима более сложная форма параметров, которая может состоять из множества наборов параметров, например, множество одних и тех же наборов параметров - период, LAC, CELLID
* задать, чтоб хотя бы один `required=True` из параметров, а не каждый

## Заметки для разрабов

* придерживаемся PEP8, так как пишем на Python, если иное не оговорено
* из название методик, форматов, отчетов, истоников, а также их файлов и модудей и т.п., должно быть сразу ясно - откуда данные и что с ними деается в задаче
* соглашение об именовании, документировании и упр версиями стандарт суфф и преф_