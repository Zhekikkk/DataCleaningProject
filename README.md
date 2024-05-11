# ПРОРОЕКТ: Анализ  резюме из HeadHunter
<img src=https://lindeal.com/ld/images/2022/11/28/top-15-sajtov-dlya-poiska-raboty-v-rossii-headhunter.png width=500px height=30%>

   В данном проекте мы рассмотрим какое влияние оказывают различные факторы на уровень ожидаемой зарплаты Соисткателей. Для этого мы произведем некоторые преобразования с текущими признаками и проимзведем очитску данных для точного анализа данных.

## Преобразование данных
   1. Произведем преобразование в признаке "Образование и ВУЗ". Напишем функцию по добавлению новых категорий образования и добавим их в новый
   признак "Образование", с помощью метода .apply(). Старый признак удалим методом .drop().

```python
hh_data["Образование"]=hh_data["Образование и ВУЗ"].apply(get_education_level)
hh_data=hh_data.drop("Образование и ВУЗ", axis=1) 
```

   2. Произведем преобразование в признаке "Пол, возраст". Разделим данный признак на два: "Пол" и "Возраст" с помощью метода .apply() и функции lambda. Старый признак удалим методом .drop().

```python
hh_data["Пол"]=hh_data["Пол, возраст"].apply(lambda x: x.split()[0])
hh_data["Пол"]=hh_data["Пол"].apply(lambda x: "М" if x=="Мужчина" else "Ж")

hh_data["Возраст"]=hh_data["Пол, возраст"].apply(lambda x: int(x.split()[2]))

hh_data=hh_data.drop("Пол, возраст", axis=1)
```

    3. Преобразуем признак "Опыт работы". Напишем фукнцию по подсчету опыта работы в месяцах и добавим данные в новый признак "Опыт работы (месяц)".Старый признак удалим методом .drop()

```python
def get_experience_months(arg):
   if arg=="Не указано" or arg is np.nan:
      return None
   exp= arg.split(' ')[:7]
   month_lst=["месяц", "месяцев", "месяца"]
   year_lst=["год", "года", "лет"]
   year=0
   month=0
   for index, item in enumerate(exp):
      if item in month_lst:
         month=int(exp[index-1])
      if item in year_lst:
         year=int(exp[index-1])
   return int(year*12+month)
   
hh_data["Опыт работы (месяц)"]=hh_data["Опыт работы"].apply(get_experience_months)

hh_data=hh_data.drop("Опыт работы", axis=1)  
```

   4. Следующим преобразуем "Город, переезд, командировки". Разделим его на три новых: "Город", "Готовность к переезду" и "Готовность к командировкам". Для этого напишем 3  функции, добавим при помощи .apply() в новые признаки и удалим старый методом .drop().
```python
def get_city_category(arg):
    city= arg.split(' , ')[0]
    million_cities = ['Новосибирск', 'Екатеринбург', 'Нижний Новгород', 'Казань', 'Челябинск', 'Омск', 'Самара', 'Ростов-на-Дону', 
                  'Уфа', 'Красноярск', 'Пермь', 'Воронеж', 'Волгоград' ]
    if (city == "Москва") or (city == "Санкт-Петербург"):
        return city
    elif city in million_cities :
        return "город-миллионник" 
    else:
        return "другие"
    
hh_data["Город"]=hh_data["Город, переезд, командировки"].apply(get_city_category)

def get_ready_removal(arg):
    if ('не готов к переезду' in arg) or ('не готова к переезду' in arg):
        return False
    elif 'хочу' in arg:
        return True
    else:
        return True

hh_data["Готовность к переезду"]=hh_data["Город, переезд, командировки"].apply(get_ready_removal)

def get_ready_business_trips(arg):
    if ('командировка' in arg):
        if ('не готов к командировкам' in arg) or('не готова к командировкам' in arg):
            return False
        else: 
            return True
    else:
        return False

hh_data["Готовность к командировкам"]=hh_data["Город, переезд, командировки"].apply(get_ready_business_trips)

hh_data=hh_data.drop("Город, переезд, командировки", axis=1)  
```

   5. Признаки "График" и "Занятость" удаляем. Добавляем новые признаки по названиям вариантов графиков работы и вариатов занятости.

```python
employment_lst=["полная занятость", "частичная занятость", "проектная работа", "стажировка", "волонтерство"]

for employment in employment_lst:
   hh_data[employment]=hh_data["Занятость"].apply(lambda x: employment in x if x else False)
   
chart_lst=["полный день", "сменный график", "гибкий график", "удаленная работа", "вахтовый метод"]
for chart in chart_lst:
   hh_data[chart]=hh_data["График"].apply(lambda x: chart in x if x else False)

hh_data=hh_data.drop(["График", "Занятость"], axis=1)    
```

   6. В признаке "ЗП" необходимо перевести валюты, в которых указаны ожидаемые зарплаты, в рубли. Для этого нужно использовать таблицу с актуальными данными по курсу валют и их значений в ISO. 
      1. Приведем значения даты в данных по Курсам валют и "Обновление резюме" к единому варианту при помощи функции pd.to_datetime()
      2. Пишем функцию по приведению всех валют к стандарту ISO
      3. Добавим получившийся результат в новый признак "Курс"
      4. Произведем присоединение таблицы с Курсом валют методом merge, указав способ и ключевые столбцы
      5. Значение в новой таблице для столбцов close и proportion заполняем 1, с помощью метода Fillna()

```python
exchange_rate = pd.read_csv('/Users/zekapavlova/IDE/Skillfactory/Project Head Hunter/Data/ExchangeRates.csv')
hh_data['Обновление резюме'] = pd.to_datetime(hh_data['Обновление резюме'], dayfirst=True).dt.date
exchange_rate['date'] = pd.to_datetime(exchange_rate['date'], dayfirst=True).dt.date

exchange_rate.drop(columns=['per', 'time', 'vol'], axis=1, inplace=True)

def get_iso_cur(arg):
#Создадим словарь, в котором будем хранить название валюты в данных и ее название в ISO
   currency_iso={ "грн" : "UAH", "USD" : "USD", "EUR" : "EUR", "белруб" : "BYN", 
                 "KGS" : "KGS", "сум" : "UZS", "AZN" : "AZN", "KZT" : "KZT"}
   curren=arg.split(" ")[1].replace('.', '')
   if curren == "руб":
      return "RUB"
   else:
      return currency_iso[curren]

hh_data["Курс"]=hh_data["ЗП"].apply(get_iso_cur)  

def get_salary(arg):
#Разделим строку, возьмем первый элементи приведем значения к типу float
   salary=float(arg.split(" ")[0])
   return salary

hh_data["Зарплата"]=hh_data["ЗП"].apply(get_salary)

merge_df=hh_data.merge(exchange_rate,                 
                       left_on=["Обновление резюме", "Курс"],
                       right_on=["currency", "date"],
                       how="left")

merge_df["close"]=merge_df["close"].fillna(1)
merge_df["proportion"]=merge_df["proportion"].fillna(1)

#Удаляем признаки ЗП, Курс, Зарплата
hh_data=hh_data.drop(["ЗП", "Зарплата", "Курс"], axis=1)   
```

### Исследование зависимостей в данных
   1. Для отслеживания распределения признака "Возраст", построим гистограмму через plotly express 
      ![](/Users/zekapavlova/IDE/Skillfactory/Project Head Hunter/Images/hist-age.png)

   2. Рассмотрим распределение признака "Опыт работы (месяц)", построим гистограмму через plotly express 
      ![](/Users/zekapavlova/IDE/Skillfactory/Project Head Hunter/Images/hist-experience.png)

   3. Рассмотрим распределение признака "ЗП (руб)", построим гистограмму через plotly express 
      ![](/Users/zekapavlova/IDE/Skillfactory/Project Head Hunter/Images/hist-salary.png)

   4. Далее рассмотрим зависимость Зарплаты от уровня Образования. Для этого построим столбчатую диаграмму через plotly express
      ![](/Users/zekapavlova/IDE/Skillfactory/Project Head Hunter/Images/bar-salary-education.png)
    
   5. Зависимость уровня Зарплаты от Города, построим на коробчатой диаграмме через plotly express
       ![](/Users/zekapavlova/IDE/Skillfactory/Project Head Hunter/Images/box-salary-city.png)

   6. Изучим зависимость уровня Зарплаты от Готовности к командировкам/переезду, построив столбчатую диаграмму через plotly express
      ![](/Users/zekapavlova/IDE/Skillfactory/Project Head Hunter/Images/bar-ready-remove.png)
    
   7. Проанализируем зависимость уровня Зарплаты от Возраста и уровня Образования, через тепловую карту в plotly express
      ![](/Users/zekapavlova/IDE/Skillfactory/Project Head Hunter/Images/imshow-age-salary-educ.png)

   8. На диграмме рассеивания рассмотрим зависимость Опыта работы (месяц) от Возраста.
      ![](/Users/zekapavlova/IDE/Skillfactory/Project Head Hunter/Images/scatter-age-experience.png)
   
### Очистка данных

#### Обнаружение и ликвидация дубликатов
   1. Чтобы отследить дубликаты используем метод .duplicated()
   Алгоритм:
      1. Применяем данный метод к копии исходной таблице
      2. Метод возвращает булевую маску значений True (если есть совпадения) или False.
   2. Далее необходимо избавиться от дубликатов. Применяем метод .drop_duplicates(). Он создает новыую таблицу, которая будет весрсией исходной, но очищенной от дубликатов.

#### Работа с пропусками в данных
   1. Проверим данные на наличе пропуском при помощи метода .isnull().
   Алгоритм:
      1. Применяем данный метод к копии исходной таблице
      2. Метод возвращает новый датафрейм, в ячейках которого значения True (если есть пропуск-NaN) или False.
   2. Удаляем столбец, если число пропусков в нем 30-40%, в остальных случаях удаляем строки. Используем метод .dropna(), передавая в него параметры для правильного удаления(axis - строка или столбец, how - во всех или в одном столбце есть пропуски, thresh - минимальное значение не пустых значений в строке/столбце).

#### Ликвидация выбросов
   Одним из этапов очистки данных является поиск выбросов. Мы используем метод трех сигм. Правило трех сигм гласит: что, если распределение данных является нормальным, то 99.73% лежат в интервале: $(\mu-3 \sigma$ , $\mu+3 \sigma)$, 
где  
* $\mu$ - математическое ожидание (для выборки это среднее значение)
* $\sigma$ - стандартное отклонение. 
Наблюдения, которые лежат за пределами этого интервала будут считаться выбросами.
   
   Алгоритм:
    1. Вычислить среднее и стандартное отклонение $\mu$ и $\sigma$ для признака, который мы исследуем
    2. Определить верхнюю и нижнюю границы:
* $bound_{upper} = \mu - 3 * \sigma$
    
* $bound_{lower} = \mu + 3 * \sigma$
    3. Найти наблюдения, которые выходят за пределы границ
    4. Можно попробовать воспользоваться методами преобразования данных, например, логарифмированием, чтобы попытаться свести распределение к нормальному или хотя бы к симметричному. 
    5. Также можно добавить вариативности количеству стандартных отклонений в левую и правую сторону распределений.
    6. Построим график, который покажет нормальное ли распредеение или есть ассиметрия.

```python

from outliers_lib.find_outliers import find_outliers_z_score

def outliers_z_score(data, feature, log_scale=False, left=3, right=4):
    if log_scale:
        x = np.log(data[feature]+1)
    else:
        x = data[feature]
    mu = x.mean()
    sigma = x.std()
    lower_bound = mu - left * sigma
    upper_bound = mu + right * sigma
    outliers = data[(x < lower_bound) | (x > upper_bound)]
    cleaned = data[(x >= lower_bound) & (x <= upper_bound)]
    return outliers, cleaned
```

```python

fig_9, ax = plt.subplots(1, 1, figsize=(8, 4))
log_age= np.log(drop_hh_data['Возраст'] + 1)
histplot = sns.histplot(log_age, bins=30, ax=ax)
histplot.axvline(log_age.mean(), color='k', lw=2)
histplot.axvline(log_age.mean()+ 4 * log_age.std(), color='k', ls='--', lw=2)
histplot.axvline(log_age.mean()- 3 * log_age.std(), color='k', ls='--', lw=2)
histplot.set_title('Log age')
```
![](/Users/zekapavlova/IDE/Skillfactory/Project Head Hunter/Images/log-age.png)

## Использованные инструменты и библиотеки
* numpy (1.26.2)
* pandas (2.1.4)
* matplotlib (3.8.2)
* seaborn (0.13.2)
* plotly (5.18.0)