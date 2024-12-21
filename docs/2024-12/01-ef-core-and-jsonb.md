- <b style="color: blue">#ef_core</b>
- <b style="color: blue">#db_design</b>
- <b style="color: blue">#postgresql</b>

# Тонкости JSONB в PostgreSQL и EF Core: Как постичь дзен?

Сегодня мы рассмотрим работу с JSONB в базе данных PostgreSQL совместно с родной для .NET ORM от Microsoft — Entity Framework Core.

> [!WARNING]  
> Сразу отметим, что изложенные материалы не гарантируют актуальность для версий фреймворка (.NET, EF) ниже 8. Возможно, некоторые примеры будут работать (например, optimistic concurrency, data encryption), а другие — нет (например, современный подход к маппингу JSONB атрибутов через Owned Entities).

Мы разберём, как удобно хранить данные в денормализованном виде, как защититься от параллельных изменений и как поступать, когда необходимо выстраивать отношения между сущностями, чьи атрибуты упакованы в JSONB. Помимо этого, мы затронем такие темы, как optimistic concurrency, data encryption, миграции подобных сущностей и их версионирование.

Итак, начнём.

## Зачем вообще денормализовывать данные в реляционной БД ?

Предположим, мы знакомы с нормальными формами из теории реляционных БД. Они помогают устранить избыточность, уменьшить объём хранения, упростить поиск и снизить вероятность появления аномалий. Зачем же сознательно отходить от нормализации, если она даёт столько преимуществ?

Зачем же тогда сознательно уходить от нормализации? Ответ прост: иногда это удобнее. Разумеется, **при условии**, что **денормализация внедрена осознанно**, а не сделана «на скорую руку».

Это не отменяет ценности нормализации и знаний о нормальных формах. Однако в некоторых сценариях, рассматривая работу с БД с уровня приложения, мы можем пожертвовать строгой нормализацией ради улучшения удобства разработки (DX — Development Experience).

В самом деле, покажите мне инженера, у которого непроизвольно не дергался бы глаз при виде подобного кода

```cs
_dbContext.Orders
    .Include(x => x.Client)
        .ThenInclude(x => x.Address)
    .Include(x => x.Request)
        .ThenInclude(x => x.RoutePoints)
    .Include(x => x.Request)
        .ThenInclude(x => x.PaymentMethod)
    .Include(x => x.Request)
        .ThenInclude(x => x.BookingOptions)
    .Include(x => x.Details)
        .ThenInclude(x => x.Places)
    .Include(x => x.Details)
        .ThenInclude(x => x.AssignedCar)
            .ThenInclude(x => x.ActiveDriver)
    .Include(x => x.Details)
        .ThenInclude(x => x.Charges)
    .FirstOrDefault(x => x.Id == orderId)
```

Такой код разворачивается в множество JOIN’ов. Современные СУБД довольно хорошо оптимизированы под подобные запросы, но представьте, сколько кода нужно написать, чтобы настроить все эти отношения!

Это десятки файлов конфигураций для EF, миграции на сотни строк, постоянные вопросы «не переборщили ли мы с индексами» и раздутая кодовая база.

Особенно обидно, если во всех этих таблицах хранится небольшой объём данных, специфичный для конкретной сущности и редко меняющийся.

Теперь представим иную структуру:

| Attribute | Description                              |
| --------- | ---------------------------------------- |
| Id        | Первичный ключ                           |
| Details   | `JSONB` атрибут с информацией о сущности |
| CreatedAt | Время создания сущности                  |
| UpdatedAt | Время обновления сущности                |

В таком случае для получения записи по идентификатору достаточно одной строки:

```cs
_dbContext.Orders.FirstOrDefault(x => x.Id == orderId)
```

Намного проще, не так ли?

Но где остальные данные? Они внутри JSONB. Такой подход позволяет существенно упростить проектирование структуры базы и маппинг .NET типов (примечание для любителей тактических паттернов DDD). Мы храним данные «as is» в удобной для приложения форме.

<details>
  <summary>Структура JSONB атрибута</summary>
  
```json
{
  "clientId": "client_1",
  "orderStatus": "pending",
  "client": {
    "age": 29,
    "name": "Some Name",
    "phoneNumber": "+7 (xxx) xxx xx-xx",
    "address": {
      "city": "Some City",
      "country": "Some Country"
    }
  },
  "request": {
    "paymentMethod": "card_k3k2bmdb4wrghq",
    "routePoins": [
      { "latitude": 0, "longitude": 0 },
      { "latitude": 0, "longitude": 0 }
    ]
  },
  "details": {
    "places": [
      {
        "latitude": 0,
        "longitude": 0,
        "info": {
          "shortName": "Some short name",
          "longName": "Some Long Name"
        }
      },
      {
        "latitude": 0,
        "longitude": 0,
        "info": {
          "shortName": "Some short name",
          "longName": "Some Long Name"
        }
      }
    ],
    "assignedCar": {
      "id": "car_1",
      "activeDriver": {
        "id": "driver_1"
      }
    }
  }
}
```

</details>

## Использование JSONB в EF Core

Рассмотрим способы интеграции JSONB с EF Core. Есть два основных подхода:

- Новый подход: Owned Entities + `ToJson()`
- Старый подход: Custom Property Converter

> [!WARNING]  
> Есть и третий вариант: хранить данные как строку и десериализовать вручную. Но это неудобно и лишает нас множества возможностей, которые предоставляет EF Core и Npgsql. Хотя иногда этот метод может пригодиться.

Каждый вариант имеет свои плюсы, минусы и нюансы. Рассмотрим их на примере сущности, описывающей заказ клиента в нашей системе.

<details>

  <summary>Описание сущности в .net</summary>

Код .net класса выглядит следующим образом

```cs
public sealed class Order
{
    [Obsolete("Only for EFCore!", error: true)]
    public Order()
    {
    }

    public Order(
        string id,
        OrderDetails orderDetails)
    {
        Id = id;
        OrderDetails = orderDetails;

        SetDates(DateTime.UtcNow);
    }

    public string Id { get; private set; }
    public OrderDetails OrderDetails { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime UpdatedAt { get; private set; }

    public void UpdateDetails(
        Func<OrderDetails, DateTime, OrderDetails> updateDetails)
    {
        UpdatedAt = DateTime.UtcNow;
        OrderDetails = updateDetails(OrderDetails, UpdatedAt);
    }

    private void SetDates(DateTime utcNow)
    {
        CreatedAt = utcNow;
        UpdatedAt = utcNow;
    }

}

public enum OrderStatus
{
    Pending = 10,
    Execution = 100,
    Completed = 1000,
    Cancelled = 10000
}

public sealed class ClientContext
{
    public string Name { get; set; }
    public string Email { get; set; }
    public string PhoneNumber { get; set; }
    public int Age { get; set; }
    public ClientAddress Address { get; set; }
}

public sealed record ClientAddress
{
    public string City { get; set; }
    public string Country { get; set; }
    public string Raw { get; set; }
}

public sealed record OrderRequest
{
    public string PaymentMethod { get; set; }
    public List<GeoPoint> RoutePoints { get; set; }
}

public record GeoPoint
{
    public double Latitude { get; set; }
    public double Longitude { get; set; }
}

public sealed record OrderDetails
{
    public string ClientId { get; set; }
    public OrderStatus OrderStatus { get; set; }
    public ClientContext Client { get; set; }
    public OrderRequest Request { get; set; }
}

```

</details>

### Property Converter

До EF Core 8.0 (и соответствующей версии Npgsql-провайдера) единственным способом подключения функциональности `JSONB` была ручная сериализация/десериализация:

```cs
public sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(
        EntityTypeBuilder<Order> builder)
    {
        _ = builder.HasKey(x => x.Id);
        _ = builder.Property(e => e.OrderDetails)
            .HasColumnType("jsonb")
            .HasConversion(
                v => JsonSerializer.Serialize(v, JsonUtils.Options),
                v => JsonSerializer.Deserialize<OrderDetails>(v, JsonUtils.Options));
    }
}
```

В этом нет никакой магии — всё довольно просто и прозрачно. При этом, несмотря на кажущуюся грубоватость такого подхода к сериализации типов, `npgsql`-провайдер предоставляет нам возможности, выходящие за рамки простого хранения данных.

Мы можем обновлять эти данные и выполнять по ним простейшие запросы. Благодаря прямой сериализации в JSON, у нас есть свобода объявлять собственные JSON-конвертеры и гибко настраивать процесс сериализации и десериализации.

Однако, в таком подходе есть ряд нюансов.

#### Обновление данных

Первый нюанс связан с тем, как мы обновляем данные в `jsonb`. Поскольку применяются собственные преобразования типов, сущность фактически содержит граф объектов, не отслеживаемых стандартным механизмом `Change Tracking`. Из-за этого код может работать не так, как мы ожидаем:

```csharp
var entity = _dbContext.Entities.First(x => x.Id == id);

entity.OrderDetails.OrderId = "order_1";

_dbContext.SaveChanges();
```

Что в итоге произойдёт? Ничего. Обновления внутреннего свойства не попадут в БД. Причина в том, что для `Change Tracker` значение свойства не изменилось: ссылка осталась прежней, а о внутренних изменениях он не знает. В результате EF Core не генерирует никакого SQL для обновления.

Как быть? Есть два варианта:

**Указать EF Core вручную на изменения**

Один из вариантов — явно сообщить EF Core, что в сущности произошли изменения. Это можно сделать, изменив состояние сущности в механизме отслеживания.

**Обновить ссылку на свойство**

Другой вариант — заставить EF Core самостоятельно заметить обновления. Для этого можно полностью перезаписать свойство `OrderDetails`, создав его копию с помощью `with`:

```csharp
entity.OrderDetails = entity.OrderDetails with { OrderId = "order_1" };
```

Это же касается и коллекций:

```csharp
entity.OrderDetails.Route.Add(new GeoPoint(lat: 0, lon: 0)); // Не сработает!

// И даже когда коллекция является самим jsonb атрибутом таблицы
entity.Route.Add(new GeoPoint(lat: 0, lon: 0)); // И так не сработает. Принципы те же
```

Придётся либо пересоздать коллекцию, создав её копию с изменённым набором элементов, либо обернуть её в тип-обёртку и применить `with` к нему.

#### Работа с перечислениями (Enum)

Довольно часто нам приходится использовать перечисления (enum), чтобы описать различные статусы или состояния сущностей. Рассмотрим простой пример с моделью статусов поездки:

```csharp
public enum OrderStatus
{
    Pending = 10,
    Execution = 100,
    Completed = 1000,
    Cancelled = 10000
}
```

Хорошей практикой считается хранение в базе строковых значений перечислений, а не числовых. Причина в том, что числовые значения могут привести к проблемам при изменении перечисления. Например, если мы не задаём номера для каждого члена enum явно, то при добавлении нового значения в середину списка .NET автоматически переиндексирует все члены, сместив их значения.

Таким образом, у нас есть два пути:

1. Явно задавать числовые значения для каждого члена enum (как в примере выше).
2. Хранить значения enum в строковом формате.

Второй вариант нередко удобнее, поскольку при работе с `raw SQL` или анализе данных проще ориентироваться по строковым статусам, чем вспоминать соответствие числовых значений.

В .NET легко настроить преобразование enum в строки и обратно с помощью `StringEnumConverter`. Мы можем добавить `JsonStringEnumConverter` в конфигурацию `JsonUtils`, и тогда все перечисления автоматически будут сериализоваться и десериализоваться как строки.

```cs
public static class JsonUtils
{
    public static JsonSerializerOptions Options { get; } = new()
    {
        NumberHandling = JsonNumberHandling.AllowNamedFloatingPointLiterals,
        Converters =
        {
            new JsonStringEnumConverter(allowIntegerValues: true)
        }
    };
}
```

На первый взгляд, всё кажется в порядке: в базе появились строковые значения enum, и записи стали более понятными. Мы также защитили себя от ошибок, связанных с изменением порядка значений enum при его доработке. Однако это лишь видимость. Проблемы возникают, когда мы пытаемся выполнить фильтрацию по полю, хранящемуся в `jsonb`, используя логику enum:

```csharp
var activeOrders = await _dbContext.Orders
    .Where(x => x.Details.ClientId == clientId
        && x.Details.OrderStatus != OrderStatus.Completed
        && x.Details.OrderStatus != OrderStatus.Cancelled)
    .ToListAsync();
```

В реальности этот запрос приводит к ошибке. Причина в том, что EF Core не знает о наших настройках сериализации. Он не понимает, что enum фактически хранится в виде строки, и формирует запрос, предполагая числовые значения:

```sql
-- @orderStatus1 = 1000
-- @orderStatus2 = 10000
SELECT * FROM Orders o
WHERE o.Details->>'ClientId' = @client_id
    AND o.Details->>'OrderStatus' != @orderStatus1
    AND o.Details->>'OrderStatus' != @orderStatus2;
```

Здесь строковые значения (например, "completed", "cancelled") не сопоставляются с числовыми (1000, 10000), ожидаемыми EF Core.

Как с этим бороться? Один из вариантов — использовать «костыль» в виде вычисляемых колонок (`computed columns`). Давайте расширим таблицу и настроим её так, чтобы EF Core мог работать с enum корректно.

```cs
public sealed class Order
{
    //
    public OrderStatus OrderStatus { get; private set; }
    //
}

// entity configuration
_ = builder.Property(e => e.OrderStatus)
    .HasConversion<string>()
    .HasComputedColumnSql(stored: true,
        sql: @"""OrderDetails""->>'OrderStatus'");
```

Теперь, когда в таблице появилась вычисляемая колонка, мы можем писать запросы, опираясь на полноценный атрибут в базе данных:

```csharp
var activeOrders = await _dbContext.Orders
    .Where(x => x.Details.ClientId == clientId
        && x.OrderStatus != OrderStatus.Completed
        && x.OrderStatus != OrderStatus.Cancelled)
    .ToListAsync();
```

Однако важно учитывать один момент: значение в этой вычисляемой колонке нельзя изменить напрямую из кода. Оно пересчитывается только при модификации полей внутри `jsonb` и после вызова `SaveChanges`. Поэтому стоит заранее договориться в команде о сценариях использования таких атрибутов. Например, можно применять вычисляемую колонку исключительно для поиска с помощью EF Core, а во всей бизнес-логике опираться на соответствующее свойство из `OrderDetails`.

Если этот нюанс упустить из виду, можно столкнуться с неприятными неожиданностями в поведении приложения.

#### Проекции

Тут особо нечего сказать. Они почти всегда не работают, поскольку свойство сериализуется и десериализуется собственным конвертором и EF о нем ничего не знает. К сожалению, это очень большой минус, в некоторых случаях заставляющий поднимать довольно большое количество данных в ОЗУ

### Owned Entities + ToJson

Помимо ручной сериализации, `JSONB` теперь можно настраивать с помощью более современного подхода, представленного в [EF Core 7.0](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-7.0/whatsnew#json-columns) и поддержанного командой Npgsql в [провайдере версии 8.0](https://www.npgsql.org/efcore/release-notes/8.0.html#ef-json-support-via-tojson). Ранее мы были вынуждены использовать только старый метод с собственным конвертором для сериализации данных.

Давайте для начала разберём конфигурацию простой сущности, а затем перейдём к рассмотрению различных нюансов, связанных с использованием `Owned Entities`.

```cs
public sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(
        EntityTypeBuilder<Order> builder)
    {
        _ = builder.HasKey(x => x.Id);
        _ = builder.OwnsOne(x => x.OrderDetails,
            orderDetailsBuilder =>
            {
                _ = orderDetailsBuilder.ToJson();
                _ = orderDetailsBuilder.Property(x => x.OrderStatus).HasConversion<string>();
                _ = orderDetailsBuilder.OwnsOne(x => x.Request,
                    orderRequestBuilder =>
                    {
                        _ = orderRequestBuilder.OwnsMany(x => x.RoutePoints);
                    });

                _ = orderDetailsBuilder.OwnsOne(x => x.Client,
                    clientDetailsBuilder =>
                    {
                        _ = clientDetailsBuilder.OwnsOne(x => x.Address,
                            clientAddressBuilder =>
                            {
                            });
                    });
            });
    }
}
```

Теперь мы больше не занимаемся ручной сериализацией. При выполнении команды `dotnet ef migrations add [Name]` EF Core сам сгенерирует миграцию, в которой будут отражены маппинги для типа `OrderDetails`.

Это означает принципиально иной уровень взаимодействия: EF Core начинает понимать структуру `JSONB` и работать с ней через `Change Tracker`. В итоге мы получаем целый ряд существенных улучшений:

- Возможность генерировать точечные обновления без перезаписи всего свойства целиком.
- Поддержку конвертации отдельных атрибутов. Например, мы можем указывать, что хотим сохранять перечисления в виде строк, и благодаря этому поиск по ним не будет ломаться.
- Генерацию более оптимального SQL-кода.

Таким образом, этот подход значительно упрощает и улучшает работу с `JSONB` в EF Core.

Поэтому код вида

```cs
var entity = _dbContext
    .Entities.First(x => x.Id == id);

entity.OrderDetails.OrderId = "order_1";

_dbContext.SaveChanges();
```

будет работать именно так, как мы ожидаем: EF Core сгенерирует SQL, обновляющий только одно значение внутри `JSONB`, что-то вроде

```sql
UPDATE "Orders" SET "OrderDetails" = jsonb_set("OrderDetails", '{OrderId}', @p0)
WHERE "Id" = @p1;
```

> [!WARNING]  
> Данный запрос является псевдокодом, который описан для передачи смысла и разницы в подходах. Точный генерируемый SQL нужно смотреть во время дебага

В общем, стало действительно лучше. Однако, в этом подходе так же есть нюансы

#### Проекции

В случае с `Owned Entities` проекции так же начинают работать ! Причем довольно сложные и витееватые, описываются ровно так же через обычный `Select`

```cs
var data = await dataContext.Orders
    .AsNoTrackingWithIdentityResolution()
    .Where(x => x.OrderDetails.ClientId == clientId)
    .Select(x => new
    {
        x.Id,
        clientInfo = new
        {
            x.OrderDetails.Client.Name,
            x.OrderDetails.Client.PhoneNumber,
            x.OrderDetails.Client.Address
        }
    })
    .ToListAsync();
```

Один нюанс: чаще всего, придется добавить `AsNoTracking` или его разновидность (`WithIdentityResolution`), в том случае, когда мы достаем иерархичные структуры

#### Генерация миграций при изменении JSONB моделей

Поскольку все типы теперь описываются через `Owned Entities`, при любом изменении структуры моделей (добавлении или удалении полей) необходимо не забывать генерировать новые миграции. Например, если вы добавите или уберёте свойство, новая миграция может оказаться с пустыми методами `Up` и `Down`, но при этом обновит `ModelSnapshot`.

Такой подход необходим для актуализации настроек маппинга `JSONB` к .NET типам и позволяет EF Core корректно интерпретировать изменения в структуре данных.

#### Примитивные типы для коллекций

Ещё один важный нюанс: при использовании такого подхода нельзя применить примитивные типы коллекций для описания вложенных структур. Например, если в модели указано:

```csharp
public GeoPoint[] Route { get; set; }
```

то при добавлении новой миграции возникнет ошибка. Вместо массива стоит использовать коллекцию, например `List<GeoPoint>` или другой подходящий тип-коллекцию.

<details>
  <summary>Ошибка, генерируемая EF Core</summary>

Unable to create a 'DbContext' of type 'DataContext'. The exception 'Unable to determine the relationship represented by navigation 'OrderDetails.Route' of type 'GeoPoint[]'. Either manually configure the relationship, or ignore this property using the '[NotMapped]' attribute or by using 'EntityTypeBuilder.Ignore' in 'OnModelCreating'.' was thrown while attempting to create an instance. For the different patterns supported at design time, see https://go.microsoft.com/fwlink/?linkid=851728

</details>

#### Скрытые идентификаторы и конфликты имен

Проблема заключается в том, как EF Core работает с вложенными коллекциями объектов. Для отслеживания изменений EF Core сгенерирует скрытое свойство `Id` в каждом объекте `GeoPoint`:

```csharp
public List<GeoPoint> Route { get; set; }

// GeoPoint
public record GeoPoint
{
    public double Latitude { get; set; }
    public double Longitude { get; set; }
}
```

На первый взгляд, ничего страшного. Но если мы захотим добавить в `GeoPoint` собственное свойство с именем `Id`, то при попытке сгенерировать новую миграцию возникнет ошибка. Это связано с конфликтом между скрытым идентификатором, созданным EF Core, и нашим собственным свойством с тем же именем.

<details>
  <summary>Ошибка, генерируемая EF Core</summary>

Unable to create a 'DbContext' of type 'DataContext'. The exception 'Entity type 'Order.Route#GeoPoint' is part of a collection mapped to JSON and has its ordinal key defined explicitly. Only implicitly defined ordinal keys are supported.' was thrown while attempting to create an instance. For the different patterns supported at design time, see https://go.microsoft.com/fwlink/?linkid=851728

</details>

Таким образом, придется переименовать свойство. Это не критично, но может быть слегка раздражающим с эстетической точки зрения. Например, можно использовать названия вроде `ExternalId` или `PointId`.

#### Проблемы с обновлением через with

Ещё одна проблема связана с тем, как EF Core обрабатывает скрытые идентификаторы во вложенных коллекциях. Рассмотрим пример:

```csharp
order.UpdateDetails(x => x with
{
    OrderId = "order_1"
});
```

При попытке сохранить изменения в этом случае будет сгенерирована ошибка. Причина в том, что EF Core не сможет корректно перемаппить принадлежность коллекции `Route` от старой версии `OrderDetails` к новой: все `GeoPoint` уже имеют скрытые идентификаторы, связанные со старым экземпляром.

Есть два решения:

1. Не использовать перезапись ссылок (то есть не применять `with` для `OrderDetails`).
2. Создавать новые экземпляры `GeoPoint` при клонировании, чтобы EF Core воспринимал их как новые сущности.

Например:

```csharp
order.UpdateDetails(x => x with
{
    OrderId = "order_1",
    Route = x.Route.Select(p => p.Clone()).ToList(),
});

// GeoPoint
[DebuggerDisplay("{Latitude},{Longitude}")]
public class GeoPoint
{
    public GeoPoint Clone() => new(latitude: Latitude, longitude: Longitude);
}
```

Теперь при обновлении EF Core сгенерирует новые скрытые идентификаторы для каждого `GeoPoint`, и проблема будет решена.

## Deep Dive

Мы уже рассмотрели, какие способы работы с `jsonb` доступны, а также их основные плюсы и минусы.

Теперь пришло время погрузиться глубже и обсудить более тонкие вопросы, связанные с миграцией данных, шифрованием и обеспечением конкурентного доступа (concurrency) к данным.

### Индексирование

Индексирование данных — важная часть проектирования базы данных. Правильно выбранный индекс с высокой кардинальностью может значительно ускорить выборку даже среди сотен миллионов записей.

Рассмотрим наш пример: нам может потребоваться быстро находить заказы по `clientId`, который также хранится внутри `jsonb`-атрибута. Здесь есть несколько вариантов:

1. Индексировать напрямую путь к полю внутри `jsonb`.
2. Вынести нужное поле из `jsonb` в отдельную вычисляемую колонку (`stored computed column`) и индексировать уже её.

Выбор за вами. Лично я предпочитаю использовать явные вычисляемые колонки, и вот почему:

1. Такой подход легко реализуется средствами EF Core.
2. Он позволяет явно задекларировать набор индексов, существующих в таблице, что может служить подсказкой и индикатором проблем, если индексов становится слишком много.
3. Это упрощает моделирование отношений с другими таблицами, поскольку у вас есть явно определённый атрибут.

Однако у `stored computed column` есть и недостатки:

1. Изменения вычисляемых колонок приводят к выполнению DDL-операций и модификации схемы. Обычно это не проблема, но на очень больших объёмах данных может вызвать замедления.
2. Нельзя использовать некоторые типы данных, такие как `timestamptz`, поскольку PostgreSQL будет считать операцию не **immutable**, что ограничивает возможности создания вычисляемых колонок с такими типами.

Пример конфигурации:

```csharp
// Модель заказа
public sealed class Order
{
    public Order(
        string id,
        OrderDetails orderDetails)
    {
        Id = id;
        OrderDetails = orderDetails;

        SetDates(DateTime.UtcNow);

        // инициализация нужна просто для удобства и консистентности, но нужно помнить, что при сохранении значение перетрется
        ClientId = orderDetails.ClientId;
    }

    // ...
    public string Id { get; private set; }
    public string ClientId { get; private set; }
    // ...
}

// Конфигурация сущности Order
public sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        // ...
        _ = builder.Property(e => e.ClientId)
            .HasConversion<string>()
            .HasComputedColumnSql(
                sql: @"""OrderDetails""->>'ClientId'",
                stored: true);
    }
}
```

#### Моделирование отношений

При моделировании отношений к другим таблицам ничего не меняется. Сначала мы создаём `computed column`, а затем задаём отношения, как и в случае с любыми другими атрибутами сущности.

Этот подход позволяет продолжать использовать стандартные механизмы EF Core для моделирования и навигации по связям, не усложняя архитектуру приложения.

### Миграция данных

В первую очередь, важно обсудить миграцию данных — процесс, с которым мы сталкиваемся постоянно в активно развивающихся проектах. В этой статье мы предполагаем, что миграции должны быть обратно-совместимыми и выполняться без даунтаймов системы.

На самом деле, процесс миграции jsonb колонок во многом схож с обычными миграциями реляционных баз данных, находящихся в третьей нормальной форме (3НФ) и выше. Существуют несколько основных сценариев миграции, а именно:

- Удаление колонки (удаление свойства из jsonb)
- Добавление колонки (добавление нового свойства в jsonb)
- Переименование или изменение существующей колонки (изменение свойства в jsonb)
  Рассмотрим каждый из этих сценариев подробнее, акцентируя внимание на безопасной и обратно-совместимой миграции данных.

**Обратно-совместимые миграции**

Обратно-совместимые миграции позволяют вносить изменения в схему данных без нарушения работы существующих функциональностей и без необходимости остановки приложения. Это особенно важно для систем, работающих в режиме 24/7 и обслуживающих большое количество пользователей.

**Основные принципы обратно-совместимых миграций:**

1. Добавление новых полей без удаления старых: новые свойства добавляются параллельно с существующими, обеспечивая поддержку обеих версий данных.
2. Двойная запись данных: при обновлении сущности данные записываются как в новое поле, так и в старое. Это гарантирует, что новые записи соответствуют актуальной схеме, а существующие постепенно обновляются.
3. Постепенная миграция существующих данных: использование миграторов, которые асинхронно обрабатывают записи с устаревшей схемой и обновляют их до текущей версии.
4. Управление версиями схемы: хранение информации о версии схемы внутри jsonb атрибута позволяет отслеживать состояние каждой записи и применять необходимые изменения только к тем записям, которые этого требуют. Помимо этого, хранение версий схем поможет аналитикам и двх-шникам корректно выстраивать витрины для аналитики, плавно мигрируя свои собственные скрипты и пайплайны

#### Реализация обратно-совместимых миграций

В первую очередь, необходимо расширить нашу модель для сохранения версии схемы. Ниже приведен пример, как можно добавить работу с версиями в нашу модель. Обратите внимание, что версия добавляется только в `jsonb`.

Весь поиск будет происходить через фильтрацию именно по этому полю без какого-либо индексирования. Почему ? Потому что у такого индекса низкая кардинальность и он скорее навредит, чем сделает хорошо.

```cs
public sealed record OrderDetails
{
    // ... предыдущий код опущен для простоты
    public SchemaVersion SchemaVersion { get; set; } = SchemaVersion.V1;
}

public enum SchemaVersion
{
    V1 = 1,
    V2 = 2
}

// Конфигурация сущности Order, предыдущий код опущен для простоты
public sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        _ = builder.OwnsOne(x => x.OrderDetails,
            orderDetailsBuilder =>
            {
                _ = orderDetailsBuilder.Property(x => x.SchemaVersion).HasConversion<string>();
            });
    }
}
```

Добавление версии схемы является важным шагом. В идеале, это следует продумать уже на этапе создания таблицы, чтобы избежать необходимости миграции записей, у которых нет свойства версии.

Теперь добавим новое свойство, например, `double Rating`

```cs
public sealed record OrderDetails
{
    // ... предыдущий код опущен для простоты
    public double Rating { get; set; }
}
```

После этого добавляем новую миграцию с помощью команды `dotnet ef migrations add` для обновления снимка модели EF Core и сразу же применяем её на продакшене.

В итоге остаётся только одно: поддержать запись данных в новую колонку. Чаще всего, использование этого свойства будет необходимо аналитикам, чтобы понимать, где именно стоит ожидать новое поле с рейтингом, а где — нет.

Однако при удалении или изменении колонки ситуация становится сложнее. Шаги в основном похожи, поэтому рассмотрим сценарий, когда рейтинг становится более сложной структурой.

Допустим, по новым требованиям пользователь должен иметь возможность оценивать заказ несколько раз до его завершения и тем самым изменять предыдущие оценки. При этом бизнесу важно сохранять историю оценок для анализа качества различных этапов процесса выполнения заказа.

При этом, бизнесу важно сохранять историю оценок, чтобы с помощью команд аналитики анализировать качество различных этапов процесса выполнения заказа. Таким образом, могла бы подойти подобная структура

```json
{
  "Rating": {
    "Value": 5.0,
    "History": [
      {
        "RatedAt": "2024-12-17T19:55:55.7428998Z",
        "Value": 2.0,
        "Context": {
          "Stage": "Checkout"
        }
      },
      {
        "RatedAt": "2024-12-19T11:20:31Z",
        "Value": 5.0,
        "Context": {
          "Stage": "Delivery"
        }
      }
    ]
  }
}
```

Как видно, эта структура не совместима с нашим прежним `double Rating`. Здесь нам поможет версия схемы.

Во-первых, обновим нашу модель:

```cs
public sealed record OrderDetails
{
    // ... предыдущий код опущен для простоты
    public double Rating { get; set; }
    public ClientRating ClientRating { get; set; }
}

public sealed record ClientRating
{
    public double Value { get; set; }
    public List<RatingHistoryRecord> History { get; set; }
}

public sealed class RatingHistoryRecord
{
    public RatingHistoryRecord()
    {
    }

    public RatingHistoryRecord(double value,
        Dictionary<string, string> context,
        DateTime? ratedAt = null)
    {
        Value = value;
        Context = context;
        RatedAt = ratedAt ?? DateTime.UtcNow;
    }

    public double Value { get; set; }
    public DateTime RatedAt { get; set; }
    public Dictionary<string, string> Context { get; set; }
}
```

Во-вторых, обновим код, который обновляет значения рейтинга, учитывая версию схемы. Сделаем так, чтобы он смотрел на значение версии схемы и выполнял одну из двух веток

1. Поднимал версию схемы, если мы обновляем старый заказ. При этом, для поддержания обратной совместимости, лучше записать оба варианта
2. Просто обновлял существующее новое значение, если это заказ с новой версией схемы. При этом, обновление старого значения так же лучше оставить на случай непредвиденного роллбека

```cs
public async Task<Result> Update(UpdateRatingData data)
{
    var order = await _context.Orders.FirstOrDefault(x => x.Id == data.OrderId && x.ClientId == data.ClientId);
    if (order is null)
    {
        /// return error
    }

    if (order.OrderDetails
        is { SchemaVersion: SchemaVersion.V1 })
    {
        order.UpdateDetails((_, x) => x with
        {
            Rating = data.Value,
            SchemaVersion = SchemaVersion.V2,
            ClientRating = new()
            {
                Value = data.Value,
                History = [
                    new RatingHistoryRecord(data.Value, data.Context)
                ]
            }
        });
    }
    else
    {
        order.UpdateDetails((_, x) => x with
        {
            Rating = data.Value,
            ClientRating = x.ClientRating with
            {
                Value = data.Value,
                History = x.ClientRating.History
                    .Concat([new RatingHistoryRecord(data.Value, data.Context)]).ToList()
            }
        });
    }
}
```

Теперь обновим код, который читает значение рейтинга при отображении заказа клиенту

```cs
public async Task<OrderDto> Get(GetOrderData data)
{
    var order = await _context.Orders.FirstOrDefault(x => x.Id == data.OrderId && x.ClientId == data.ClientId);
    if (order is null)
    {
        /// return error
    }

    double rating = 0;

    if (order.OrderDetails
        is { SchemaVersion: SchemaVersion.V1 })
    {
        rating = order.OrderDetails.Rating;
    }
    else
    {
        rating = order.OrderDetails.ClientRating.Value;
    }

    return new()
    {
        //
    };
}
```

После прохождения проверки и успешного прогона тестов наступает этап раскатки. Поскольку изменения обратно-совместимы и мы намеренно сохранили запись старого значения даже для новой схемы, мы можем безболезненно откатиться в любой момент.

Теперь имеет смысл оставить эту версию приложения на несколько дней для наблюдения. Если всё в порядке, можно переходить к самому интересному — написанию мигратора.

В целом, миграторы можно реализовать с помощью IHostedService. Суть их проста: запускаем бесконечный цикл, который постоянно проверяет наличие старых версий записей и обновляет их до необходимого формата. Такая миграция может выполняться даже несколько месяцев, и, к слову, это абсолютно нормально.

Давайте напишем наш мигратор

```cs
public class OrderDetailsMigratorHostedService : IHostedService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<OrderDetailsMigratorHostedService> _logger;

    public OrderDetailsMigratorHostedService(
        IServiceProvider serviceProvider,
        ILogger<OrderDetailsMigratorHostedService> logger)
    {
        _serviceProvider = serviceProvider;
        _logger = logger;
    }

    public Task StartAsync(CancellationToken cancellationToken)
    {
        _taskCts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
        _task = Execute(_taskCts.Token);

        return Task.CompletedTask;
    }

    public async Task StopAsync(CancellationToken cancellationToken)
    {
        if (_task is null)
        {
            return;
        }

        try
        {
            _taskCts?.Cancel();
        }
        catch (Exception ex)
        {
            // handle ex
        }

        await _task.WaitAsync(cancellationToken)
            .ConfigureAwait(ConfigureAwaitOptions.SuppressThrowing);
    }

    private async Task Execute(CancellationToken ct)
    {
        using var scope = _serviceProvider.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<DataContext>();
        while (true)
        {
            try
            {
                // в идеале, тут нужно подумать ещё и о параллельных вычитках или concurrency, поэтому неплохо было бы добавить что-то вроде select for update skip locked или row version (предпочтительнее)
                var order = await context.Orders
                    .FirstOrDefaultAsync(x => x.OrderDetails.SchemaVersion == SchemaVersion.V1);

                if (order is null)
                {
                    return; // данных для миграции не осталось
                }

                // do update
                order.UpdateDetails((_, x) => x with
                {
                    SchemaVersion = SchemaVersion.V2,
                    ClientRating = new()
                    {
                        //
                    }
                });

                await context.SaveChangesAsync();
            }
            catch
            {
            }

            context.ChangeTracker.Clear();
        }

    }
}
```

Преимущества такой миграции заключаются в двух аспектах:

1. **Надёжность:** Как уже упоминалось, мигратор работает стабильно и непрерывно.
2. **Тестируемость:** Его можно покрывать тестами, обеспечивая дополнительную гарантию корректности миграции.

Итак, мы разработали мигратор и покрыли его тестами. Теперь, как и прежде, приступаем к его развёртыванию на продакшене.

После того как все данные будут успешно мигрированы, останется выполнить три шага:

1. Удалить мигратор.
2. Перевести код чтения и записи исключительно на версию V2 схемы.
3. Удалить из модели лишние поля, связанные с предыдущей версией.

Да, этот процесс может показаться трудоёмким, но, к сожалению, это неизбежная цена за надёжные релизы и поддержание стабильности системы.

### Data Encryption

Шифрование данных — одна из самых интересных и важных тем, затрагиваемых в этой статье. С практической точки зрения это критический аспект для проектов, обрабатывающих клиентскую или иную конфиденциальную информацию, особенно если необходимо соблюдать требования законодательства по хранению и обработке персональных данных.

Кроме того, данная задача интересна и технически: реализация шифрования в `EF Core` не всегда очевидна, а сложности могут возрастать в зависимости от того, как вы конфигурируете свои сущности (например, внутри переопределения `OnModelCreating` в `DbContext` или же в специализированных классах, реализующих `IEntityTypeConfiguration`).

Для начала разберёмся с вопросом: «Как и чем шифровать клиентские данные?» В `ASP.NET Core` уже предусмотрен удобный механизм [защиты данных (Data Protection)](https://learn.microsoft.com/en-us/aspnet/core/security/data-protection/introduction?view=aspnetcore-8.0). Он избавляет нас от необходимости самостоятельно придумывать алгоритмы шифрования, подбирать ключи, ротацию или хранение секретов.

Всё, что требуется от нас — создать `IDataProtector` с помощью `IDataProtectionProvider`, указав определённую цель (`purpose`) для шифрования выбранной категории данных. Например, можно создать единый дата-протектор для всей таблицы заказов:

```csharp
var protector = Provider.CreateProtector("orders::dp");
```

> [!WARNING]  
> Вариантов настройки Data Protection множество. Вы можете, например, создать отдельный дата-протектор для каждой колонки — тогда данные одной колонки нельзя будет расшифровать дата-протектором другой. Или же создавать дата-протектор для каждой сущности, добавляя в `purpose` её уникальный идентификатор. Такой подход максимально усложнит задачу потенциальному злоумышленнику, даже если ему удастся завладеть набором ключей и `purpose`.

Всё, что нам остается сделать в таком случае - пробросить созданный дата-протектор в конфигурации и начать использовать его для преобразования строк и данных в наших сущностях. Давайте посмотрим, как это сделать

Во-первых, давайте подключим механизм защиты данных в нашу систему и сконфигурируем его

```cs
ConnectionMultiplexer redisConnection = ConnectionMultiplexer.Connect(builder.Configuration["Storage:Redis:Connection"], cfg =>
{
    cfg.Password = builder.Configuration["Storage:Redis:Connection_Password"];
});
// ...
builder.Services
    .AddDataProtection(cfg =>
    {
        cfg.ApplicationDiscriminator = "efcore";
    })
    .PersistKeysToStackExchangeRedis(redisConnection, "dp-keys");
```

В приведённом примере используется хранение ключей защиты данных в `Redis`. Однако для «боевых» приложений стоит рассмотреть более надёжное персистентное хранилище и дополнительно зашифровать сами ключи. Например, можно хранить ключи в S3, используя шифрование через какой-либо Vault.

Теперь настроим нашу сущность таким образом, чтобы при сохранении в БД данные автоматически шифровались. Для этого потребуется немного поработать с провайдером и конфигурациями.

Основная сложность заключается в том, что логика работы провайдеров в `EF Core` достаточно неочевидна. Сам `EF Core` пользуется собственным внутренним провайдером, отдельным от того, что настраивается в приложении, чтобы не засорять его сотнями внутренних сервисов. Из-за этого EF Core в чистом виде «не знает» о том, что мы зарегистрировали какие-то дополнительные сервисы в провайдере приложения.

Кроме того, `GetService<T>` внутри контекста использует провайдер приложения, область видимости которого может быть ограничена. Если неосторожно получать из него зависимости, можно столкнуться с `ObjectDisposedException` в рантайме.

К счастью, в нашем случае эта проблема не возникнет, так как нужные нам сервисы зарегистрированы как `singleton`.

Итак, перейдем к коду. Сначала расширим наш `ModelCustomizer` и достанем экземпляр `IDataProtectionProvider`

```
internal sealed class BlogModelCustomizer : ModelCustomizer
{
    public BlogModelCustomizer(
        ModelCustomizerDependencies dependencies)
        : base(dependencies)
    {
    }

    public overorder void Customize(ModelBuilder modelBuilder,
        DbContext dataContext)
    {
        IDataProtectionProvider dataProtectionProvider =
            dataContext.GetService<IDataProtectionProvider>();

        _ = modelBuilder.ApplyConfiguration(new OrderConfiguration(dataProtectionProvider));
    }
}
```

Экземпляр сервиса можно передать в конфигурацию сущности. Для этого расширим конструктор класса конфигурации и реализуем шифрование необходимых клиентских данных.

```cs
public sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public OrderConfiguration(
        IDataProtectionProvider dataProtectionProvider)
    {
        DataProtector = dataProtectionProvider.CreateProtector("orders::dp");
    }

    private IDataProtector DataProtector { get; }

    public void Configure(
        EntityTypeBuilder<Order> builder)
    {
        _ = builder.HasKey(x => x.Id);
        _ = builder.OwnsOne(x => x.OrderDetails,
            orderDetailsBuilder =>
            {
                _ = orderDetailsBuilder.ToJson();

                _ = orderDetailsBuilder.Property(x => x.OrderStatus).HasConversion<string>();
                _ = orderDetailsBuilder.OwnsOne(x => x.Request,
                    orderRequestBuilder =>
                    {
                        _ = orderRequestBuilder.OwnsMany(x => x.RoutePoints);
                    });

                _ = orderDetailsBuilder.OwnsOne(x => x.Client,
                    clientDetailsBuilder =>
                    {
                        _ = clientDetailsBuilder.Property(x => x.Name)
                            .HasConversion(
                                x => DataProtector.Protect(x),
                                x => DataProtector.Unprotect(x));

                        _ = clientDetailsBuilder.Property(x => x.Email)
                            .HasConversion(
                                x => DataProtector.Protect(x),
                                x => DataProtector.Unprotect(x));

                        _ = clientDetailsBuilder.Property(x => x.PhoneNumber)
                            .HasConversion(
                                x => DataProtector.Protect(x),
                                x => DataProtector.Unprotect(x));

                        _ = clientDetailsBuilder.OwnsOne(x => x.Address,
                            clientAddressBuilder =>
                            {
                                _ = clientAddressBuilder.Property(x => x.Raw)
                                    .HasConversion(
                                        x => DataProtector.Protect(x),
                                        x => DataProtector.Unprotect(x));
                            });
                    });
            });
    }
}
```

Код, представленный выше, обеспечивает шифрование данных при их сохранении в БД для таких полей, как имя, адрес электронной почты, номер телефона и полный клиентский адрес.

И, по сути, на этом всё!

<details>

  <summary>Результат сохранения в БД</summary>

```json
{
  "Client": {
    "Age": 20,
    "Name": "CfDJ8Jr4xJHLMAVCgLvg01QoM0bO7xYJebi--auy83prWnwWS1K3munY7gHEJUzHmrWwZ0JCu2bGMLQrpL4GP9qQXZvbJ5ea6ifVfKHl8IYZo_lXQB1FjgF9XwDmAX5lOioqnw",
    "Email": "CfDJ8Jr4xJHLMAVCgLvg01QoM0YcPY_S07IF2Mlt6L38G0IJGYDYuRXyCsYRBDKcmIH9JFYFFOl_P-kgAOi8BtD3_FRm8_042Z3fIvBSOoq9wvFp3ZJEuYNFj2zrmrQnQNuK2-_5TweEo--dPxgMCaSLYTA",
    "Address": {
      "Raw": "CfDJ8Jr4xJHLMAVCgLvg01QoM0ZNQorPxFLziGEa-EsPg866BEq3adjtaVHGx0Aj5qRNBzgOEqtRgSArzgF8kDNWNlywY5p6gkQDV67OxLjbR_15qwWu2r1_xxJn8AWE13IrAh1I0pDe8dfgoXfsCztQWJJzF4PlNVvo1Qt-ni2kZa8N",
      "City": "Moscow",
      "Country": "Russia"
    },
    "PhoneNumber": "CfDJ8Jr4xJHLMAVCgLvg01QoM0Y2G4Ywh9gbtMSG9TXFAarW_qs7wE-8k8Bn31wqBoRcAzWO0YhagWdqaXXCTw6WVh2RsdQoNiAne0I2OBgAJ9UiJUzvu15ZWZBMD0hqA08mxw"
  },
  "Request": {
    "RoutePoints": [
      {
        "Latitude": 0,
        "Longitude": 0
      },
      {
        "Latitude": 0,
        "Longitude": 0
      }
    ],
    "PaymentMethod": "client::1::payment-method::1"
  },
  "ClientId": "client::1",
  "OrderStatus": "Pending"
}
```

</details>

С точки зрения кода приложения, это ничего не ломает. Мы можем делать проекции и выборки точечных полей — они будут расшифрованы автоматически при чтении из БД (однако, стоит учитывать, что если вы шифруете данные в БД, то поиск по ним станет невозможным)

```cs
var data = await dataContext.Orders
    .AsNoTrackingWithIdentityResolution()
    .Where(x => x.OrderDetails.ClientId == clientId)
    .Select(x => new
    {
        x.Id,
        clientInfo = new
        {
            x.OrderDetails.Client.Name,
            x.OrderDetails.Client.PhoneNumber,
            x.OrderDetails.Client.Address
        }
    })
    .ToListAsync();
```

### Concurrency Conflicts

При работе с базами данных, особенно в многопользовательских приложениях, возникает необходимость эффективно управлять одновременными изменениями одних и тех же записей. Конфликты конкуренции возникают, когда два или более процесса пытаются одновременно изменить одну и ту же запись. В таком случае важно определить, какие изменения сохранить, а какие отклонить, чтобы обеспечить целостность данных.

В контексте использования JSONB в PostgreSQL и EF Core необходимо быть особенно внимательным, поскольку фактически множество пользователей будет обновлять один и тот же атрибут.

Во многом степень опасности подобных обновлений зависит от того, каким образом вы вносите изменения в значения атрибута `JSONB`.

- **Обновление конкретных свойств:** Если вы изменяете отдельные свойства, например, используя конструкцию `order.OrderDetails.Property = value`, вероятность возникновения проблем снижается. Такой подход более безопасен, однако полностью избежать конфликтов конкуренции не всегда возможно.
- **Полное перезаписывание атрибута:** Если же вы полностью перезаписываете значение атрибута `JSONB`, это может привести к более серьёзным ситуациям. В таких случаях рекомендуется осуществлять контроль изменений вручную, чтобы предотвратить потерю данных или возникновение конфликтов.

Одним из эффективных способов управления такими конфликтами является использование системного столбца `xmin`. При этом, есть и другие механизмы обеспечения согласованности. Подробнее можно почитать в [документации](https://learn.microsoft.com/en-us/ef/core/saving/concurrency?tabs=data-annotations).

**Что такое xmin?**

В PostgreSQL каждый раз, когда происходит изменение записи, система автоматически обновляет специальный системный столбец `xmin`. Этот столбец хранит идентификатор транзакции, которая последней изменила запись. Использование `xmin` позволяет отслеживать изменения и определять, была ли запись обновлена после её последнего считывания в приложении.

#### Настройка xmin в EF Core

Необходимо сконфигурировать нашу модель

```cs
public sealed class Order
{
    public uint RowVersion { get; private set; }
}

public sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(
        EntityTypeBuilder<Order> builder)
    {
        //
        _ = builder.Property(x => x.RowVersion).IsRowVersion();
    }
}
```

#### Обработка конфликтов конкуренции с использованием xmin

Когда несколько пользователей или процессов пытаются одновременно обновить одну и ту же запись, может возникнуть конфликт конкуренции. С использованием xmin EF Core способен определить, была ли запись изменена после её последнего считывания, и соответственно обработать конфликт.

Например, это можно делать так (код намеренно упрощен)

```cs
public async Task<Result> MoveOrderToDelivery(UpdateOrderData data)
{
    int attempts = 0;
    while (true)
    {
        attempts++;
        try
        {
            return await ExecuteInternal(data);
        }
        catch (DbUpdateConcurrencyException)
        {
            if (attempts == 3)
            {
                throw;
            }
        }

        _dataContext.ChangeTracker.Clear();
    }
}

private async Task<Result> ExecuteInternal(UpdateOrderData data)
{
    var order = await _dataContext.Orders.FirstOrUpdate(x => x.Id == data.Id && x.ClientId == data.ClientId);

    if (order is null)
    {
        //
    }

    if (order.OrderDetails.Status != OrderStatus.Pending)
    {
        if (order.OrderDetails.Status == OrderStatus.Execution)
        {
            // статус уже был обновлен кем-либо ещё, просто завершаем работу процесса
            return new()
            {
            };
        }

        // возник конфликт и статус не тот, который позволяет переместить заказ в статус Execution
        // return error
    }

    order.UpdateDetails((_, x) => x with
    {
        //
    });

    await _dataContext.SaveChangesAsync();

    return new()
    {
    };
}
```

Решение о том, как конкретно разрешать те или иные конфликты нужно принимать каждый раз с учетом контекста, тут, к сожалению, универсального рецепта нет.

**Нормализация - как способ уменьшения конфликтов**

В контексте работы с `JSONB` в `PostgreSQL` и использованием `EF Core`, правильное проектирование структуры базы данных играет ключевую роль в обеспечении эффективности и стабильности приложения.

Одной из важнейших задач является выбор между нормализованной и денормализованной структурой данных. Нормализация позволяет значительно снизить вероятность возникновения конфликтов конкуренции и упрощает управление данными.

На первый взгляд, это может показаться противоречивым, особенно если учесть, что в начале статьи мы обсуждали, как денормализация может существенно упростить проектирование схемы базы данных. Однако это ощущение возникает лишь на поверхностном уровне.

Внимательные читатели вспомнят ключевую мысль, озвученную ранее: денормализация должна быть введена **осознанно** и **обдуманно**.

> **Денормализация ≠ отсутствие необходимости думать**

Кроме того, недостаточно просто денормализовать схему на этапе её создания. В процессе развития продукта **модель данных неизбежно будет усложняться** из-за изменений в продуктовых требованиях, и это совершенно нормально.

Ключевым аспектом здесь является регулярный анализ постоянно обновляющегося списка требований. Это позволит своевременно выявить ситуации, где вероятность конфликтов конкуренции становится высокой, и принять решения об оптимизации схемы данных.

Приведу пример. Допустим, у нас есть таблица Cars, которая содержит информацию об автомобиле, рабочей смене и координатах автомобиля, передающихся каждые 2–3 секунды.

```cs
public selaed class Car
{
    public CarDetails Details { get; private set; }
}

public sealed record CarDetails
{
    public GeoPoint Position { get; set; }
    public ShiftInfo ShiftInfo { get; set; }
    public CarAttributes CarAttributes { get; set; }
}
```

С одной стороны, такой дизайн удобен тем, что все данные об автомобиле находятся в одном месте. С другой — в данном примере существует как минимум три участника, которые могут изменять информацию об автомобиле:

1. Водители
2. Устройство, передающее информацию о GPS
3. Сотрудники операций, редактирующие информацию об автомобилях

В результате, из-за частого стриминга координат, процессы обновления информации об автомобиле или смене будут часто сталкиваться с проблемами конкуренции (`concurrency issues`). Наш механизм повторных попыток сможет справиться с этим, но в логах будет накапливаться множество ошибок.

Чтобы этого избежать, мы можем немного переосмыслить структуру и найти лаконичное решение, позволяющее нормализовать отношения и устранить проблемы конкуренции.

Например, разделив модель автомобиля с текущей схемой на три разные таблицы

```cs
public class Car
{
    public CarDetails Details { get; private set; }

    // relations
    public CarGpsInfo GpsInfo { get; private set; }
    public DrivingSession ActiveSession { get; private set; }
    public List<DrivingSession> DrivingSessions { get; private set; }
}

public class DrivingSession
{
    public DrivingSessionDetails Details { get; private set; }
}

public class CarGpsInfo
{
    public CarGpsInfoDetails Details { get; private set; }
}

public record CarDetails
{
    public CarAttributes CarAttributes { get; set; }
}
```

Как видим, мы всё ещё продолжаем работать с JSONB атрибутами в каждой из таблиц. Однако, разделив различные обновляемые части автомобиля на отдельные таблицы, мы уменьшили количество источников обновления одной записи, что снижает вероятность возникновения конфликтов конкуренции.

Комбинация разных подходов позволяет эффективно управлять сложными структурами данных, минимизируя риски конфликтов конкуренции и обеспечивая стабильность работы приложения. Своевременный анализ изменяющихся требований и осознанный подход к проектированию схемы данных помогут избежать множества потенциальных проблем и обеспечить устойчивость системы в долгосрочной перспективе.

## Резюме

`JSONB` может существенно упростить проектирование базы данных и маппинг моделей, позволяя хранить данные в естественной для приложения форме. Однако этот инструмент требует осознанного подхода. Без продуманного дизайна мы рискуем столкнуться с проблемами конкурентности, производительности и сложностями миграций.

Мы рассмотрели варианты использования `JSONB` в `EF Core` — от классической сериализации через свой конвертер до нового подхода с Owned Entities и ToJson(), а также затронули вопросы enum, индексирования, конкуренции, шифрования и миграций.

Надеемся, что этот материал поможет лучше понимать возможности и ограничения `JSONB` и `EF Core`, и принимать более взвешенные решения при работе с данными в ваших проектах.
