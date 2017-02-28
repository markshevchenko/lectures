# Итератор-декоратор

## Текучий интерфейс

*Текучий интерфейс*&nbsp;&mdash; способ реализации методов класса, благодаря
которому программисты легче понимают код.

Правило текучего интерфейса: методы возвращают либо тот же объект, у которого они вызваны, либо
объект того же типа. Предположим, что у класса `String` есть метод `replace`, который не возвращает ничего,
то есть он просто меняет одну подстроку на другую:

```
var s = "abcdef";
s.replace("a", "z");
s.replace("b", "y");
```

Если метод `replace` будет возвращать изменённую строку, то мы могли бы записать этот же код так:

```
var s = "abcdef".replace("a", "z")
                .replace("b", "y");
```

Пример текущего интерфейса LINQ из C#:

```c#
// Читаем: сумма квадратов чётных чисел
public int EvenSquaresSum(IEnumerable<int> values)
{
  return values.Where(i => i % 2 == 0) // Читаем: выбираем чётные числа
               .Select(i => i * i) // Читаем: возводим в квадрат
               .Sum(); // Читаем: складываем
}
```

Текучий интерфейс удобно использовать для построения цепочки декораторов. Именно это происходит в примере выше. Метод
`Where` создаёт объект класса, реализующий интерфейс *Итератор*, который в конструкторе получает другой *Итератор*&nbsp;&mdash;
`values`. В то же время этот класс&nbsp;&mdash; *Декоратор*. Он добавляет функцию *фильтра* к интерфейсу *Итератора*,
то есть *декорирует* его.

## Реализация *Декоратора* `map` на JavaScript

```JavaScript
'use strict';

class Iterator {
  constructor() { }

  get current() { throw new Error("Property 'current' is not overriden.") }

  get hasNext() { throw new Error("Property 'hasNext' is not overriden.") }

  next() { throw new Error("Method 'next' is not overriden.") }

  map(f) { return new MapIteratorDecorator(this, f) }

  filter(predicate) { return new FilterIteratorDecorator(this, predicate) }
}

class MapIteratorDecorator extends Iterator {
  constructor(iterator, f) {
    super();
    this._iterator = iterator;
    this._f = f;
  }

  get current() { return this._f(this._iterator.current) }

  get hasNext() { return this._iterator.hasNext }

  next() { this._iterator.next() }
}
```

`MapIteratorDecorator` реализует операцию `Map`, которую можно назвать одним из паттернов функционального программирования.
На вход `Map` получает итератор и функцию, и применяет эту функцию к каждому элементу исходного итератора.

## Реализация *Декоратора* `filter` на JavaScript

```JavaScript
class FilterIteratorDecorator extends Iterator {
  constructor(iterator, predicate) {
    super();
    this._iterator = iterator;
    this._predicate = predicate;

    this._skip();
  }

  _skip() {
    while (this._iterator.hasNext && !this._predicate(this._iterator.current))
      this._iterator.next();

    this._hasNext = this._iterator.hasNext;
  }

  get current() { return this._iterator.current }

  get hasNext() { return this._hasNext }

  next() {
    this._iterator.next();
    this._skip();
  }
}
```
