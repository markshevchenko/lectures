# Принцип инверсии зависимостей

Разрабатываем веб-приложение, которое умеет:

1. Строить маршрут между двумя точками.
2. Расчитывать стоимость поездки по маршруту.

## Google Maps Directions API

Мы будем обращаться к внешним сервисам для того, чтобы строить маршрут. Сначала попробуем Google Maps Directions API.

Напишем код на JavaScript, чтобы его можно было запускать из браузера. К сожалению, Google Maps Directions API
не поддерживает CORS и не позволяет обращаться к сервису с произвольных страниц. Чтобы обойти это ограничение, мы задействуем
сервис [CORS anywhere](http://cors-anywhere.herokuapp.com/).

Для тестирования кода вы должны зарегистрироваться в Google как разработчик, и сгенерировать ключ в [Диспетчере API](https://console.developers.google.com/apis/library).

Чтобы построить маршрут, мы будем посылать GET-запрос вида

```
http://cors-anywhere.herokuapp.com/maps.googleapis.com:443/maps/api/directions/json?origin={from}&destination={to}&key={key}
```

Здесь `from`&nbsp;&mdash; адрес отправления, `to`&nbsp;&mdash; адрес назначения, и `key`&nbsp;&mdash; ключ, полученный в Диспетчере API.
Сервис возвращает JSON-объект, в котором нас интересуют только часть полей, а именно:

```javascript
{
  routes: [
    {
      legs: [
        {
          steps: [
            {
              distance: {
                value: 123,
              },
              duration: {
                value: 32
              }
            },
            . . .
          ]
        },
        . . .
      ]
    },
    . . .
  ]
}
```

Directions API возвращает массив объектов `routes`, поскольку мы можем попросить несколько маршрутов на выбор. Но мы не просим, поэтому этот массив всегда содержит только один элемент.
Маршрут состоит из массива участков `legs`, в котором тоже только один элемент, потому что мы не задавали промежуточных точек. Наконец, участок содержит массив шагов `steps`.
Для каждого шага мы можем узнать длину в метрах (`distance.value`) и продолжительность в секундах (`duration.value`).

Посчитаем длину маршрута и его продолжительность:

```javascript
var meters = response.routes[0]
                     .legs
                     .reduce((steps, leg) => steps.concat(leg.steps), [])
                     .maps(step => step.distance.value)
                     .reduce((totalMeters, meters) => totalMeters + meters, 0);

var seconds = response.routes[0]
                      .legs
                      .reduce((steps, leg) => steps.concat(leg.steps), [])
                      .maps(step => step.duration.value)
                      .reduce((totalSeconds, seconds) => totalSeconds + seconds, 0);

```

Если расчёт цены маршрута ведётся на основании стоимости километра, то используем формулу:

```javascript
var price = kilometerPrice * meters/1000.0;
```

Код целиком будет выглядеть так:

```html
<label><span>Откуда</span> <input id="from" type="text" size="80" value="" /></label>
<label><span>Куда</span> <input id="to" type="text" size="80" value="" /></label>
<label><span>Стоимость километра</span> <input id="kilometer-cost" type="text" size="80" value="15" /></label>
<label><span>Maps Key</span> <input id="key" type="text" size="80" value="" /></label>
<button id="calculate-price">Рассчитать стоимость</button>
<label><span>Стоимость поездки</span> <input id="price" type="text" readonly size="80" /></label>
<script>
document.getElementById('calculate-price')
        .addEventListener('click', function() {
          var from = document.getElementById('from')
                             .value;
          var to = document.getElementById('to')
                           .value;
          var key = document.getElementById('key')
                            .value;

          var uri = 'http://cors-anywhere.herokuapp.com/maps.googleapis.com:443/maps/api/directions/json'
                  + '?origin=' + from
                  + '&destination=' + to
                  + '&key=' + key;
          
          var xhr = new XMLHttpRequest();
          xhr.onload = function() {
            var response = JSON.parse(xhr.responseText);

            if (response.status != 'OK')
              throw new Error('Invalid Google Maps Response status ' + response.status, ', message: ' + response.error_message);

            var meters = response.routes[0]
                                 .legs
                                 .reduce((steps, leg) => steps.concat(leg.steps), [])
                                 .maps(step => step.distance.value)
                                 .reduce((totalMeters, meters) => totalMeters + meters, 0);

            var kilometerCost = parseFloat(document.getElementById('kilometer-cost').value);

            document.getElementById('price').value = kilometerCost * meters/1000.0;
          };

          xhr.onerror = function() {
            throw new Error(xhr.statusText);
          };

          xhr.open('GET', uri, true);
          xhr.send();
        }, false);
</script>
```

Код получился простым и понятным.

## Многоуровневая архитектура

К нам приходит руководство, и говорит, что теперь нам надо строить маршруты не только на клиенте, но и на сервере&nbsp;&mdash; в **Node.js**.

Наш код не готов к быстрой реализации этого требования. На практике код должен быть не только простым для понимания, но и простым для изменения.

Что затрудняет изменения? Изменения затрудняют связи между разными участками кода&nbsp;&mdash; мы меняем код в одном месте, и это заставляет нас менять код в другом.

Задача заключается в том, чтобы ограничивать связи между разными участками кода.

Первое, что мы сделаем&nbsp;&mdash; разобьём код на уровни. В больших проектах, на уровни разбивают модули, но наш проект учебный, поэтому мы будем разносить отдельные функции.
Классический подход заключет в том, чтобы разбить приложение на три уровня: *интерфейс*, *предметную область* и *доступ к данным*.

Эрик Эванс, создатель DDD, называет уровень доступа к данным *инфраструктурным*. Когда-то его назначением была только работа с СУБД, но сейчас там реализуют взаимодействие
с разными сервисами, например, с построителями маршрутов. Уровень интерфейса часто называют уровнем *представления*, а уровень предметной области&nbsp;&mdash;уровнем *бизнес-логики*.

Эти названия полезно знать, чтобы общаться с другими разработчиками.

Уровни принято рисовать сверху-вниз в таком порядке:

```
┌──────────────────────────────────┐
│                                  │
│      Уровень представления       │
│                                  │
└──────────────────────────────────┘
                ↓↓↓
┌──────────────────────────────────┐
│                                  │
│    Уровень предметной области    │
│                                  │
└──────────────────────────────────┘
                ↓↓↓
┌──────────────────────────────────┐
│                                  │
│     Инфраструктурный уровень     │
│                                  │
└──────────────────────────────────┘
```

Связь между кодом на разных уровнях упорядочена так: код на инфраструктурном уровне знает только про себя. Он не может ссылаться на объекты и функции верхних уровней.
Код уровня предметной области знает про себя и про инфраструктурный уровень. Уровень представления знает про всё.

Перепишем код так, что разбить его на уровни. Начнём снизу. На инфраструктурном уровне у нас находится код построения маршрута. Вынесем его в отдельный метод. Из-за того,
что код выполняет асинхронно, выглядеть он будет немного странно:

```javascript
function buildRoute(from, to, key, callback) {
  var uri = 'http://cors-anywhere.herokuapp.com/maps.googleapis.com:443/maps/api/directions/json'
          + '?origin=' + from
          + '&destination=' + to
          + '&key=' + key;
  
  var xhr = new XMLHttpRequest();
  xhr.onload = function() {
    var response = JSON.parse(xhr.responseText);

    if (response.status != 'OK')
      throw new Error('Invalid Google Maps Response status ' + response.status, ', message: ' + response.error_message);

    callback(response.routes[0]);
  };

  xhr.onerror = function() {
    throw new Error(xhr.statusText);
  };

  xhr.open('GET', uri, true);
  xhr.send();
}
```

Функция `buildRoute` строит маршруты. Она ничего не знает про поля ввода `from`, `to`, и `key`, поэтому её можно вызывать и на сервере. Она также ничего не знает про расчёт цены,
поскольку это знание относится к предметной области. Функция запрашивает маршрут у Google Maps Directions API и передаёт его в функцию `callback`.

Получив маршрут на уровне предметной области, мы складываем длину всех участков и вычисляем общую продолжительность маршрута. Умножив её на стоимость километра, получаем общую стоимость
поездки:

```javascript
function calculatePrice(route, kilometerCost) {
  var meters = route.legs
                    .reduce((steps, leg) => steps.concat(leg.steps), [])
                    .maps(step => step.distance.value)
                    .reduce((totalMeters, meters) => totalMeters + meters, 0);

  return kilometerCost * meters/1000.0;
}
```

Кажется, что эта функция не знает про инфраструктурный уровень, но это не так. Функция знает структуру `route`, которая описана в Google Maps Directions API. Если
компания Google решит внести изменения в свой код, нам также придётся вносить изменения в свой. Пока запомним этот факт.

Зависимость с верхним уровнем представления у нас разорвана: мы больше не знаем про поле ввода `price` и не меняем его значение, как раньше.
Теперь расчёт цены также можно вызывать на сервере.

Завершаем рефакторинг программы на уровне представления:

```javascript
document.getElementById('calculate-price')
        .addEventListener('click', function() {
          var from = document.getElementById('from')
                             .value;
          var to = document.getElementById('to')
                           .value;
          var key = document.getElementById('key')
                            .value;

          buildRoute(from, to, key, function(route) {
            var kilometerCost = parseFloat(document.getElementById('kilometer-cost').value);

            var price = calculatePrice(route, kilometerCost);

            document.getElementById('price').value = price;
          });
        }, false);
```

Здесь мы извлекаем значения из полей `from`, `to`, `key` и `kilometer-cost`, и меняем значение поля `price`. Мы вызываем функции нижних уровней `buildRoute` и `calculatePrice`.
Эти функции мы можем вызывать как в бразуере, так и на сервере, потому что они больше не обращаются к методу `getElementById` объекта `document`.

Удивительно, но рефакторинг кода сделал его понятнее, поскольку нам пришлось вычленить небольшие функции, которым мы дали читаемые названия `buildRoute` и `calculatePrice`.

## Bing Maps API и принцип Инверсии зависимостей

Снова к нам приходит руководство и говорит, что с компанией Google не удалось договориться. Нам придётся использовать сервис компании Microsoft&nbsp;&mdash; Bing Maps API.
Интерфейс метода HTTP GET изменится не очень сильно, теперь он будет выглядеть так:

```
http://cors-anywhere.herokuapp.com/http://dev.virtualearth.net/REST/v1/Routes?wp.1={from}&wp.2={to}&key={key}
```

А вот возвращаемый JSON отличается очень сильно:

```javascript
{
  resourceSets: [
    {
      resources: [
        {
          routeLegs: [
            {
              itineraryItems: [
                {
                  travelDistance: 0.283,
                  travelDuration: 37,
                },
                {
                  travelDistance: 0.311,
                  travelDuration: 47,
                },
                {
                  travelDistance: 0.150,
                  travelDuration: 12,
                },
                . . .
              ]
            },
            . . .
          ]
        },
        . . .
      ]
    },
    . . .
  ]
}
```

Время прохождения шага `travelDuration` возвращается в секундах, как и в Google Maps API, а вот расстояние&nbsp;&mdash; в километрах. Из-за того, что функкция предметной области `calculatePrice`
зависит от того, как хранится маршрут, мы вынуждены теперь переписать и её. Однако, вот проблема: если нам нужно оставить возможность вернуться к Google Maps API, мы должны будем
написать две версии `calculatePrice`&nbsp;&mdash; для Bing, и для Google.

Можем ли мы убрать эту зависимость совсем? Нет. Но мы можем изменить её направление:

```
┌──────────────────────────────────┐
│                                  │
│      Уровень представления       │
│                                  │
└──────────────────────────────────┘
                ↓↓↓
┌──────────────────────────────────┐
│                                  │
│    Уровень предметной области    │
│                                  │
└──────────────────────────────────┘
                ↑↑↑
┌──────────────────────────────────┐
│                                  │
│     Инфраструктурный уровень     │
│                                  │
└──────────────────────────────────┘
```

Как видите, теперь инфраструктурный уровень зависит от уровня предметной области. Это и есть *инверсия зависимости*.

### Интерфейсы

Сначала давайте обсудим, как её реализовать, а затем&nbsp;&mdash; что она нам даёт. Первая трудность заключается в том, что для Google нам приходиться вызывать один метод построения маршрута,
а для Bing&nbsp;&mdash; другой. Очевидное решение в объектно-ориентированных языках&nbsp;&mdash; сделать метод частью полиморфного класса и изменять его реализацию в наследниках:

```javascript
class RouteBuilder {
  constructor() { }

  build(from, to, callback) { throw new Error("Method 'build' is not overriden."); }
}

class GoogleMapsRouteBuilder extends RouteBuilder {
  constructor(key) {
    super();

    this.key = key;
  }

  build(from, to, callback) {
    var uri = 'http://cors-anywhere.herokuapp.com/maps.googleapis.com:443/maps/api/directions/json'
            + '?origin=' + from
            + '&destination=' + to
            + '&key=' + this.key;
    
    var xhr = new XMLHttpRequest();
    xhr.onload = function() {
      var response = JSON.parse(xhr.responseText);

      if (response.status != 'OK')
        throw new Error('Invalid Google Maps Response status ' + response.status, ', message: ' + response.error_message);

      callback(response.routes[0]);
    };

    xhr.onerror = function() {
      throw new Error(xhr.statusText);
    };

    xhr.open('GET', uri, true);
    xhr.send();
  }
}

class BingMapsRouteBuilder extends RouteBuilder {
  constructor(key) {
    super();

    this.key = key;
  }

  build(from, to, callback) {
    var uri = 'http://cors-anywhere.herokuapp.com/http://dev.virtualearth.net/REST/v1/Routes'
            + '?wp.1=' + from
            + '&wp.2=' + to
            + '&key=' + this.key;
    
    var xhr = new XMLHttpRequest();
    xhr.onload = function() {
      var response = JSON.parse(xhr.responseText);

      if (response.statusDescription != 'OK')
        throw new Error('Invalid Bing Maps Response status ' + response.status, ', message: ' + response.errorDetails.join(' '));

      callback(response.resourceSets[0].resources[0]);
    };

    xhr.onerror = function() {
      throw new Error(xhr.statusText);
    };

    xhr.open('GET', uri, true);
    xhr.send();
  }
}
```

Теперь у нас есть абстрактный метод `RouteBuilder.build(from, to, callback)`, который мы реализуем в наследниках `GoogleMapsRouteBuilder` и `BingMapsRouteBuilder`.

### Объекты переноса данных

Вторая трудность заключается в том, что Google и Bing присылают маршрут каждый в своём формате. Мы должны зафиксировать не только интерфейс метода `build`,
но и структуру данных, которую ожидаем в методе `callback`.

Что нам нужно знать о маршруте в предметной области? Нам нужно знать, что он состоит из отдельных участков, и у каждого участка есть длина в километрах и продолжительность в минутах.
Мы расчитываем цену поездки на основании километров и минут, поэтому и хотим получать данные сразу в этих единицах измерения:

```javascript
{
  [
  {
    kilometers: 0.283,
    minutes: 0.371
  },
  . . .
  ]
}
```

Структура, которую мы создали, предназначена для переноса данных с инфраструктурного уровня на уровень предметной области.
*Объект переноса данных* (*Data Transfer Object*, *DTO*)&nbsp;&mdash; известный паттерн, описанный [Мартином Фаулером](http://www.ozon.ru/context/detail/id/1616782/).

Осталось только конвертировать данные из формата Google и Bing в наш формат:

```javascript
var route = response.routes[0]
                    .legs
                    .reduce((accumulator, leg) => accumulator.concat(leg.steps), [])
                    .map(step => ({
                      kilometers: step.distance.value/1000.0,
                      minutes: step.duration.value/60.0
                    }));

callback(route);

. . .

var route = response.resourceSets[0]
                    .resources[0]
                    .routeLegs
                    .reduce((accumulator, leg) => accumulator.concat(leg.itineraryItems), [])
                    .map(item => ({
                      kilometers: item.travelDistance,
                      minutes: item.travelDuration/60.0
                    }));

callback(route);
```
Теперь наш код расчёта стоимости выглядит так:
```javascript
function calculatePrice(route, kilometerCost) {
  var totalKilometers = route.maps(segment => segment.kilometers)
                             .reduce((totalKilometers, kilometers) => totalKilometers + kilometers, 0);

  return kilometerCost * totalKilometers;
}
```
Код построения маршрута:
```javascript
routeBuilder.build(from, to, key, function(route) {
  var kilometerCost = parseFloat(document.getElementById('kilometer-cost').value);

  var price = calculatePrice(route, kilometerCost);

  document.getElementById('price').value = price;
});
```
Здесь `routeBuilder`&nbsp;&mdash; переменная типа `RouteBuilder`, в которой находится экземпляр `GoogleMapsRouteBuilder` или `BingMapsRouteBuilder` в зависимости от наших нужд.

### Что даёт нам инверсия зависимостей?

В результате инверсии зависимостей код на уровне предметной области стал стабильным. Функцию `calculatePrice` не придётся
переписывать каждый раз, когда мы захотим изменить сервис построения маршрутов.

Теперь мы можем написать модульные тесты для всех функций, которые раньше обращались напрямую к инфраструктурному уровню,
потому что мы можем использовать тестовые реализации интерфейсов.

За это мы платим цену: во-первых, мы вводим дополнительный набор *интерфейсов*, а в во-вторых&nbsp;&mdash;описываем
*объекты переноса данных* и вынуждены их заполнять.

### Корень композиции

Маленькая, но существенная деталь заключается в том, что поскольку у нас разорвана зависимость, наша программа не будет работать.

Уровень представления теперь не знает о классах `GoogleMapsRouteBuilder` и `BingMapsRouteBuilder`, но знает о `RouteBuilder`. Проблема
в том, что он не может создать объект класса `RouteBuilder`, поскольку это абстрактный класс. Где-то в нашей программе
нужно создать конкретный экземпляр и присвоить его переменной `routeBuilder`.

Это &laquo;где-то&raquo;&nbsp;&mdash; *корень композиции*. Здесь мы создаём экземпляры конкретных классов и присваиваем их переменным.
Другое название этого процесса&nbsp;&mdash; *внедрение зависимости* (*dependency injection*).

```
routeBuilder = new GoogleMapsRouteBuilder(key);
```

### Фиксируем

Для того, чтобы безболезненно расширять инфраструктурный уровень, мы инвертируем зависимость между ним и уровнем предметной области. Сначала мы
описываем *интерфейсы* и *объекты переноса данных*, которые потребуются на уровне предметной области. Затем мы реализуем эти интерфейсы
и конвертируем данные в нужный формат на инфраструктурном уровне.

Инвертировав зависимости, мы сделали уровни независимыми, но теперь нам приходится *внедрять зависимости*. Место программы, где происходит внедрение,
мы называем *корнем композиции*.

## Паттерн Стратегия (Strategy)

И снова к нам приходит руководство. Что бы мы без них делали? На этот раз руководство хочет расширить способы расчёта цены. Речь идёт о четырёх разных способах:

* **По километрам**. Умножаем общую длину маршрута на стоимость километра. Этот способ мы уже реализовали.
* **По минутам**. Умножаем общую продолжительность маршрута на стоимость минуты.
* **По километрам и минутам**. Складываем стоимость по километрам и стоимость по минутам. Этот способ использует Яндекс.Такси. Из-за того, что цены Яндекса в полтора-два
  раза дешевле средних по рынку, общая цена получается небольшой. Это простой способ учитывать пробки.
* **По километрам, но в пробке по минутам**. Вычисляем скорость на каждом участке. Если она выше 6км/час, считаем стоимость по километрам, если ниже&nbsp;&mdash; по минутам.
  Это сложный способ учитывать пробки.

В объектно-ориентированном языке мы снова можем воспользоваться наследованием:

```javascript
class PriceStrategy {
  constructor() { }

  calculate(segments) { throw new Error("Method 'calculate' is not overriden."); }
}
```

Мы описываем интерфейс расчёта цены. Для расчёта нам требуется маршрут, то есть массив участков. Другую информацию, например, стоимость километра, мы передаём
в класс, реализующий конкретную стратегию расчёта.

```javascript
class KilometersStrategy extends PriceStrategy
{
  constructor(tariff) {
    super();

    this.tariff = tariff;
  }

  calculate(segments) {
    var kilometers = segments.reduce((totalKilometers, segment) => totalKilometers + segment.kilometers, 0)

    return this.tariff.kilometer * kilometers;
  }
}
```

Паттерн *Стратегия* позволяет разорвать зависимость с алгоритмом решения задачи, если существует целое семейство алгоритмов, и клиент может выбирать один или другой.

## Паттерн Заместитель (Proxy)

Ещё один полезный паттерн&nbsp;&mdash;*Заместитель* (*Proxy*). Взглянем на реализацию расчёта цены по километрам из предыдущего примера. В конструктор мы передаем **тариф**, в котором содержится стоимость километра и стоимость минуты:

```javascript
class Tariff {
  constructor() { }

  get kilometer() { throw new Error("Property 'kilometer' is not overriden."); }

  get minute() { throw new Error("Property 'minute' is not overriden."); }
}
```

Мы не знаем, где хранятся эти цены, нам достаточно знать, что они есть. В коде нашей HTML-страницы мы хотим, чтобы эти цены вводил пользователь. Паттерн *Заместитель* позволяет скрыть информацию о том, где действительно хранятся данные объекта.

```javascript
class TariffProxy extends Tariff {
  constructor(kilometerId, minuteId) {
    super();

    this.kilometerId = kilometerId;
    this.minuteId = minuteId;
  }

  get kilometer() { return parseFloat(document.getElementById(this.kilometerId).value); }

  get minute() { return parseFloat(document.getElementById(this.minuteId).value); }
}
```
Теперь стоимость километра и минуты извлекается из полей ввода, при этом весь остальной код мы не трогаем. В случае необходимости
мы можем извлекать цены из файла конфигурации, и даже «зашить» их непосредственно в код.

## Заключение

См. [демонстрационный пример](demo.html).
