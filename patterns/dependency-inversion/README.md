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
            {
              distance: {
                value: 76,
              },
              duration: {
                value: 21
              }
            },
            . . .
          ]
        },
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
            {
              distance: {
                value: 76,
              },
              duration: {
                value: 21
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

Если расчёт цены маршрута ведётся на основании стоимости километра, то используем формулу

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

К нам приходит руководство, и говорит, что теперь нам надо строить маршруты не только на клиенте, но и на сервере&nbsp;&mdash; в Node.js.

Наш код не готов к быстрой реализации этого требования. На практике код должен быть не только простым для понимания, но и простым для изменения. Что затрудняет изменения?
Это связи между разными участками кода&nbsp;&mdash; мы меняем код в одном месте, и это заставляет нас менять код в другом.

Значит, задача заключается в том, чтобы ограничивать связи между разными участками кода.

Первое, что бы сделаем&nbsp;&mdash; разобьём код на уровни. В больших проектах, на уровни разбивают модули, но наш проект учебный, поэтому мы будем разносить отдельные функции.
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

Функция `buildRoute` строит маршруты. Она ничего не знает про поля ввода `from`, `to`, и `key`, поэтому её можно вызывать и на сервере тоже. Она также ничего не знает про расчёт цены,
поскольку это знание относится к предметной области. Функция запрашивает маршрут у Google Maps Directions API и передаёт его в функцию `callback`.

Получив маршрут на уровне предметной области, мы складываем длину всех участков и вычисляем общую продолжительность маршрута. Умножив её на стоимость километра получаем общую стоимость
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
компания Google решит внести изменения в свой код, нам также придётся вносить изменения в свой.

Однако, зависимость с верхним уровнем представления у нас разорвана: мы польше не знаем про поле ввода `price` и не меняем его значение, как раньше.

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

На верхнем уровне мы извлекаем значения из полей `from`, `to`, `key` и `kilometer-cost`, и меняем значение поля `price`. Мы вызываем функции нижних уровней `buildRoute` и `calculatePrice`.
Эти функции мы можем спокойно вызывать и в серверном коде, потому что они теперь не обращаются к методу `getElementById` объекта `document`.
