# edtech_complex_analysis
Анализ нескольких сфер деятельности онлайн образовательной платформы:
1. Анализ результатов A/B теста с новым методом оплаты
2. Подсчёт основных метрик по результатам A/B теста на данных из CH
3. Автоматизация тестирования на поступающих данных


## Анализ результатов A/B теста с новым методом оплаты

### Контекст и цель

В ходе тестирования одной гипотезы целевой группе была предложена новая механика оплаты услуг на сайте, у контрольной группы оставалась базовая механика.

### Стек

#### Python
import pandas as pd # Работа с таблицами
import numpy as np # Математика над таблицами
import pingouin as pg # Для статистических методов
import requests # Для загрузки данных
from urllib.parse import urlencode # Для загрузки данных
import seaborn as sns # Для визуализации
import matplotlib.pyplot as plt # Для файнтюнинга визуализации
from random import sample # Для симмуляции

### Этапы работы
Сначала были проанализированы данные на пример аномалий в виде несбалансированности выборок и пропусков. Пропусков обнаружено не было, несбалансированность не вносила вклад в дисперсию (поэтому не повредила t-тесту) и не была слишком выраженной (поэтому не повредила тесту хи-квадрат)

Затем встал вопрос, как учесть две поставки данных. Решено было отказаться от второй поставки: её невозможно анализировать отдельно из-за малого размера, а добавлять к первой порции посчитали не уместным, так как разный временной промежуток и неизвестны причины разрыва в подаче

Затем были выбраны метрики: целевой метрикой служила выручка, которая складывалась из конверсии и среднего чека

Конверсию проверили хи-квадратом и не обнаружили статистически значимых различий (анализ мощности показал, что мы должны были бы заметить даже маленький эффект)

Средний чек проверили с помощью t-теста и обнаружили значимые различия

Затем с помощьб бутстрапирования было симмулировано множественное повторение взятия одинаковых по размеру выборок для каждого механизма оплаты. Были построены распределения суммарной выручки. Был проведён t-тест с поправку на разницу дисперсий. Результаты показали эффективности внедрения нового способа оплаты.

### Результат
Статистический анализ и симмуляция показали обоснованность **выкатки в прод нового механизма оплаты** и даёт основания ожидать существенного увеличения выручки

## Автоматизация тестирования на поступающих данных

### Контекст и цель

В ходе тестирования одной гипотезы целевой группе была предложена новая механика оплаты услуг на сайте, у контрольной группы оставалась базовая механика. Нужно было составить первичное представление о результатах теста. Были подсчитаны метрики ARPU, ARPAU, и три вида CR (из посетителя в покупку, из активности в покупку, из активности по математике в покупку по математике)

### Стек

#### Python
import pandas as pd # Работа с таблицами
import numpy as np # Математика над таблицами
import pandahouse as ph # Для подключения к серверу ClickHouse

### Этапы работы
Сначала определились с некоторыми свойствами конструктов, которыми оперируем:
- ARPU считается относительно всех пользователей, попавших в группы.
- Активным считается пользователь, за все время решивший больше 10 задач правильно в любых дисциплинах.
- Активным по математике считается пользователь, за все время решивший 2 или больше задач правильно по математике.

Затем были отдельно составлены подзапросы, которые считали отдельные метрики. Далее встал вопрос о порядке действий и балансе читаемости и оптимальности использования БД. Решено было остановиться на отдельных CTE для всех метрик и финальном джоине таблицу 2x2, что дало оптимальное сочетание блочной структуры кода и скорости выполнения операций

P.s. Числа для CH в любом случае очень маленькие

### Результат
Мы, на первый взгляд, видим улучшение по всем метрикам, однако требуедтся дополнительная статистическая проверка
перед тем, как можно было бы делать какие-то окончательные выводы

## Удобства разрботки

### Контекст и цель

Процесс подсчёта метрик может иметь инкриментальный характер: данные продолжают поступать. Для удобства процесса необходимо реализовать 2 функции, которые сведут процесс до двух кликов:

1) Функция, которая будет автоматически подгружать информацию из дополнительного файла groups_add.csv (заголовки могут отличаться) и на основании дополнительных параметров пересчитывать метрики.

2) Функцию, которая будет строить графики по получаемым метрикам.

### Стек

#### Python
import pandas as pd # Работа с таблицами
import numpy as np # Математика над таблицами
from random import sample # Для симмуляции
from random import seed # Для воспроизводимости
import requests # Для загрузки данных
from urllib.parse import urlencode # Для загрузки данных
import seaborn as sns # Для визуализации
import matplotlib.pyplot as plt # Для файнтюнинга визуализации


### Этапы работы
Сначала функции были спроективрованы:формат входа, формат вывода, логика внутренних преобразований.

`update_data()`-функция принимает на вход два набора данных (которые есть, которые нужно добавить) в виде ссылки или непосредственно датафрейма. Внутри функции данные обогащаются, стандартизируются, по ним считаются метрики. На выходе мы имеем словарь следующей структуры: {<metric_one>:{df:<dataframe>, a :<group_a_metric_value>, b :<group_b_metric_value>},...}. df - итоговый датафрейм, на котором посчитана метрика, чтобы можно было визуализировать, a и b - значения метрик в двух группах.

`draw()` - функция принмает на вход словарь вида экспорта прошлой функции, ничего не возвращает, но строит графики: средние и доверительный интервал + распредление для чеков и суммы прибыли в случае равенства групп; нормализованный stacked barchart для конверсии.
    
Затем функции были протестированы на данных блока с анализом A/B теста


### Результат
Были реализованы две функции, которые оптимизируют процесс тестирования в ситуации поступающих данных