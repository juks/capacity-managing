<img src="https://github.com/juks/capacity-planning/assets/147685/bf0073db-86f9-4041-8102-1044e9e0478d" width="900">

### Идея

Упреждающее планирование аппаратных ресурсов (Capacity Planning) для больших проектов — довольно непростая задача. «Прикладывание линейки» или ручное вычисление зависимости между основными метриками проекта и количеством используемых ресурсов не всегда дают достаточно точный результат.

В этой статье я проведу эксперимент. Используя регрессионную модель машинного обучения, я попробую «предсказать» количество ресурсов, необходимых проекту для работы на уровне, определяемом ключевыми метриками. Решение будет основано на апрекрасной библиотеке PyCaret, сверх-доступном инструменте, созданном для задач машинного обучения на основе временных рядов. В качестве метрик данных будут использованы данные условного проекта в продакшене.

Набор данных представляет собой CSV-файлы, содержащий следующие метрики:
* Временная метка.
* Количество запросов для отдельных методов.
* Условная бизнес-метрика (Orders).
* Количество виртуальных ядер, используемых проектом для обработки текущей нагрузки (CPU).

### Важно
Лучший способ определения реальной ёмкости сервиса, так чтобы это было безопасно, управляемо и обратимо — использовать систему автоматического контроля ёмкости (нагрузочного тестирования). К сожалению, не все владельцы сервисов находят время и возможности на организацию такого процесса, поэтому временами делают заказ на оборудование, основываясь на умозрительных гипотезах. Предлагаемый способ прогноза разворачивается и выполняется в буквальном смысле за пятнадцать минут. При всей своей простоте он даёт существенно более точную альтернативу методу «пальцем в небо на миллионы рублей», хоть и не учитывает в полной мере внутреннюю специфику сложной архитектуры, в которой ресурсы, своевременно добавленные в одном месте, на раннем рубеже следования нагрузки, могут вызвать несвоевременный крах в другом, просто обеспечив дальнейшее её прохождение.

### Пошаговый план
Всё намного проще, чем можно представить. Задача решается в три шага.

1. Установка PyCaret и Jupyter Notebook

С Jupyter все знакомы, все работают. Процесс установки PyCaret хорошо описан в <a href="https://pycaret.gitbook.io/docs/">официальной документации</a>. Лично я без проблем установил библиотеку в среде самого блокнота, то есть выполнял pip install в интерфейсе Jupyter Notebook.

2. Сбор данных

Следует выбрать проект для изучения, затем произвести выгрузку ключевых метрик, необходимых для обучения модели и построения прогноза. В этом примере используется некий условно-абстрактый проект, для которого есть набор общепринятых метрик: RPS отдельных методов, основная бизнес-метрика (количество заказов), число ядер, утилизируемых проектов в конкретный момент вркемени. Попробуем построить регрессию для этого набора данных и понять, чем она может быть полезна в деле аргументированного планирования ресурсов, необходимых для роста проекта.

3. Обучение модели

Запускаем Jupiter Notebook. Составляем первый рецепт. Задача: загрузить исходный набор данных, добавить в него дополнительные календарные свойства, построить по одной регрессии из доступного набора и выбирать ту из них, у которой будет минимальной величина MAE (Mean Absolute Error).

Для простоты примера мы не используем дополнительные факторы, влияющие на прогнозируемую метрику (количество ядер), как было сказано выше, наш прогноз будет основан на суммарной величине всех RPS и количестве заказов, обрабатываемых условным сервисом.

Исходный файл содержит данные с мая по неполный ноябрь. Мы исключаем весь ноябрь из обучающего набора, чтобы проверить то, как точно модель сможет его предсказать. Цель обучения (предсказания) — количество утилизируемых ядер (target = 'Cpu'). Получаемая величина по-умолчанию называется Label.


```python
import pandas as pd
from pycaret.regression import *

train_until = '2022-12-01'

data = pd.read_csv('consumption.csv', usecols=['Orders', 'Timestamp', 'Cpu'])
data['Date'] = pd.to_datetime(data['Timestamp'] + 3600 * 3, unit='s')
data['ValueIndex'] = [i for i in range(0, len(data['Date']))]

train = data[(data['Date'] < train_until)]
test = data

s = setup(
    data = train,
    test_data = test,
    target = 'Cpu',
    fold_strategy = 'timeseries',
    fold = 3,
    transform_target = False,
    session_id = 123
)

best = compare_models(sort = 'MAE')

predictions = predict_model(best, data=data)
```
![capacity1](https://github.com/juks/capacity-managing/assets/147685/0ec4fd3d-68ba-4b8d-8384-f6b724570823)

С небольшим отрывом победу одержала Orthogonal Matching Pursuit. Последняя строка кода выполняет предсказание искомой величины. Результат сохраняется в поле Label.

Вычислим два поля: RPS (суммарная величина количества запросов в секунду) и Delta (Разница между прогнозным значением Cpu и фактическим), затем попробуем визуализировать три набора данных:
* Исходные значения метрик на всем временном интервале.
* Факт и предсказание на ноябрь.
* Факт и предсказание за ноябрь в соотношении только с величиной суммарного RPS.


```python
import plotly.express as px
import numpy as np

# На реальных данных, использованных для обучения
predictions['Date'] = pd.to_datetime(predictions['Timestamp'] + 3600 * 3, unit='s')

fld = ['Cpu', 'Label', 'Orders']
fig = px.line(predictions[predictions['Date'] < train_until],
              x='Date', y=fld, template = 'plotly_white')
fig.show()
```
![capacity2](https://github.com/juks/capacity-managing/assets/147685/46cb3f9f-3ff7-4135-9cc1-c8b27fd9638e)

Запускаем, смотрим.

Общий график. Ничего интересного. Но нам известен заведомо важный факт: в середине декабря сервис работал с повышенной нагрузкой. Модель не обладает знаниями о том, как сервис ведет себя в таких условиях. Рассмотрим один месяц графиков поближе.


```python
# На реальных данных, не использованных для обучения
data_rps = pd.read_csv('consumption.csv')
data_rps['Date'] = pd.to_datetime(data_rps['Timestamp'] + 3600 * 3, unit='s')

predictions['Rps'] = data_rps[list(data_rps.keys())[3:]].sum(axis=1)
predictions['Delta'] = predictions['Cpu'] - predictions['Label']

fig = px.line(predictions[predictions['Date'] >=  train_until],
              x='Date', y=['Cpu', 'Label', 'Orders', 'Delta', 'Rps'], template = 'plotly_white')
fig.update_layout(barmode='group', hovermode='x')
fig.show()
```

![capacity3](https://github.com/juks/capacity-managing/assets/147685/ad3248e4-d3b0-48e6-b933-4fa725244135)

Фактическое и прогнозное число потребляемых ядер идут ноздря в ноздрю, этого говорит в пользу точности прогноза. Однако 9-го декабря значения расходятся: фактическое количество утилизируемых ядер становится ниже прогнозируемого. Отмасштабируем график ещё ближе.


```python
fig = px.line(predictions[(predictions['Date'] >= '2022-12-09 06:00:00') & (predictions['Date'] <= '2022-12-09 18:00:00')],
              x='Date', y=['Cpu', 'Label'], template = 'plotly_white')
fig.update_layout(barmode='group', hovermode='x')
fig.show()
```
![capacity4](https://github.com/juks/capacity-managing/assets/147685/e798cbe8-bd17-4b76-9530-7b331d73e80f)

Разница существенная: прогноз отличается от факта примерно на 60% (860 ядер прогноза против фактических 510). В чём же причина? Причина в успешной оптимизации проекта, применённой именно в это время.

Вывод: кроме прогноза количества используемых ресурсов, модель можно использовать для пассивного поиска деградаций или, как в этом случае, оптимизаций!

Но вернёмся к основной идее. Нас интересует прогноз количества ядер, необходимых сервису для роста. Определим рост предельно просто: обработка нагрузки, скажем, соответствующей тысяче заказов за пять минут, то есть примерно трёх заказов в секунду.

Чтобы визуализировать зависимость количества необходимый ядер от количества заказов, представим её в двух простых синтетических видах: линейный рост и синусоидальные изменения. Заменим параметр Orders соответствующими последовательностями значений и построим на полученные графики.

```python
# Линейная синтетика
data_new = data.copy()
data_new['Orders'] = [i / 50 for i in np.arange(0, len(data), 1)]

predictions_new = predict_model(best, data=data_new, verbose=False)
predictions_new['Date'] = pd.to_datetime(predictions_new['Timestamp']+ 3600 * 3, unit='s')

fld = ['Label', 'Orders']
fig = px.line(predictions_new,
              x='Date', y=fld, template = 'plotly_white')


fig.add_shape(
    type="line", line_color="grey", line_width=3, opacity=1, line_dash="dot",
    x0=0, x1=1, xref="paper", y0=1000, y1=1000, yref="y"
)

fig.update_layout(barmode='group', hovermode='x')
fig.show()
```
![capacity5](https://github.com/juks/capacity-managing/assets/147685/1e326212-a967-4d24-bf85-0dd47841beb7)

Итак, для обработки трех заказов в секунду в идеальных условиях проекту потребуется примерно 370 ядер. Значение соответствует прогнозу модели, простроенной на основе данных о потреблении проектом ресурсов с мая по конец октября.

Для более красивого завершения повествования, приведу результат, возвращаемый моделью при поступлении синусоидальных данных на вход.

```python
# Синусоидальная синтетика
data_new = data.copy()
data_new['Orders'] = [abs(int(np.sin(i/50) * 1000)) + i / 10 - 5000 for i in np.arange(0, len(data), 1)]

predictions_new = predict_model(best, data=data_new, verbose=False)
predictions_new['Date'] = pd.to_datetime(predictions_new['Timestamp'] + 3600 * 3, unit='s')

fig = px.line(predictions_new[predictions_new['Date'] > train_until],
              x='Date',y=fld, template = 'plotly_white')

fig.show()
```

![capacity6](https://github.com/juks/capacity-managing/assets/147685/18b2ad9a-c1d6-49f8-bf57-cf5016efb082)

### Заключение
Результат можно обобщить следующим образом:
* Возможность построения статистической регрессии с помощью машинного обучения предельно проста и доступна.
* Полученная модель предоставляет достаточно точный прогноз зависимости целевой величины от исходных метрик.
* Результат прогноза можно использовать для планирования количества аппаратных ресурсов, необходимых для работы сервисов в условиях растущей нагрузки. Этот прогноз, хоть и существует в идеальных условиях, обладает значительно большей точностью, нежели умозрительное допущение.
