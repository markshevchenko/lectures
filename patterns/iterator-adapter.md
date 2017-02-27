# Итератор-адаптер

Паттерн *Адаптер* это переходник между двумя интерфейсами.

## Реализация *Адаптера* между *Итератором* и *Перечислителем*

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

## Реализация *Адаптера* между *Итератором* и *Массивом*

```c#
public static class ArrayExtensions
{
  public static IIterator<T> ToIterator<out T>(this IReadOnlyList<T> array)
  {
    return new ReadOnlyListAdapter<T>(array);
  }
  
  private class ReadOnlyListAdapter<out T> : IIterator<T>
  {
    private readonly IReadOnlyList<T> _array;
    private int _index;
    
    public ReadOnlyListAdapter(IReadOnlyList<T> array)
    {
      _array = array;
      _index = 0;
    }
    
    public T Current => _array[_index];
    
    public bool HasNext => _index < _array.Length;
    
    public void MoveNext() => { _index++; }
  }
}
```

```javascript
"use strict";

class Iterator {
  constructor() {
  }
  
  get current() {
    throw new Error("Property 'current' is not overriden.");
  }
  
  get hasNext() {
    throw new Error("Property 'hasNext' is not overriden.");
  }
    
  next() {
    throw new Error("Method 'next' is not overriden.");
  }
}

class ArrayIterator extends Iterator {
  constructor(a) {
    super();
    this._a = a;
    this._i = 0;
  }

  get current() {
    return this._a[this._i];
  }
  
  get hasNext() {
    return this._i <  this._a.length;
  }
    
  next() {
    this._i++;
  }
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
внутри может быть конкретный класс, но снаружи *Адаптер* всегда реализует интерфейс.

2. У *Адаптера* один входной и один выходной интерфейс. Если за выходным интерфейсом мы прячем несколько входных, тогда
речь идёт о паттерне *Фасад*.
