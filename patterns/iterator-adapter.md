# Итератор-адаптер

Паттерн *Адаптер* это переходник между двумя интерфейсами.

## Реализация *Адаптера* между &laquo;нашим&raquo; *Итератором* и *Итератором* из .NET (C#)

```c#
public interface IIterator<out T>
{
  T Current { get; }
  
  bool HasNext { get; }
  
  void MoveNext();
}

public static class EnumeratorExtensions
{
  public static IIterator<T> ToIterator<out T>(this IEnumerator<T> enumerator)
  {
    return new EnumeratorAdapter<T>(enumerator);
  }
  
  private class EnumeratorAdapter<out T> : IIterator<T>
  {
    private readonly IEnumerator<T> _enumerator;
    
    public EnumeratorAdapter(IEnumerator<T> enumerator)
    {
      _enumerator = enumerator;
      
      MoveNext();
    }
    
    public T Current => _enumerator.Current;
    
    public bool HasNext { get; private set; }
    
    public void MoveNext() => { HasNext = _enumerator.MoveNext(); }
  }
}
```

## Реализация *Адаптера* между *Итератором* и *Многострочным текстом* (JavaScript)

```javascript
'use strict';

class Iterator {
  constructor() { }

  get current() { throw new Error("Property 'current' is not overriden.") }

  get hasNext() { throw new Error("Property 'hasNext' is not overriden.") }

  next() { throw new Error("Method 'next' is not overriden.") }
}

class MultilineIteratorAdapter extends Iterator {
  constructor(multiline) {
    super();

    this._lines = multiline.split('\n');
    this._index = 0;
  }

  get current() { return this._lines[this._index] }

  get hasNext() { return this._index < this._lines.length }

  next() { this._index++ }
}
```

## Обсуждение

*Адаптер*&nbsp;&mdash; структурный паттерн из GoF, описан на стр. 141.

> Преобразует интерфейс одного класса в интерфейс другого, который ожидают клиенты. *Адаптер* обеспечивает совместную
работу классов с несовместимыми интерфейсами, которая без него была бы невозможна.

Подчёркиваю важные моменты:

1. Речь идёт об *интерфейсах*. В реалиях языка реализации *Адаптер* не конвертирует объект одного конкретного класса в другой.
Он, с одной стороны, реализует интерфейс или абстрактный класс, а с другой&nbsp;&mdash; получает интерфейс или абстрактный класс
в конструкторе. Несколько *Адаптеров* можно объединить в цепочку, если типы их входных и выходных интерфейсов совместимы.
В этом *Адаптер* похож на *Декоратор* с той разницей, что тип входного и выходного интерфейса у *Декоратора* один и тот же. Иногда
внутри *Адаптера* может быть конкретный класс, но снаружи *Адаптер* всегда реализует интерфейс.

2. У *Адаптера* один входной и один выходной интерфейс. Если за выходным интерфейсом мы прячем несколько входных, тогда
речь идёт о паттерне *Фасад*.
