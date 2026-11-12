# Climate Control — МДК 03.02, вариант 3

Учебный модуль АИС «Умный кампус». Автор: Бойцов Даниил Антонович.
Принимает подписанные измерения температуры и влажности. Физическим климатическим оборудованием не управляет.

## Что внутри
- src/Climate.Core — DTO, FluentValidation и HMAC-протокол.
- src/Climate.Infrastructure — relational EF Core/SQLite.
- src/Climate.Api — ASP.NET Core 8: HMAC, policy, защита от повторов, лимит и контроллер.
- samples/Climate.Before — намеренно дефектная библиотека для аудита; API на неё не ссылается.
- samples/Climate.After — законченный пример рефакторинга для измерения метрик.
- tests/Climate.UnitTests — 29 xUnit-случаев, включая 2 наблюдения Legacy-дефекта.
- tests/Climate.IntegrationTests — 20 xUnit HTTP-сценариев с WebApplicationFactory и relational SQLite in-memory.
- tools/Climate.TestRunner — запуск официального xUnit AssemblyRunner с JSON-результатами.
- tools/Climate.Metrics — вызов настоящей библиотеки Microsoft.CodeAnalysis.Metrics 4.14.0.
- docs/ — требования и ТЗ.

## Требования и проверка
.NET 8 SDK и доступ к nuget.org для первого восстановления пакетов.
В Visual Studio открыть ClimateControl.sln. Для модулей API не нужен Windows Forms workload.

    dotnet restore ClimateControl.sln
    dotnet build ClimateControl.sln -c Release
    dotnet test tests/Climate.UnitTests -c Release
    dotnet test tests/Climate.IntegrationTests -c Release

В приложенном программном стенде оба набора реально исполнены официальным xUnit AssemblyRunner:

    dotnet run --project tools/Climate.TestRunner -c Release -- Climate.UnitTests artifacts/unit-results.json
    dotnet run --project tools/Climate.TestRunner -c Release -- Climate.IntegrationTests artifacts/integration-results.json

Ожидается 29 и 20 сценариев, FAILED 0, SKIPPED 0.
Два успешных B01 — подтверждение ошибки старого образца, а не проверки его корректности.
FluentAssertions используется в обоих тестовых проектах; TestRunner не заменяет xUnit собственными assert.

## Самостоятельный запуск API
Без реестра устройство не аутентифицируется. Производственных ключей в исходниках нет.
Для одного учебного устройства задать в PowerShell:

    $key = New-Object byte[] 32
    [Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($key)
    $env:Climate__Devices__0__SensorId = "DVC-001"
    $env:Climate__Devices__0__KeyId = "key-1"
    $env:Climate__Devices__0__RoomId = "A101"
    $env:Climate__Devices__0__KeyBase64 = [Convert]::ToBase64String($key)
    $env:Climate__Devices__0__CanWrite = "true"
    dotnet run --project src/Climate.Api --urls http://127.0.0.1:5080

/health доступен без подписи. POST /api/v1/telemetry требует всех заголовков HMAC;
пример построения запроса — ClimateFactory.Signed в интеграционных тестах.
Без подписи ожидается 401; при отсутствии scope либо чужом SensorId — 403.
Не публиковать локальный HTTP-стенд как промышленную службу: TLS и доверенный сетевой шлюз настраиваются отдельно.

## HMAC-контракт
UTF-8, LF между полями, без завершающего LF:
v1, HTTP-метод верхним регистром, путь, deviceId, keyId, Unix timestamp, nonce UUID N,
SHA-256 точных байтов JSON в нижнем hex. X-Signature = HMAC-SHA256 этой последовательности, hex.
X-Timestamp: ±120 секунд включительно. Повтор nonce отклоняется.

Тело: sensorId DVC-000, sequence 1..1000000000, temperatureC -40..85,
humidityPercent 0..100, measuredAt с явным смещением и возрастом от -10 минут до +30 секунд.
При самостоятельном запуске SQLite создаёт climate.db в рабочем каталоге.

## Метрики
Для повторения измерения передать корень установки .NET 8 SDK, где есть packs:

    dotnet run --project tools/Climate.Metrics -c Release -- samples/Climate.Before/LegacyClimateController.cs "C:/Program Files/dotnet" artifacts/metrics-before.json
    dotnet run --project tools/Climate.Metrics -c Release -- samples/Climate.After/RefactoredClimateController.cs "C:/Program Files/dotnet" artifacts/metrics-after.json

## Ограничения
ReplayGuard и аудит находятся в памяти одного процесса. Для перезапуска и кластера нужны общий атомарный nonce-store
и защищённая постоянная доставка аудита. HMAC не предотвращает ложные показания от уже скомпрометированного датчика.
Нагрузочные испытания, реальный TLS и аппаратная калибровка не проводились.
SQLite — реальная relational БД стенда; PostgreSQL-сервер к проекту не подключён.
