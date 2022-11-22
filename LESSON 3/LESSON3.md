**1. Установить MongoDB в ВМ в GCP**
**2. Заполнить данными в какой-либо предметной области, например интернет-магазин**
**3. Написать несколько запросов на выборку, обновление и удаление данных**
**\* Создать индексы и сравнить производительность (в монге можно включить время выполнения запросов, ttl)**

MongoDB 6.0.3 установлена на виртуальной машине Ubuntu 22.04, аутентификация через логин/пароль. 

## Импорт данных
В коллекцию __example__ импортируем датасет [titanic](datasets/titanic.csv) c данными о пассажирах ([A Titanic Probability](https://web.stanford.edu/class/archive/cs/cs109/cs109.1166/problem12.html)). 
Импорт осуществляется через утилиту _mongoimport_.

```
mongoimport -h localhost --port 27017 --authenticationDatabase "admin" -u "admin" -p "admin" -d example -c titanic --type csv --headerline --ignoreBlanks --stopOnError  /media/sf_Downloads/VM\ data/titanic.csv
```
> 887 document(s) imported successfully. 0 document(s) failed to import.

Логинимся в _mongosh_ и проверяем коллекцию.
```
mongosh --port 27017 -u "admin" -p "admin" --authenticationDatabase "admin"
admin> use example
example> db.titanic.stats()
```
>{ns: 'example.titanic',
  __size: 159143,__
  __count: 887,__
  avgObjSize: 179,
  numOrphanDocs: 0,
  storageSize: 65536,
  freeStorageSize: 0,
  capped: false,
  }

## Выборка, вставка, обновление, удаление

Выбрать всех пассажиров 1 и 2 класса, до 5 лет включительно. Вывести возраст, пол, класс, статус выживания. Отсортировать по классу и возрасту (класс от 1 ко 2, возраст по убыванию)
```
db.titanic.find({Age: {'$lte': 5}, Pclass: {'$in': [1,2]}},{_id: 0, Pclass: 1, Age: 1, Survived: 1}).sort({Pclass: 1, Age: -1})
```
>[
  { Survived: 1, Pclass: 1, Age: 4 },
  { Survived: 0, Pclass: 1, Age: 2 },
  { Survived: 1, Pclass: 1, Age: 0.92 },
  { Survived: 1, Pclass: 2, Age: 5 },
  { Survived: 1, Pclass: 2, Age: 4 },
  { Survived: 1, Pclass: 2, Age: 4 },
  { Survived: 1, Pclass: 2, Age: 3 },
  { Survived: 1, Pclass: 2, Age: 3 },
  { Survived: 1, Pclass: 2, Age: 3 },
  { Survived: 1, Pclass: 2, Age: 2 },
  { Survived: 1, Pclass: 2, Age: 2 },
  { Survived: 1, Pclass: 2, Age: 1 },
  { Survived: 1, Pclass: 2, Age: 1 },
  { Survived: 1, Pclass: 2, Age: 0.83 },
  { Survived: 1, Pclass: 2, Age: 0.83 },
  { Survived: 1, Pclass: 2, Age: 0.67 }
]

Вывести 5 пассажиров: замужних женщин с именем Anna.
```
db.titanic.find({Name: {'$regex': 'Mrs.*Anna'}},{_id: 0, Name: 1}).limit(5)
```
>[
  { Name: 'Mrs. William (Anna Bernhardina Karlsson) Skoog' },
  { Name: 'Mrs. William (Anna) Hamalainen' },
  { Name: 'Mrs. William (Anna Sylfven) Lahtinen' },
  { Name: 'Mrs. Frank Manley (Anna Sophia Atkinson) Warren' },
  { Name: 'Mrs. Ernst Gilbert (Anna Sigrid Maria Brogren) Danbom' }
]

Вывести количество выживших/невыживших.
```
db.titanic.aggregate([{$group: {_id: "$Survived", count: {$sum: 1}}}])
```
>[ { _id: 0, count: 545 }, { _id: 1, count: 342 } ]

Добавляем 3 новые записи для экспериментов.
```
db.titanic.insertMany([
	{Survived: 1, Pclass: 1, Name: 'Mr. X', Sex: 'male', Age: 100, 'Siblings/Spouses Aboard': 0, 'Parents/Children Aboard': 0, Fare: 100.01},
	{Survived: 1, Pclass: 2, Name: 'Mr. Y', Sex: 'male', Age: 101, 'Siblings/Spouses Aboard': 0, 'Parents/Children Aboard': 0, Fare: 100.02},
	{Survived: 1, Pclass: 3, Name: 'Mrs. Z', Sex: 'female', Age: 102, 'Siblings/Spouses Aboard': 0, 'Parents/Children Aboard': 0, Fare: 100.03}
])
```

Для "Mr. X" добавить новое поле "Dog Aboard".
```
db.titanic.updateOne({Name: 'Mr. X'},{'$set': {'Dog Aboard': 1}})
```

Обновить стоимость проезда для пассажиров старше 100 лет (снизить на $10).
```
db.titanic.updateMany({Age: {'$gte': 100}},{'$inc': {'Fare': -10.}})
```

Выводим измененные строки:
```
db.titanic.find({Age: {'$gte': 100}})
```
>[  {
    _id: ObjectId("637caf7ac6e4e57bb5ddc360"),
    Survived: 1,
    Pclass: 1,
    Name: 'Mr. X',
    Sex: 'male',
    Age: 100,
    'Siblings/Spouses Aboard': 0,
    'Parents/Children Aboard': 0,
    __Fare: 90.01__,
    __'Dog Aboard': 1__
  },
  {
    _id: ObjectId("637caf7ac6e4e57bb5ddc361"),
    Survived: 1,
    Pclass: 2,
    Name: 'Mr. Y',
    Sex: 'male',
    Age: 101,
    'Siblings/Spouses Aboard': 0,
    'Parents/Children Aboard': 0,
    __Fare: 90.02__
  },
  {
    _id: ObjectId("637caf7ac6e4e57bb5ddc362"),
    Survived: 1,
    Pclass: 3,
    Name: 'Mrs. Z',
    Sex: 'female',
    Age: 102,
    'Siblings/Spouses Aboard': 0,
    'Parents/Children Aboard': 0,
    __Fare: 90.03__
  }]

Удаляем из коллекции поле 'Dog Abroad'.
```
db.titanic.updateMany({},{'$unset': {'Dog Aboard': ''}})
```

Удаляем свои строки.
```
db.titanic.deleteMany({Age: {'$gte': 100}})
```

## Индексы:
Загружаем коллекцию [stationary](datasets/stationary_data.zip) из 757357 строк (данные с датчиков Лондона по качеству воздуха [Breathe London AQMesh pods](https://data.london.gov.uk/dataset/breathe-london-aqmesh-pods)):
```
db.stationary.stats()
```
>{
  ns: 'example.stationary',
  size: 209198497,
  count: 757357,
  avgObjSize: 276,
  numOrphanDocs: 0,
  storageSize: 29360128,
  freeStorageSize: 0,
  capped: false
…
}

При импортировании по умолчанию создался индекс на поле _\_id_.
```
db.stationary.getIndexes()
```
>[ { v: 2, key: { _id: 1 }, name: '_id_' } ]
>
Проанализируем 2 запроса ДО и ПОСЛЕ создания индекса на строковое поле _utc_date_. 
```
db.stationary.createIndex({date_utc: 1})	
db.stationary.getIndexes()
```
>[
  { v: 2, key: { _id: 1 }, name: '_id_' },
  { v: 2, key: { date_utc: 1 }, name: 'date_utc_1' }
]

Анализ будем проводить по следующим показателям команды ``explain('executionStats')``:
__executionTimeMillis__: время в мс потраченное сервером на выбор оптимального плана и возврат результата (поскольку во всех тестируемых запросах ``rejectedPlans: []``, можем принимать это время за фактическое выполнение запроса)
__totalKeysExamined__: сколько раз был просмотрен индекс (если он использовался в плане)
__totalDocsExamined__: сколько раз пришлось совершить чтений документов с диска

- ищем все записи c датчиков на 2019-11-11 12:00:00
```
db.stationary.find({date_utc: '2019-11-11 12:00:00'}},{_id: 0, location_name: 1, date_utc: 1, pm2_5_ugm3: 1, ratification_status_pm2_5: 1, no2_ugm3: 1, ratification_status_no2: 1}).sort({no2_ugm3: 1}). explain('executionStats')
```
ДО индекса [план](plans/query1_BEFORE.json):
>nReturned: 89,
executionTimeMillis: __626__,
totalKeysExamined: 0,
totalDocsExamined: 757357,

ПОСЛЕ индекса [план](plans/query1_AFTER.json)
>nReturned: 89,
executionTimeMillis: __1__,
totalKeysExamined: 89,
totalDocsExamined: 89,

В целом отличный показатель, время выполнения кардинально уменьшилось за счет задействования в плане IXSCAN.

- ищем все записи на 10 утра (дата любая)
```
db.stationary.find({date_utc: {'$regex': '/*10:00:00*'}},{_id: 0, location_name: 1, date_utc: 1, pm2_5_ugm3: 1, ratification_status_pm2_5: 1, no2_ugm3: 1, ratification_status_no2: 1}).sort({no2_ugm3: 1}).explain('executionStats')
```
ДО индекса [план](plans/query2_BEFORE.json) :
>nReturned: 31518,
executionTimeMillis: __1236__,
totalKeysExamined: 0,
totalDocsExamined: 757357,

ПОСЛЕ индекса [план](plans/query2_AFTER.json):
>nReturned: 31518,
executionTimeMillis: __1503__,
totalKeysExamined: 757357,
totalDocsExamined: 31518,

Видимо, при поиске текста в середине строкового поля наличие индекса не улучшает ситуацию, быстрее просмотреть все записи через COLLSCAN, чем пользоваться IXSCAN.


