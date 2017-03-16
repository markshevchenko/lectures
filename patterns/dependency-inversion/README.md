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

```json
{
  'routes': [
    {
      'legs': [
        {
          'steps': [
            {
              'distance': {
                'value': 123,
              },
              'duration': {
                'value': 32
              }
            },
            {
              'distance': {
                'value': 76,
              },
              'duration': {
                'value': 21
              }
            },
          ]
        },
        {
          'steps': [
            {
              'distance': {
                'value': 123,
              },
              'duration': {
                value': 32
              }
            },
            {
              'distance': {
                'value': 76,
              },
              'duration': {
                'value': 21
              }
            },
          ]
        },
      ]
    },
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

            document.getElementById('price').value = kilometerPrice * meters/1000.0;
          };

          xhr.onerror = function() {
            throw new Error(xhr.statusText);
          };

          xhr.open('GET', uri, true);
          xhr.send();
        }, false);
</script>

```