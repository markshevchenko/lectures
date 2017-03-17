# Обработка исключений

В массовое сознание программистов *исключения* проникли через C++. Сейчас их можно встретить во всех языках-наследниках: Java, JavaScript, C#, PHP и так далее.
Языки функционального программирования можно использовать для удобной обработки исключений. Мы решим эту задачу на F# и C#, чтобы показать, что функциональный подход проще.

## Для чего нужны исключения

*Исключения* позволяют упросить код. Вложенность вызовов в программе может быть очень большой, и, если ошибка возникает где-то внизу, информацию о ней приходится в
явном виде передавать наверх через результаты функций. Как правило, на каждом уровне приходится писать код обработки. Вот как это выглядело когда-то на C:

```c
/*
** Программа для подсчёта количества вхождений строки в указанный файл.
** Вызов:
**   crep <строка> <имя_файла>
*/
#include <stdio.h>

int count_string(char* string, char* filename);

int main(int argc, char *argv[])
{
  int count;
  char *string;
  char *filename;

  if (argc != 3) {
    fputsf("Вызов: crep <строка> <имя_файла>", stderr);
    return -1;
  }

  string = argv[1];
  filename = argv[2];

  count = count_string(string, filename);

  if (count < 0) {
    perror("Can't open file");
    return -2;
  }

  printf("%d\n", count);

  return 0;
}

#define BUFFER_SIZE 1024

int count_string(char* string, char* filename)
{
  int count = 0;
  int eof = 0;
  char buffer[BUFFER_SIZE];
  FILE* input;

  if (NULL == string)
    return -1;

  if (NULL == filename)
    return -2;

  input = fopen(filename, "rt");

  if (input == NULL)
    return -3;

  while (NULL != fgets(buffer, BUFFER_SIZE, input)) {
    if (NULL != strstr(buffer, string))
      count++;
  }

  eof = feof(input);
  fclose(input);

  if (eof)
    return count;

  return -4;
}
```

Видим, что код получился большой и запутанный. Это потому, что у нас учебный пример. В реальной программе код будет ещё запутанней.

Как нам помогают *исключения*? Они позволяют не писать промежуточный код. Функции не должны больше ничего возвращать, если это не обусловлено их логикой.
Функции, которые возвращают результаты, не должны больше смешивать их в странных комбинациях с кодами ошибок.

Создать *исключения* не сложнее, чем вернуть значение. Мы видим, что плюсов много. Но есть и минусы.

Один из них заключается в том, что концепция *исключений* сложна для освоения. Недостаточно выучить ключевые слова или идиомы, необходимо усвоить набор принципов.
Если вы вставляете блок `try`/`catch` в каждом методе, вы неправильно применяете *исключения*.

## Внутренние исключения

Если в C++ вы можете бросить исключение любого типа, то в Java и C# тип исключения должен быть наследником класса `Exception`.
В обоих каркасах исключение может ссылаться на внутреннее исключение, то есть на свою причину. Такой подход
позволяет связывать исключения в цепочки.

```c#
public void CreateUser(string username, string email)
{
    using (var command = commandFactory.Create("INSERT INTO [Users] (username, email) VALUES (?, ?)", username, email))
    {
        CheckedExecuteNonQuery(command);
    }
}

private void ChekedExecuteNonQuery(SqlCommand command)
{
    try
    {
        command.ExecuteNonQuery();
    }
    catch (SqlException exception)
    {
        switch (exception.Number)
        {
        case -2:
            throw new TimeOut(exception);

        case 2601:
            throw new DuplicateException("There's a user with same email.", exception);
        }
    }
}
```

В C# существует класс `AggregateException`, с помощью которого можно собирать независимые исключения, например, возникшие в параллельных процессах. Таким образом исключения
могут образовывать не только цепочки, но и деревья.

Важный вопрос: зачем нам вообще цепочки исключений? [Википедия утверждает](https://en.wikipedia.org/wiki/Exception_chaining), что методы должны бросать исключения того же
уровня абстракции, на котором они находятся.

Звучит слишком абстрактно? Рассмотрим конкретный пример. Мы сохраняем в БД пользователя с существующим электронным адресом. В таблице пользователей создан уникальный индекс для поля `email`.
При выполнении команды `INSERT` возникнет исключение `SqlException` со свойством `Number` равным 2601. Для уровня предметной области эта информация слишком детальна.
Мы можем описать класс `DuplicateException` и создавать его в случае конфликта.

Код, который будет обрабатывать это исключение, должен будет знать только об уровне предметной области и о `DuplicateException`.
Если мы захотим изменить СУБД, или перейти на хранилище NoSQL, нам не придётся переписывать код высокого уровня, вырезая из него обработку `SqlException`.

С другой стороны, если возникла ошибка, нам нужна вся информация о её причинах, поэтому мы не должны просто так &laquo;терять&raquo; исключение низкого уровня.
Создавая экземпляр `DuplicateException`, мы сохраняем ислючение-причину в свойстве `InnerException`.

Реализуя лог ошибок, мы будем писать в него не только исключения высшего уровня, а всю цепочку целиком. Если ошибка возникнет
мы получим всю доступную информацию.

## Обработка исключений

Есть важный принцип, который применяется при работе с исключениями: *throw early, catch later*. Бросать исключения как можно раньше, обрабатывать как можно позже.
Мы не будем подробно останавливаться на первой части принципа, а перейдём ко второй, поскольку это и есть тема нашего обсуждения.

Если вы не знаете, что делать с исключенем, не обрабатывайте его, положитесь на код, который выше. На самом верху программы реализуйте обработчик, который будет
делать простые вещи. Например, если ваша программа&nbsp;&mdash; это REST-сервис, обработчик
[должен возвращать код и тексовое описание в строке статуса](http://www.w3.org/Protocols/rfc2616/rfc2616-sec6.html):

    HTTP/1.1 409 Confict

Полезно также вернуть [JSON с описанием ошибки](http://jsonapi.org/format/#errors):

```javascript
{
  errors: [
    {
      id: 'email',
      status: 409,
      title: 'Пользователь с таким электронным адресом уже зарегистрирован.',
      detail: 'Кажется, вы уже регистрировались у нас. Если вы забыли пароль, восстановите его.'
    }
  ]
}
```

В ASP.NET Web API 2 это делается с помощью [такого кода](https://docs.microsoft.com/en-us/aspnet/web-api/overview/error-handling/web-api-global-error-handling):

```c#
class JsonApiError
{
    public string Id { get; set; }

    public HttpStatusCode? Status { get; set; }

    public string Title { get; set; }

    public string Detail { get; set; }
}

class RestError
{
    [JsonIgnore]
    public HttpStatusCode Status { get; set; }

    public IReadOnlyCollection<JsonApiError> Errors { get; set; }
}

class RestErrorResult : IHttpActionResult
{
    private readonly HttpRequestMessage _requestMessage;
    private readonly RestError _restError;

    public RestErrorResult(HttpRequestMessage requestMessage, RestError restError)
    {
        _requestMessage = requestMessage;
        _restError = restError;
    }

    public Task<HttpResponseMessage> ExecuteAsync(CancellationToken cancellationToken)
    {
        var response = new HttpResponseMessage(_restError.Status);
        var json = JsonConverter.Serialize(_restError);
        response.Content = new StringContent(json);
        response.RequestMessage = _requestMessage;

        return Task.FromResult(response);
    }
}

class RestExceptionHandler : ExceptionHandler
{
    public override void HandleCore(ExceptionHandlerContext context)
    {
        var restError = ExtractRestError(context.Exception);
        context.Result = new RestErrorResult(context.ExceptionContext.Request, restError);
    }
}
```

Этот код решает несколько задач, поэтому выглядит сложным. Отбросив лишнее, обнаружим важный метод, который нам предстоит реализовать.
Речь идёт об `ExtractRestError`, который в качестве параметра получает исключение, а возвращает `RestError` с HTTP-статусом и массивом ошибок.

На входе у нас не просто исключение, а первое исключение в цепочке, или даже корень дерева, если где-то нам встретится `AggregateException`.
