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
        CheckedExecuteNonQuery(command, "Ошибка при создании пользователя.");
    }
}

private void ChekedExecuteNonQuery(SqlCommand command, string message)
{
    try
    {
        command.ExecuteNonQuery();
    }
    catch (Exception exception)
    {
        throw new EntityException(message, exception, typeof(User));
    }
}
```

![Цепочка исключений](exception-chain.png)

В C# существует класс `AggregateException`, с помощью которого можно собирать независимые исключения, например, возникшие в параллельных процессах. Таким образом исключения
могут образовывать не только цепочки, но и деревья.

![Дерево исключений](exception-tree.png)

Важный вопрос: зачем нам вообще цепочки исключений? [Википедия утверждает](https://en.wikipedia.org/wiki/Exception_chaining), что методы должны бросать исключения того же
уровня абстракции, на котором они находятся.

Звучит слишком абстрактно? Рассмотрим конкретный пример. Мы сохраняем в БД пользователя с существующим электронным адресом. В таблице пользователей создан уникальный индекс для поля `email`.
При выполнении команды `INSERT` возникнет исключение `SqlException` со свойством `Number` равным 2601. Для уровня предметной области эта информация слишком детальна.
Мы можем описать класс `EntityException` и создавать его в случае конфликта.

Код, который будет обрабатывать это исключение, должен будет знать только об уровне предметной области и о `EntityException`.
Если мы захотим изменить СУБД, или перейти на хранилище NoSQL, нам не придётся переписывать код высокого уровня, вырезая из него обработку `SqlException`.

С другой стороны, если возникла ошибка, нам нужна вся информация о её причинах, поэтому мы не должны просто так &laquo;терять&raquo; исключение низкого уровня.
Создавая экземпляр `EntityException`, мы сохраняем ислючение-причину в свойстве `InnerException`.

Реализуя лог ошибок, мы будем писать в него не только исключения высшего уровня, а всю цепочку целиком. Если ошибка возникнет,
мы получим всю доступную информацию.

## Обработка исключений

Есть важный принцип, который применяется при работе с исключениями: *throw early, catch later*. Бросать исключения как можно раньше, обрабатывать как можно позже.
Мы не будем подробно останавливаться на первой части принципа, а перейдём ко второй, поскольку это и есть тема нашего обсуждения.

Если вы не знаете, что делать с исключенем, не обрабатывайте его, положитесь на код, который знает&nbsp;&mdash; это и есть *catch later*. Кажется, что таких
обработчиков может быть много, но на практике речь идёт об одном или двух. Почему?

Рассмотрим ситуацию на примере REST-сервиса. Что делает REST-сервис в случае ошибки?
Он возвращает [код и тексовое описание в строке статуса](http://www.w3.org/Protocols/rfc2616/rfc2616-sec6.html):

    HTTP/1.1 409 Confict

Он может вернуть [JSON с детальным описанием](http://jsonapi.org/format/#errors):

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

Метод, обрабатывающий запрос REST и есть код, который знает, что делать с исключением: если &laquo;что-то пошло не так&raquo;, надо вернуть подходящий
код ошибки и подходящий объект JSON.

```c#
public class UserController : ApiController
{
    private readonly IUserRepository userRepository;
    private readonly DtoMapper dtoMapper;

    . . .

    public IHttpActionResult Get(int id)
    {
        try
        {
            var user = userRepository.ReadById(id);
            var userDto = dtoMapper.Map(user);
            return Content(HttpStatusCode.OK, userDto, new JsonMediaTypeFormatter()); 
        }
        catch (Exception exception)
        {
            var restError = GetRestError(exception);
            return Content(httpError.Status, httpError.Object, new JsonMediaTypeFormatter());
        }
    } 
}
```

Мы видим, что каждый метод сервиса надо оборачивать в конструкцию 'try'/'catch', при этом способ обработки исключений во всех методах один и тот же.
Такое решение нарушает принцип [DRY](https://ru.wikipedia.org/wiki/Don%E2%80%99t_repeat_yourself).

Но в XXI веке все промышленные библиотеки позволяют внедрить глобальный обработчик, куда попадут всенеобработанные исключения.
Здесь мы их исследуем, и вернём пользователю корректный HTTP-ответ. Код REST-запросов станет гораздо чище.

```c#
public class UserController : ApiController
{
    private readonly IUserRepository userRepository;
    private readonly DtoMapper dtoMapper;

    . . .

    public UserDto Get(int id)
    {
        var user = userRepository.ReadById(id);

        return dtoMapper.Map(user);
    } 
}
```

В ASP.NET Web API 2 глобальный обработчик исключений [выглядит так](https://docs.microsoft.com/en-us/aspnet/web-api/overview/error-handling/web-api-global-error-handling):

```c#
class JsonApiError
{
    public string Id { get; set; }

    public HttpStatusCode? Status { get; set; }

    public string Title { get; set; }

    public string Detail { get; set; }
}

class JsonApiTopLevelError
{
    public IReadOnlyCollection<JsonApiError> Errors { get; set; }
}

class RestError
{
    public HttpStatusCode Status { get; set; }

    public JsonApiTopLevelError Object { get; set; }
}

class RestErrorResult : IHttpActionResult
{
    private readonly HttpRequestMessage requestMessage;
    private readonly RestError restError;

    public RestErrorResult(HttpRequestMessage requestMessage, RestError restError)
    {
        this.requestMessage = requestMessage;
        this.restError = restError;
    }

    public Task<HttpResponseMessage> ExecuteAsync(CancellationToken cancellationToken)
    {
        var response = new HttpResponseMessage(restError.Status);
        var json = JsonConverter.Serialize(restError.Object);
        response.Content = new StringContent(json);
        response.RequestMessage = requestMessage;

        return Task.FromResult(response);
    }
}

class RestExceptionHandler : ExceptionHandler
{
    public override void HandleCore(ExceptionHandlerContext context)
    {
        var restError = GetRestError(context.Exception);
        context.Result = new RestErrorResult(context.ExceptionContext.Request, restError);
    }
}
```

Этот код решает несколько задач, поэтому выглядит сложным. Мы не будем обсуждать детали, а сразу перейдём к главному методу.
`GetRestError` получает на вход исключение (за которым, как мы помним, прячется дерево исключений), а на выходе возвращает
статутс HTTP и объект, который мы сериализуем в JSON.

## Реализация обработки на C# #

Как обработка исключений выглядит на императивном объекто-ориентированном языке? Как мы определим, что пользователь с таким электронным
адресом уже есть в базе? На уровне предметной области у нас будет `EntityException`, откуда мы извлечём тип
объекта (`EntityException.EntityType == typeof(User)`). На инфраструктурном уровне мы получим внутреннее исключение `SqlException` с полем
`Number` равным 2601.

Эта пара исключений должна встретиться нам где-то в дереве исключений. Чтобы пройти по дереву, используем паттерн *Посетитель* (*Visitor*).
Создадим абстрактный *Элемент* и метод расширения *Visit*:

```c#
abstract class Element
{
    public abstract bool Accept(Exception exception);
}

static class ExceptionExtensions
{
    public static void Visit(this Exception exception, Element element)
    {
        if (element.Accept(exception))
            return;

        var aggregateException = exception as AggregateException;
        if (aggregateException != null)
        {
            foreach (var innerException in aggregateException.InnerExceptions)
                innerException.Visit(element);

            return;
        }

        if (exception.InnerException != null)
            exception.InnerException.Visit(element);
    }
}
```

Здесь всё по учебнику. Если вы не понимаете, как работает этот код, обратитесь к описанию паттерна *Посетитель* в книге банды четырёх (GoF).
Последнее, что нам осталось: реализовать конкретный *Элемент*, который будет собирать информацию об ошибке.

```c#
class RestErrorElement : Element
{
    public HttpStatusCode Status { get; protected set; }

    public string Title { get; private set; }

    RestErrorElement()
    {
        Status = HttpStatusCode.InternalServerError;
        Title = "На сервере возникла внутренняя ошибка. Скоро всё поправим.";
    }

    public override bool Accept(Exception exception)
    {
        var entityException = exception as EntityException;
        if (entityException != null)
        {
            var sqlException = exception.InnerException as SqlException;
            if (sqlException != null)
            {
                if (sqlException.Number == 2601)
                {
                    Status = HttpStatusCode.Conflict;

                    if (entityException.EntityType == typeof (User))
                        Title = "Пользователь с таким электронным адресом уже существует.";
                    else if (entityException.EntityType == typeof (Document))
                        Title = "Документ с таким названием уже существует.";
                    else
                        Title = "Сущность с таким названием уже существует.";

                    return true;
                }
            }
        }

        return false;
    }
}
```

Метод `Accept` будет вызван для каждого исключения в дереве. Он выглядит не очень сложным, но следует иметь в виду, что полная проверка
всех исключений сделает его большим и запутанным.

## Реализация обработки на F# #

Теперь посмотрим, как та же задача решается в функциональном стиле. Мы берём дерево исключений и функцию, которая применяет набор
*правил* к каждому исключению в дереве. Если правило *срабатывает*, то его результатом будет статус HTTP и описание ошибки.

Правило это набор условий и результат&nbsp;&mdash; код и описание. Условия (предикаты)&nbsp;&mdash; функции, которые берут исключение
возвращают `true`, если оно удовлетворяет условию.