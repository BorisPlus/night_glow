# Фреймворк "NiGHT GLOW"

## Пример реализации методики с использованием "персональных" параметров подключения к истонику данных

Допустим имеется методика, внесенная в ядро фреймворка "NiGHT GLOW", например, `night_glow\tasks\rail\http.py` (содержание можно опустить)

```python

import datetime
# импортируем API Лампир
import user_task_base as lmp
# импортируем источник сведений для данной методики
from night_glow.sources.rail import RailTSessionsSource
# импортируем формат сведений, получаемых в результате данной методики
from night_glow.headers.rail import RailTSessions
# импортируем составитель отчета
from night_glow.reports.common import AsIsReport
# импортируем базовый класс для всех методик на Рейке
from night_glow.tasks.rail import base_rail_task


# Методика получения сведений о HTTP биллинге c моего сервера
# который сконфигурирован как у большинства моих сотоварищей

class RailHTTPBilling(base_rail_task.BaseRailTask):
    # экранное имя
    display_name = "HTTP-биллинг"
    # формат сведений, получаемых в результате данной методики
    headers = (RailTSessions,)
    # входные параметры данной методики
    enter_params = (
        lmp.EnterParamField(
            "start_date",
            "c",
            lmp.ValueType.Datetime,
            required=True,
            default_value=datetime.datetime.today()
        ),
        lmp.EnterParamField(
            "end_date",
            "по",
            lmp.ValueType.Datetime,
            required=True,
            default_value=datetime.datetime.today()
        ),
        lmp.EnterParamField(
            "rules_ids",
            "Номера",
            lmp.ValueType.String,
            required=True,
            default_value=0,
            is_array=True
        ),
    )

    # источник сведений для данной методики
    # если унаследовать какую-то методику от данной методики
    # и необходимо поменять источник сведений
    # например, изменить SQL запрос, то менять нужно именно это свойство
    source = RailTSessionsSource

    # получение источника сведений для данной методики
    # используется для инкапсуляции при наследовании
    @classmethod
    def get_source(cls):
        return cls.source

    # получение данных из источника сведений для данной методики
    # используется для инкапсуляции при наследовании
    def get_source_data(self, rules_ids, start_date, end_date):
        source = self.get_source()
        return source().get_data(
            rules_ids=rules_ids,
            start_date=start_date,
            end_date=end_date
        )

    def execute(self, enter_params, result_writer, log_writer, *args, **kwargs):
        log_writer.info('START execute')

        # значения входных параметров
        rules_ids = ','.join(set(enter_params.rules_ids))
        start_date = enter_params.start_date
        end_date = enter_params.end_date

        # данные из источника сведений
        data = self.get_source_data(
            rules_ids=rules_ids,
            start_date=start_date,
            end_date=end_date
        )

        # составление отчета
        AsIsReport(
            result_writer,
            log_writer=log_writer
        ).make(
            from_data=data,
            to_header_format=RailTSessions
        )

        log_writer.info('END execute')

```

Чтоб использовать ее со своими "персональными" параметрами, создадим свою "онтологию" в дирекотрии с фреймворком "NiGHT GLOW"

```
mkdir ontology_muhosransk/constants/rail
mkdir ontology_muhosransk/sources/rail
mkdir ontology_muhosransk/tasks/rail
```

Файл параметров подключения `ontology_muhosransk/constants/rail/connections_params.py`:

```python
T_SESSIONS_CONNECTION_PARAMS = dict(
    driver='{SQL Server}',
    server='217.118.78.14\MUHOSRANSK',
    port='1433',
    database='rail',
    uid='user',
    pwd='123456'
)
```

Файл источника данных `ontology_muhosransk\sources\rail\rail_muhosransk_t_sessions_source.py`:

```python
from night_glow.sources.rail import RailTSessionsSource
from ontology_muhosransk.constants.rail import connections_params

class RailMuhosranskTSessionsSource(RailTSessionsSource):
    # Указываем, что подключение к источнику должно происходить
    # с нашими персональными параметрами подключения
    def __init__(self, *args, **kwargs):
        super().__init__(**connections_params.T_SESSIONS_CONNECTION_PARAMS)
```

Файл методики `ontology_muhosransk\sources\rail\rail_muhosransk_t_sessions_source.py`:

```python
import sys

# Все потому, что рабочая директория находится выше,
# так как определяется каталогом, где сам Lampyre.exe
sys.path.append('./user_tasks')
from night_glow.tasks import rail as rail_tasks
from ontology_muhosransk.sources.rail import RailTSessionsSource

class RailMuhosranskHTTPBilling(rail_tasks.RailHTTPBilling):
    # так как подключение к источнику должно происходить
    # с нашими персональными параметрами подключения
    # указываем наш настроенный усточник
    source = RailTSessionsSource

```

Подключаем в ПО "Лампир" файл `ontology_muhosransk\sources\rail\rail_muhosransk_t_sessions_source.py` с настроенной на свой источник методики и... И ЭТО ВСЁ!!!