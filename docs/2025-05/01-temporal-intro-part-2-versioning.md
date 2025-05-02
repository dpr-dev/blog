- <b style="color: blue">#temporal</b>
- <b style="color: blue">#distributed_transactions</b>
- <b style="color: blue">#orchestration</b>

# Temporal: Введение. Часть 2: workflow versioning

В этой статье мы продолжим говорить о Temporal, а именно о том, какие подходы существуют для версионирования temporal workflow.

Она является продолжением цикла статей о temporal и нашем опыте работы с ним. Предыдущие статьи можно найти тут

- [Temporal: Введение. Часть 1](https://dpr-dev.github.io/blog/#/2024-11/01-temporal-intro-part-1)
- [Temporal: Введение. Часть 2: workflow versioning (эта статья)](https://dpr-dev.github.io/blog/#/2025-01/01-temporal-intro-part-2-versioning)

## Проблема версионирования Workflow в Temporal

Прежде чем обсуждать конкретные подходы к версионированию workflow в Temporal, стоит подробно остановиться на самой сути проблемы — почему это вообще важно и откуда возникают сложности.

В первой части цикла я упоминал, что Temporal сохраняет **историю выполнения workflow** в базе данных. Это одно из ключевых преимуществ системы: мы получаем надёжное восстановление процессов после падений, возможность резюмировать выполнение с любого шага, автоматическую защиту от сбоев и дедупликацию действий. Но за эту устойчивость приходится платить ограничениями: **Temporal повторно исполняет код workflow в момент восстановления или повторной активации**, строго следуя истории событий.

Когда workflow восстанавливается, Temporal "проигрывает" его заново: вызывает `activity`, проверяет `timer`-ы, сравнивает их с сохранёнными результатами и ждёт полного совпадения. Если выполняемый код **хоть немного отличается** от того, который был при изначальном запуске (например, поменялась сигнатура `activity`, добавилось новое условие, изменилась логика ветвления), возникает расхождение. В этом случае Temporal не может безопасно продолжать выполнение — и workflow просто **"замирает"** в невалидном состоянии. Он не завершится, не упадёт, не вернётся к жизни. Единственный выход — **форсированно завершить процесс** или попытаться "откатиться" назад с помощью `reset` и задеплоить старую версию кода.

Однако `reset` — это не универсальное спасение:

- Он требует точного понимания момента, с которого нужно перезапустить выполнение.
- Системы, с которыми взаимодействует workflow, могут не быть идемпотентными — повторный вызов тех же ручек приведёт к побочным эффектам или ошибкам.
- Возможность `reset` зависит от логики самих workflow и их интеграций (например, нельзя reset-ить workflow, если были запущены child workflow).

Очевидно, что в продакшене **так жить нельзя** (проверено:)). Любая попытка обновить бизнес-логику, изменить шаги процесса, переименовать `activity` или добавить новую проверку может **уронить** активные процессы, которые всё ещё находятся в состоянии исполнения.

Поэтому ключевой вопрос для каждого, кто использует Temporal в бою, — **как вносить изменения в workflow, не ломая обратную совместимость?**

**Хорошая новость:** Temporal предоставляет несколько инструментов и паттернов, которые позволяют **эволюционировать workflow безопасно**, без простоя и разрушений.

**Плохая новость:** каждый из этих способов требует внимания, дисциплины и осознанного выбора архитектурного подхода.

Кстати, ещё вы столкнётесь не только с необходимостью версионировать сам workflow, но и с тем, что **интеграции** с другими системами тоже потребуют версионирования, фичефлагов или адаптационного слоя. Иначе — никакого zero-downtime и стабильных релизов не получится.

## Подходы к версионированию Temporal Workflow

Теперь, когда мы обсудили, почему тема версионирования в Temporal настолько критична, можно перейти к разбору конкретных подходов, которые позволяют безопасно вносить изменения в бизнес-процессы.

Важно понимать, что Temporal — это не просто “исполнитель кода”, а система, построенная вокруг детерминированного воспроизведения истории. Любые изменения в коде workflow должны учитывать это фундаментальное ограничение. Поэтому при разработке вам придётся не только писать логику, но и заранее проектировать стратегию её эволюции.

В Temporal существует несколько популярных способов версионирования workflow. Все они имеют разные цели, уровень трудозатрат и масштаб влияния на инфраструктуру. Ни один из них не является универсальным — подход нужно выбирать в зависимости от контекста, типа изменений и архитектурных ограничений вашего проекта.

Вот основные стратегии, которые мы разберём:

- Дублирование Workflow-классов — один из самых простых, но не всегда масштабируемых способов.
- Версионирование через Task Queue — изоляция процессов по отдельным очередям выполнения.
- Использование Patched API (Workflow.Patched) — фреймворк-специфичный способ встраивания feature-flag логики.
- Версионирование воркеров (Worker Versioning) — перспективный, но пока экспериментальный подход, направленный на автоматическую изоляцию по версиям.

Каждая стратегия имеет свои плюсы и минусы, которые зависят от архитектурных решений: используете ли вы монорепозиторий или микросервисы, как устроены интеграции, насколько часто вы вносите breaking changes и т. д.

В следующих подразделах мы подробно рассмотрим каждый из этих подходов: где они работают лучше всего, какие ошибки стоит избегать и что из всего этого мы используем в продакшене.

### Дублирование workflow

Одним из самых простых и интуитивно понятных способов внесения изменений в Temporal workflow является дублирование. Суть подхода проста: вместо того чтобы менять существующий workflow, вы создаёте новый класс (или тип) с другой сигнатурой, именем и логикой. Например, вместо ProcessPayment вы создаёте ProcessPaymentV2, и с этого момента запускаете в продакшене именно его.

Такой подход может показаться наивным, но он отлично работает в ряде сценариев:

- ваш workflow не вложенный и не вызывается из других workflow;
- вы не используете сигналы или query, которые сильно завязаны на внешний контракт;
- вы можете чётко отделить запуск новых процессов от старых (например, на уровне контроллера, API или task dispatcher).

Workflow в Temporal — это по сути "бесконечный процесс", живущий до завершения. Создавая новый тип workflow, вы не трогаете уже существующие экземпляры. Они продолжают исполняться в своём оригинальном виде, пока не завершатся естественным образом. Все новые запуски можно направить на новую версию, просто изменив точку входа (например, контроллер или event consumer).

Если вы вызываете Temporal workflow из .NET-контроллера или любого другого клиента, переключение на новую версию может быть столь же простым, как замена типа в вызове StartWorkflowAsync:

```csharp
WorkflowHandle<UserGrantedWorkflowV2> handle = await _temporalClient
    .StartWorkflowAsync<UserGrantedWorkflowV2>(x => x.Run(args), options);
```

При этом предыдущие процессы (UserGrantedWorkflowV1) останутся активными до своего завершения, и вам не придётся заботиться об их миграции или инвалидировании.

<details> 
<summary>🛠️ Пример замены workflow в контроллере</summary>
В этом примере мы видим, как просто можно направить новые вызовы на обновлённую версию:

```csharp
[HttpPost("system.clients.startUserGrantedWorkflow")]
public async Task<IActionResult> StartUserGrantedWorkflow([FromBody] StartUserGrantedWorkflowRequest request)
{
    WorkflowOptions options = new()
    {
        Id = $"UserGranted.{request.UserId}",
        RequestEagerStart = true,
        TaskQueue = TemporalTaskQueues.Clients
    };

    var args = new UserGrantedWorkflowArgsV2
    {
        UserId = request.UserId,
    };

    WorkflowHandle<UserGrantedWorkflowV2> handle = await _temporalClient
        .StartWorkflowAsync<UserGrantedWorkflowV2>(x => x.Run(args), options);

    return Json(new StartWorkflowResponse
    {
        Id = handle.Id,
        RunId = handle.RunId,
    });
}
```

</details>

> [!WARNING]  
> Важно учитывать: при дублировании workflow вы не изолируете activity, если они остались общими. Это значит, что любые изменения в логике activity, вызываемых из обоих воркфлоу, могут непреднамеренно сломать старые процессы. Именно поэтому в большинстве случаев приходится дублировать и activity.

**Когда подходит этот подход**

- Workflow маленький и автономный
- Он не вызывается другими workflow и не принимает внешние сигналы
- Вы можете чётко контролировать запуск: например, вы явно определяете, какой workflow запускать при каком условии (через версию, параметр, feature flag)

**Когда лучше НЕ использовать**

- Workflow глубоко интегрирован в другие процессы
- Он получает сигналы из внешней системы (через SignalWorkflowAsync) — менять сигнатуру сигналов может быть опасно
- Вы используете общие activity, и не готовы дублировать их

### Версионирование через task queue

Ещё один популярный подход к безопасному обновлению Temporal workflow — это разделение версий процессов по разным Task Queue. Вместо того чтобы переиспользовать одну и ту же очередь задач (task queue) для обработки всех версий workflow и activity, мы создаём отдельные очереди для каждой новой версии.

Пример:

- payments::v1 — очередь для текущей стабильной версии
- payments::v2 — очередь для обновлённого workflow, с новыми activity, таймерами и логикой

В этом случае новые worker-процессы поднимаются с конфигурацией на v2-очередь и обрабатывают только те workflow, которые на неё приходят. Старые worker продолжают обрабатывать v1, пока не завершатся все живые процессы.

<details> 
<summary>🛠️ Пример использования разных task queue</summary>
В контроллере или другом инициирующем компоненте вы можете прокинуть task queue в зависимости от условия:

```csharp
var taskQueue = request.UseNewVersion
    ? "payments::v2"
    : "payments::v1";

WorkflowOptions options = new()
{
    Id = $"ProcessPayment.{request.PaymentId}",
    TaskQueue = taskQueue
};

WorkflowHandle<IProcessPaymentWorkflow> handle = await _temporalClient
    .StartWorkflowAsync<IProcessPaymentWorkflow>(x => x.Run(args), options);
```

</details>

Task Queue — это один из ключевых механизмов изоляции в Temporal. Workflow (или activity) всегда исполняются worker’ом, подписанным на конкретную очередь. Это значит, что вы можете разнести исполняемый код по изолированным процессам, не мешая старым версиям. Такой подход не требует патчей в коде workflow и позволяет изменить:

- типы аргументов
- сигнатуру activity
- тайминги и delays
- даже структуру самих workflow (в разумных пределах)

**Когда подходит этот подход**

- У вас много долгоживущих workflow, и вы не можете «убить» старые процессы
- Вы хотите радикально поменять логику, аргументы или шаги в новом workflow
- Вы можете контролировать, с какой очередью будет запускаться процесс (например, через контроллер или очередь событий)
- Вы используете микросервисную архитектуру и можете поднять новые worker-процессы с отдельной конфигурацией

**Когда лучше НЕ использовать**

Если вы работаете в монолитной архитектуре и не можете легко разделить worker’ы

- Когда у вас десятки workflow с версиями — управлять task queues может стать хаотично
- Когда изоляция по task queue недостаточна — например, вы хотите различать не только worker-код, но и историю исполнения (см. Patched API)

**На что обратить внимание?**

- Вам придётся поддерживать две версии worker-ов в течение некоторого времени, пока не завершатся старые процессы. Это может повлиять на стоимость инфраструктуры.
- Необходимо явно указать новую task queue при запуске нового workflow
- Также нужно убедиться, что activity отправляются в ту же очередь, иначе может произойти ошибка исполнения (Activity worker not found for queue ...).

### Использование Patched API

Один из самых безопасных и поддерживаемых способов изменять Temporal workflow без остановки или дублирования — это использование встроенного API: `Workflow.Patched(...)` и `Workflow.DeprecatePatch(...)`.

Этот подход реализует концепцию soft migration, или «мягкой эволюции», при которой новый код может быть внедрён без нарушения выполнения уже запущенных процессов. Это особенно полезно, когда:

- вы хотите изменить поведение без создания новой версии workflow;
- вы не можете изменить task queue;
- вы хотите сохранить один кодовый путь, избежав дублирования классов и логики.

Когда вы вызываете `Workflow.Patched("feature-key")`, Temporal SDK сохраняет в историю специальное событие — патч-маркер. При последующем воспроизведении (replay), Temporal использует этот маркер, чтобы точно повторить путь исполнения.

Это позволяет безопасно обернуть новую логику в условие:

```csharp
if (Workflow.Patched("use-new-delay-logic"))
{
    await Workflow.DelayAsync(TimeSpan.FromSeconds(30)); // новая логика
}
else
{
    await Workflow.DelayAsync(TimeSpan.FromSeconds(10)); // старое поведение
}
```

Все workflow, созданные до добавления патча, пойдут по else. Все новые — по if. Так Temporal обеспечивает обратную совместимость.

#### Жизненный цикл Patch

Добавить `Workflow.Patched(...)` — просто. Но удалять его нужно с осторожностью, соблюдая жизненный цикл патча:

**Шаг 1. Вводим патч**

```csharp
if (Workflow.Patched("use-new-delay"))
{
    await Workflow.DelayAsync(TimeSpan.FromSeconds(30));
}
else
{
    await Workflow.DelayAsync(TimeSpan.FromSeconds(10));
}
```

**Шаг 2. Помечаем как устаревший**

После того как новые workflow стабильно работают, а старые завершились, добавьте:

```csharp
Workflow.DeprecatePatch("use-new-delay");
await Workflow.DelayAsync(TimeSpan.FromSeconds(30));
```

С этого момента новые workflow больше не получают патч-маркер , из-за чего мы можем удалить всю if-else конструкцию и упросить код, как на примере выше.

> [!WARNING]  
> Если вы удалите if-else конструкцию слишком рано, в проде вас может ждать "упс". Связано это с тем, что у вас могут остаться какие-то долгоиграющие workflow, которые были подвязаны на else-ветку, поэтому при перепроигрывании истории они так же сломаются из-за non-deterministic апдейта. Необходимо убедиться, что все workflow, которые были запущены до внедрения Patched(_version_) были завершены.

**Шаг 3. Удаляем**

Когда вы убедились, что все старые workflow завершены, а `Patched(...)` больше не используется в активных историях — можно удалить `Workflow.DeprecatePatch`.

### Версионирование воркеров (Worker Versioning)

Тут, к сожалению, мне особо нечего сказать. Этот подход всё ещё в pre-release, поэтому на практике я его не использовал. Однако, он довольно похож на task queue версионирование и, как я понимаю, разрабатывается командой temporal для того, чтобы улучшить его и ввести некоторый out of the box подход, который заменит версионирование с помощью task queue.

Будем наблюдать вместе :)

### Тестирование изменений

Остается один вопрос: как убедиться, что мы точно не нарушили deterministic flow очередным релизом ? Ведь всегда остается некоторая вероятность того, что очередное изменение что-то поломает.

Концептуально, есть несколько подходов

- тестирование workflow на dev/stage среде
- тестирование обновленных workflow с помощью тестов и workflow reply

Первый подход довольно понятен. Последовательность действий в нем такая

- запускаем старую версию workflow в нашей тейстовой среде
- доходим до определенного чекпоинта и ждем (либо сигнала, либо тушим под с workflow)
- раскатываем новую версию и наблюдаем за тем, как отработает процесс

Подход рабочий, но иногда может занимать довольно много времени (особенно, если речь идет о довольно сложных workflow с большим количеством шагов)

Второй подход более интересен, поскольку позволяет проверить наши изменения без релиза и последующих багофиксов, если все же что-то отломалось. Однако, для его реализации потребуется сформировать привычку писать workflow тесты.

Например, если мы говорим о .net, то вот так можно выполнить проверку обратной совместимости с помощью `xunit` тестов

```cs
[Theory(DisplayName = "Replay workflow with success")]
[ClassData(typeof(ReplayHistoryDataProvider))]
public async Task ReplayPaymentProcessingV2Workflow_Success(string name, WorkflowHistory history)
{
    var replayer = new WorkflowReplayer(
        new WorkflowReplayerOptions()
            .AddWorkflow<PaymentProcessingV2>());

    _ = await replayer.ReplayWorkflowAsync(history);

}

public class ReplayHistoryDataProvider : IEnumerable<object[]>
{
    private const string Namespace = "Blog.Temporal.Part2.Tests.Workflows.Histories.PaymentProcessingV2";

    public IEnumerator<object[]> GetEnumerator()
    {
        var assembly = Assembly.GetExecutingAssembly();

        IEnumerable<string> source = assembly.GetManifestResourceNames()
            .Where(x => x.StartsWith(Namespace));

        foreach (string item in assembly.GetManifestResourceNames())
        {
            if (!item.StartsWith(Namespace))
            {
                continue;
            }

            using Stream stream = assembly.GetManifestResourceStream(item);
            yield return [
                item.Replace(Namespace, string.Empty),
                WorkflowHistory.FromJson(
                    Guid.NewGuid().ToString(),
                    new StreamReader(stream).ReadToEnd())
            ];
        }
    }

    IEnumerator IEnumerable.GetEnumerator()
    {
        return GetEnumerator();
    }
}
```

Код выше демонстрирует, как можно выполнить проигрывание обновленного workflow по истории какого-либо предыдущего запуска. Это позволяет гарантировать, что новая версия не обрушит нам production.

Давайте разберем его подробнее.

Во-первых, давайте взглянем на сам тест `ReplayPaymentProcessingV2Workflow_Success`. Это довольно небольшой тестовый метод, который использует компонент `WorkflowReplayer`, поставляемый temporal sdk. Он как раз-таки и позволяет проиграть историю по коду workflow.
Сам тестовый метод является теорией, которая поддерживает запуск с множеством разных аргументов. Это необходимо для того, чтобы выполнить прогон нескольких предыдущих историй по новому коду.

Во-вторых, давайте изучим класс `ReplayHistoryDataProvider`, который служит источником данных для теории.
Этот класс так же довольно прост, он выполняет следующее

- считывает все json файлы с историей запуска workflow из сборки, которая содержит тесты
- выполняет парсинг содержимого каждого файла в тип `WorkflowHistory` и возвращает собранный тип для дальнейшей передачи его в аргументы метода

Остается лишь один вопрос: где взять историю предыдущих запусков ?

Сделать это можно так же из тестов. Перед тем, как изменять версию workflow, необходимо выполнить прогон существующих тестов и сохранить execution history. Сделать это можно довольно легко, просто дописав в коде теста следующие строки

```cs
await notificationsWorker.ExecuteAsync(async () =>
{
    WorkflowHandle<PaymentProcessingV2> runHandle =
        await env.Client.StartWorkflowAsync<PaymentProcessingV2>(x => x.RunAsync(new() { }),
            options: new WorkflowOptions
            {
                Id = id,
                TaskQueue = TaskQueueNames.PaymentProcessingV2
            });

    await runHandle.GetResultAsync();

    var json = (await runHandle.FetchHistoryAsync()).ToJson(); // новая строка
});
```

Затем можно запустить тест под дебаггером и скопировать содержимое переменной `json`, после чего это содержимое можно встроить в сборку с тестами, например, как embedded resource.

<details> 
<summary>🛠️ Пример полного workflow теста</summary>

```csharp
[Fact(DisplayName = "Execute workflow")]
public async Task ExecuteWorkflow_Success()
{
    string id = $"trx.{Guid.NewGuid()}";
    await using WorkflowEnvironment env =
        await WorkflowEnvironment.StartLocalAsync(new()
        {
            LoggerFactory = LoggerFactory,
        });

    [Activity(NotificationsActivitiesV1.Names.Notify)]
    static Task<NotificationsActivitiesV1.NotifyResult> Notify(NotificationsActivitiesV1.NotifyResult @params)
    {
        return Task.FromResult(new NotificationsActivitiesV1.NotifyResult { });
    }

    [Activity(TransactionActivitiesV2.Names.BeginAuthorization)]
    static Task<TransactionActivitiesV2.BeginAuthorizationResult> BeginAuthorization(TransactionActivitiesV2.BeginAuthorizationResult @params)
    {
        return Task.FromResult(new TransactionActivitiesV2.BeginAuthorizationResult { });
    }

    [Activity(TransactionActivitiesV2.Names.ConfirmAuthorization)]
    static Task<TransactionActivitiesV2.ConfirmAuthorizationResult> ConfirmAuthorization(TransactionActivitiesV2.ConfirmAuthorizationResult @params)
    {
        return Task.FromResult(new TransactionActivitiesV2.ConfirmAuthorizationResult { });
    }

    using TemporalWorker processingWorker = CreateWorker(TaskQueueNames.PaymentProcessingV2, cfg =>
    {
        _ = cfg.AddActivity(BeginAuthorization);
        _ = cfg.AddActivity(ConfirmAuthorization);
        _ = cfg.AddWorkflow<PaymentProcessingV2>();
    });

    using TemporalWorker notificationsWorker = CreateWorker(TaskQueueNames.NotificationsV1, cfg =>
    {
        _ = cfg.AddActivity(Notify);
    });

    await processingWorker.ExecuteAsync(async () =>
    {
        await notificationsWorker.ExecuteAsync(async () =>
        {
            WorkflowHandle<PaymentProcessingV2> runHandle =
                await env.Client.StartWorkflowAsync<PaymentProcessingV2>(x => x.RunAsync(new() { }),
                    options: new WorkflowOptions
                    {
                        Id = id,
                        TaskQueue = TaskQueueNames.PaymentProcessingV2
                    });

            await runHandle.GetResultAsync();

            var json = (await runHandle.FetchHistoryAsync()).ToJson();
        });
    });


    TemporalWorker CreateWorker(string taskQueue, Action<TemporalWorkerOptions> configure)
    {
        var opts = new TemporalWorkerOptions(taskQueue);
        configure?.Invoke(opts);
        return new TemporalWorker(env.Client, opts);
    }
}
```

</details>

Стоит учесть, что вам придется поддерживать разные сценарии выполнения workflow в своих тестах (как обычных, так и reply) для достижения наиболее полных гарантий работоспособности. Это может стать нелегкой задачей, если ваши workflow будут большими и сложными.

Помимо этого, вам придется следить за актуальностью состояний в ваших json файлах с историей, регулярно (при каждом изменении workflow) обновляя их, а так же удаляя старые версии (поскольку после выведения `DeprecatePatch` выполнение workflow через reply будет ломаться на старых версиях состояния)

## Итого

Версионирование workflow в Temporal — это не просто технический приём, а фундамент архитектурной надёжности. Без него любая мелкая правка может превратиться в продакшен-инцидент, если нарушить deterministic replay или сломать ожидания по взаимодействию между системами.

Temporal даёт большую гибкость, но требует от команды дисциплины. Правильное версионирование workflow — это не только защита от ошибок, но и способ развивать продукт с уверенностью.

В следующей статье мы спроектируем небольшой workflow, который занимается процессингом платежей, покроем его тестами и посмотрим на живом примере, как

- его можно задокументировать и описать
- применить концепции и идеи, которые были изложены в этой статье
