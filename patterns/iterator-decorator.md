# Итератор-декоратор

Есть неверное мнение, что Fluent Interface (Fluent Syntax)&nbsp;&mdash; это одно, а цепочка Декораторов&nbsp;&mdash; это другое,
и нельзя их использовать вместе.

В действительности эти вещи независимы друг от друга, и могут применяться вместе, как это происходит в LINQ. Чтобы убедиться в этом,
реализуем методы `Map`, `Reduce` и `Filter` для интерфейса `IIterator<T>`.

## `Map`

public static class IteratorExtensions
{
  public static IIterator<T> Map(this IIterator<T>, Func<T, TResult> mapper)
  {
  
  }
  
  private class MapIterator<out T> : IIterator<T>
  {
  
  }
}
