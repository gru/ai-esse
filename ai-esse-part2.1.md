# ИИ для разработчика. Часть 2. Сервис.

Итак, после того как наш шаблон готов, пришло время использовать его для разработки сервиса. Для начала я решил написать максимально простой сервис, который позволит загружать файлы в хранилище по протоколу S3 и отдавать нам идентификатор контента, по которому мы после сможем этот файл скачать.

Я пропущу описание того, как я добавлял пакеты и создавал классы для Entity Framework с помощью ИИ, чтобы мы могли сохранить информацию о файле в БД. Давайте сразу перейдем к реализации основной функциональности по загрузке файлов и проведем небольшое ревью того кода, который написал ИИ. Так как в решении у меня уже был класс ContentHandler с реализацией, которая появилась там после инициализации из шаблона, то я в своем промпте пишу "измени" вместо "добавь", но в общем промпт для создания бизнес-логики нашего сервиса выглядел вот так:

```
Нужно добавить в наш сервис возможность загружать контент в S3 при помощи метода CreateContent класса ContentHandler:
* Измени параметры метода CreateContent на HttpContext и CancellationToken.
* Так как наши файлы будут приходить в виде multipart-data запроса, то прочитай данные из HttpContext.Request.Body при помощи MultipartReader.
* Каждый файл отправь в S3 при помощи IAmazonS3 клиента.
* Для каждого загруженного файла создай ContentEntity и заполни поля FileName и ContentType и вставь в БД.
* Создай класс CreateContentResponse со свойством ContentIds и добавь его в AI.Storage.Http.Contracts.
* Возвращаемое значение метода CreateContent должно быть CreateContentResponse и содержать список вставленных в БД идентификаторов ContentEntity.
```

Первый вариант, который предложил ИИ, был самым очевидным. Не буду тут приводить его целиком, только фрагменты, которые показались мне интересными. Например, в первой реализации ИИ выбрал отправку файла напрямую в S3 без буферизации на клиенте. Такой вариант будет работать, но очевидно появятся проблемы с клиентами, у которых медленная сеть. 

```csharp
public async Task<List<long>> CreateContent(HttpContext httpContext, CancellationToken cancellationToken)
{
    ... 

    var reader = new MultipartReader(httpContext.Request.GetMultipartBoundary(), httpContext.Request.Body);
    var section = await reader.ReadNextSectionAsync(cancellationToken);
    
    while (section != null)
    {
        var key = $"{Guid.NewGuid()}/{fileName}";
        using (var stream = section.Body)
        {
            var putRequest = new PutObjectRequest
            {
                BucketName = _bucketName,
                Key = key,
                InputStream = stream,
                ContentType = contentType
            };

            await _s3Client.PutObjectAsync(putRequest, cancellationToken);
        }

        ...

        section = await reader.ReadNextSectionAsync(cancellationToken);
    }

    ...
}
```
Конечно, здесь сказывается отсутствие контекста в промпте, и я даже не просил его писать максимально эффективный с точки зрения реализации код, но это уже наводит нас на мысль, что необходимо следить за тем, что написал ИИ. В следующей итерации я попросил его исправить проблему и добавить буферизацию, что он формально и сделал:

```csharp
var s3Key = $"{Guid.NewGuid()}/{fileName}";
await using var memoryStream = new MemoryStream();
await section.Body.CopyToAsync(memoryStream, cancellationToken);
memoryStream.Position = 0;

var putObjectRequest = new PutObjectRequest
{
    BucketName = _bucketName,
    Key = s3Key,
    InputStream = memoryStream,
    ContentType = contentType
};

await _s3Client.PutObjectAsync(putObjectRequest, cancellationToken);
```

По факту я получил что просил, но создавать MemoryStream без параметров в конструкторе и потом копировать в него данные очень неэффективно с точки зрения работы с памятью. Чтобы этого избежать, я попросил ИИ использовать `RecyclableMemoryStream`. 

```csharp
public class ContentHandler
{
    private readonly RecyclableMemoryStreamManager _streamManager = new();

    ...

    public async Task<IEnumerable<long>> CreateContent(HttpContext httpContext, CancellationToken cancellationToken)
    {
        ...

        var s3Key = $"{Guid.NewGuid()}/{fileName}";
        using (var memoryStream = _streamManager.GetStream())
        {
            await section.Body.CopyToAsync(memoryStream, cancellationToken);
            memoryStream.Position = 0;

            var putObjectRequest = new PutObjectRequest
            {
                BucketName = _bucketName,
                Key = s3Key,
                InputStream = memoryStream,
                ContentType = contentType
            };

            await _s3Client.PutObjectAsync(putObjectRequest, cancellationToken);
        }

        ...
    }
}
```

И этот код тоже написан неверно. В [документации](https://github.com/microsoft/Microsoft.IO.RecyclableMemoryStream) написано, что использование `RecyclableMemoryStreamManager` без параметров в конструкторе не рекомендуется и может приводить к утечке памяти, просим ИИ их добавить:

![Нужно указать параметры, чтобы не было утечки памяти.](images\ai-esse-part2-image-1.PNG)

В итоге мы получили некоторые цифры в параметрах, отсутствие которых было проблемой, но наличие их тоже не является гарантией того, что они подходят для решаемой нами задачи.

```csharp
public class ContentHandler
{
    private static readonly RecyclableMemoryStreamManager _streamManager = new(
        blockSize: 1024 * 64,
        largeBufferMultiple: 1024 * 1024,
        maximumBufferSize: 1024 * 1024 * 16,
        useExponentialLargeBuffer: true,
        maximumSmallPoolFreeBytes: 1024 * 1024 * 64,
        maximumLargePoolFreeBytes: 1024 * 1024 * 128);
    
    ...
}
```

В итоге осталась последняя не такая явная ошибка: метод Dispose вызывается дважды, и чтобы этого не происходило, нужно поставить свойство `PutObjectRequest.AutoCloseStream` в `false`. Это можно было бы отловить только в случае, если бы мы подписались на событие `StreamDoubleDisposed`, но, к сожалению, ИИ не читает документацию, в которой написано о проблеме.

При этом нельзя сказать, что ИИ допустил ошибки из-за недостатка контекста. Как минимум, решение с `MemoryStream` не подходит для продуктовой среды, а реализация с использованием `RecyclableMemoryStreamManager` просто не была до конца корректной. Это все наводит на грустные мысли, что ИИ действительно может переносить ошибки из других проектов на которых он учился, и их количество в результирующем коде напрямую зависит от опыта и количества времени у программиста, который использует ИИ. Если честно, перед тем как я начал использовать ИИ в разработке, у меня была надежда на то, что ИИ принесет в мой код больше хороших подходов, чем плохих, но, как оказалось, это не так. Соотношение нормального, проблемного и хорошего кода приблизительно 50/40/10.

При этом есть задачи, с которыми ИИ справляется просто прекрасно. Например, генерация одного кода на основе другого на уровне отдельных классов и методов, создание entity классов для EF Core, миграций для них, DTO для запросов и ответов, причем сразу с комментариями, все это проходит на ура. Исходной точкой может быть просто описание на обычном русском языке в виде перечисления сущностей и их атрибутов.

Ну и иногда ИИ просто поражает своей способностью понимать, что от него хотят. В качестве примера приведу промт, в котором мне, чтобы проверить работоспособность сервиса через swagger, понадобилась кнопка для загрузки файла. Причем писать что-то конкретное с деталями мне было лень:

```
Для метода CreateContent контроллера ContentController сделай в swagger кнопку нажав по которой я могу загрузить файл. 
```

И на выходе я получил реализацию `IOperationFilter`, который ИИ сам зарегистрировал в `Startup` нашего приложения, и все заработало "из коробки". Это действительно здорово. 
   
В общем на реализацию у меня ушло около 2 часов, и где то час на последущую отладку сервиса и устранение мелких недочетов. Оцениваю это как очень хороший показатель.

На этом очередная статья могла бы закончится, но я начал смотреть как Claude может конвертировать код с одного языка на другой. По хорошему это должно быть очень простой задачей для LLM, но Claude не справился. Я предпринял несколько попыток и каждый раз получалось очень плохо, причем каждый раз получалось новое плохое решение. Он вроде верно ухватывал суть, но постоянно забывал какую то часть функционала и не мог правильно положить его на мой шаблон. Поэтому мне пришлось сделать конвертацию кода сервиса [ddproxy](https://github.com/Aleksandr-ru/ddproxy) c javascript на C# вручную. 

Чтобы это сделать я попросил ИИ описать что вообще в этом сервисе реализовано, в качестве результата получил такой вывод:

```
Архитектура:

* Проект использует Node.js для создания HTTP-сервера.
* Redis используется в качестве хранилища кеша.

Проксирование запросов:

* Сервер принимает POST-запросы на тех же URL-адресах, что и оригинальный сервис dadata.
*Запросы к dadata выполняются с использованием библиотеки request, при этом добавляется токен авторизации.

Кеширование:

* Для каждого запроса создается уникальный ключ кеша на основе URL и данных запроса.
* Перед отправкой запроса к dadata, сервер проверяет наличие данных в кеше Redis.
* Если данные найдены в кеше, они возвращаются клиенту без обращения к dadata.
* Если данных нет в кеше, выполняется запрос к dadata, результат сохраняется в кеше и возвращается клиенту.

Управление временем жизни кеша:

* Для разных типов запросов можно настроить разное время жизни кеша.
* Специальная обработка для запросов информации о юридических лицах по ИНН.

Ограничение количества запросов:

* Реализована возможность ограничения количества запросов к dadata в сутки.
* Счетчик запросов хранится в Redis и сбрасывается ежедневно.

Дополнительные функции:

* Создание дополнительных ключей кеша для удобного поиска данных (например, по ИНН).
* Валидация ИНН.

```

И решил просто пойти по шагам, на кажом шаге прося реализовать одну функциональность. В общем сам процесс не сильно отличается от написания нового сервиса, который я описывал выше. Но некоторые нюансы я хотел бы рассказать. Слоность например возникла когда я попросил его сделать лимит на количество запросов в день. При этом получил такой код:

```csharp
private async Task<bool> CheckAndIncrementDailyRequestCounter()
{
    var counterKey = $"dadata:daily_request_counter:{DateTime.UtcNow:yyyyMMdd}";
    var currentCount = await _cache.GetStringAsync(counterKey);

    int count = 0;
    if (currentCount != null)
        count = int.Parse(currentCount);

    if (count >= _cachingOptions.DailyRequestLimit)
        return false;

    count++;
    
    await _cache.SetStringAsync(counterKey, count.ToString(), new DistributedCacheEntryOptions
    {
        AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(_cachingOptions.RequestCounterExpirationHours)
    });

    return true;
}
```

Здесь просто очевидная гонка, которая может приводить к неправильному увеличению счетчика между параллельными запросами, что у меня даже возникли вопросы откуда ИИ взял этот код. Поэтому мне пришлось открыть код, который написан на JavaScript, и посмотреть, как там это реализовано. Там было правильное решение с использованием инкремента на стороне Redis. Возможно, ИИ решил, что раз мы до этого использовали IDistributedCache, то он должен использовать его и здесь. Но правильным решением был бы только отказ от него и переход на прямую работу с Redis. Хорошо, я предложил ему сделать это. Полученный код был еще более странным - там были повисшие в воздухе асинхронные методы без await. Я попросил переделать его, потом еще раз и еще, в итоге минут через 20 мы пришли к рабочему решению. Но самое забавное тут, что после каждой итерации Claude писал, что вот этот вариант точно уж соответствует коду, написанному на JavaScript, но код естественно был далек от того, что собственно мы портируем.

Кроме того, по сути Claude никогда не спорит с тем, что ему предлагают. Например, мне было лень пытаться сделать, чтобы количество запросов в день точно ограничивалось значением DailyRequestLimit и алгоритм не допускал проскальзываний, и я об этом сказал ИИ, на что он ответил:

```
Как вы верно заметили, это может привести к небольшому "проскальзыванию" времени жизни счётчика в случае, если между проверкой и инкрементированием произойдёт сброс счётчика. Однако, учитывая, что время жизни счётчика обычно устанавливается на 24 часа, а промежуток между проверкой и инкрементированием очень мал, этим можно пренебречь.
```

То есть по сути, если я принимаю решение, то оно не проходит никакую верификацию со стороны ИИ, и он продолжит имплементировать то, что я его прошу, даже если мое решение некорректно. По сути, конечно, LLM так и должны работать, но то, как хорошо LLM пишет код, не должно вводить нас в заблуждение, что он что-то знает, проверяет наш код или несет ответственность за его корректность или работоспособность, как это могло бы быть в случае с программистом-наставником.

Через пару итераций ИИ исправил код и вот что я получил:

```csharp
private async Task<bool> CheckAndIncrementDailyRequestCounter()
{
    var db = _redisConnection.GetDatabase();
    var counterKey = $"dadata:daily_request_counter:{DateTime.UtcNow:yyyyMMdd}";

    var transaction = db.CreateTransaction();
    var incrementTask = transaction.StringIncrementAsync(counterKey);
    await transaction.KeyExpireAsync(counterKey, TimeSpan.FromHours(_cachingOptions.RequestCounterExpirationHours));
    
    if (!await transaction.ExecuteAsync())
    {
        throw new Exception("Failed to increment request counter");
    }

    var newCount = await incrementTask;

    if (newCount > _cachingOptions.DailyRequestLimit)
    {
        await db.StringDecrementAsync(counterKey);
        return false;
    }

    return true;
}
```

Основная проблема заключается не в том, что код неверный, а в том, что я не могу ему доверять, как было показано выше. Чтобы разобраться с этим кодом, мне необходимо изучить документацию по Redis для понимания механизма работы транзакций. Кроме того, требуется сверить написанный код с оригинальной JavaScript-версией, чего я изначально пытался избежать.

Таким образом, ИИ не только не избавляет нас от необходимости разбираться в технологии, но даже усложняет эту задачу. Это происходит потому, что метрики, по которым он принимает решение о генерации того или иного кода, никак не связаны с его корректностью. ИИ представляет собой черный ящик, который может выбрать решение, основываясь лишь на том, какое из них требует меньше вычислительных ресурсов.

Я испытываю сочувствие к людям, которые пытаются программировать без должного опыта, бездумно применяя предложения Cursor'а. Хотя ИИ действительно способен создать код, который будет компилироваться и даже заработает после множества попыток (особенно если речь идет о небольшом решении), высока вероятность того, что такой код будет функционировать только на машине разработчика.

Но одно я могу сказать точно, одна из двух проблем в программировании инвалидации кеша, проблема названий перменных и проблема ошибок на единицу решена, ИИ может прекрасно называть переменные и методы. Во всем проекте я ни разу сам не дал ни одного имени пермеенной или методу.