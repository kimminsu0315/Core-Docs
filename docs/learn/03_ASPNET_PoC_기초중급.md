# Core 구현참고 — ASPNET PoC 샘플 코드 v1.0_d10

이 파일은 Core 프로젝트 ASP.NET Core 구현 학습용 임시 자료입니다. 본체 SAD v1.0_d118 기준의 핵심 흐름을 짧은 코드로 따라갈 수 있게 구성했습니다. 실제 프로덕션 코드가 아닌 학습 샘플입니다.

**환경 전제**: .NET 8, ASP.NET Core 8, EF Core 8, PostgreSQL.

**명명 규칙**: 백엔드 C#은 일반 PascalCase Convention (헝가리언은 WinForms Dashboard 측만).


> **기준 본체**: Core_구현참고_ASPNET_PoC_SampleCode_v1_0_d10.md (학습 원본 변경 시 사용자 명시 때만 sync)
> 이 문서는 `Core_구현참고_ASPNET_PoC_SampleCode_v1_0_d10.md` 기준으로 작성되었습니다.
> 최종 업데이트: 2026-07-04 14:21
---

## 1. 프로젝트 구조

```
Core/
├── Core.csproj
├── Program.cs
│
├── Domain/                          # 도메인 객체 (POCO)
│   ├── Unit.cs
│   ├── Slot.cs
│   ├── Recipe.cs
│   └── Events/                      # 도메인 Event
│       ├── IDomainEvent.cs
│       ├── SlotOccupiedEvent.cs
│       └── RecipeUpdatedEvent.cs
│
├── Infra/                           # 인프라 계층 (§4.6)
│   ├── InputChannel.cs              # State Service 입력 Channel
│   ├── DomainEventBus.cs            # 도메인 Event pub/sub
│   ├── CoreDbContext.cs             # EF Core
│   └── Inputs/                      # 입력 Channel Payload 타입
│       ├── IInputMessage.cs
│       ├── GmRecipeUpdate.cs
│       └── AmmrJobResult.cs
│
├── Adapters/                        # Adapter 계층 (§4.4)
│   ├── Gm/
│   │   ├── IGmAdapter.cs
│   │   ├── GmAdapter.cs             # 실 REST 호출
│   │   ├── GmAdapterMock.cs         # 테스트용 Mock
│   │   └── GmPollingWorker.cs       # BackgroundService
│   ├── Mqtt/
│   │   ├── IMqttSendPort.cs
│   │   └── MqttAdapter.cs
│   └── (SM/WIP/SignalR/CommandApi 동일 패턴)
│
└── Services/                        # 서비스 계층 (§4.5)
    ├── State/
    │   ├── IStateService.cs
    │   └── StateService.cs          # 입력 Channel 단일 Consumer
    ├── Transfer/
    │   └── TransferService.cs       # Event 구독자
    └── Ammr/
        └── AmmrService.cs
```

---

## 2. 흐름 A — 입력 흐름 (GM Polling → State → Event → Transfer)

### 2.1 도메인 객체 정의

```csharp
// Domain/Recipe.cs
namespace Core.Domain;

// immutable record — 외부 서비스가 내부 필드 변경 못하게 (§4.6 원칙)
public record Recipe(string RecipeId, string Name, string Version);
```

```csharp
// Domain/Events/IDomainEvent.cs
namespace Core.Domain.Events;

public interface IDomainEvent
{
    DateTime OccurredAt { get; }
}
```

```csharp
// Domain/Events/RecipeUpdatedEvent.cs
namespace Core.Domain.Events;

public record RecipeUpdatedEvent(Recipe Recipe, DateTime OccurredAt) : IDomainEvent;
```

```csharp
// Infra/Inputs/IInputMessage.cs — 입력 Channel Payload marker 인터페이스
namespace Core.Infra.Inputs;

public interface IInputMessage
{
    DateTime ReceivedAt { get; }
}
```

```csharp
// Infra/Inputs/GmRecipeUpdate.cs — GM Polling이 만들어 Channel에 넣는 Payload
namespace Core.Infra.Inputs;

using Core.Domain;

public record GmRecipeUpdate(Recipe Recipe, DateTime ReceivedAt) : IInputMessage;
```

### 2.2 인프라 — 입력 Channel

State Service가 단일 Consumer. 모든 Adapter가 이 Channel에 넣음 (§4.6).

```csharp
// Infra/InputChannel.cs
using System.Threading.Channels;
using Core.Infra.Inputs;

namespace Core.Infra;

public interface IInputChannel
{
    // Adapter 측 — 다수 Writer
    ValueTask WriteAsync(IInputMessage msg, CancellationToken ct = default);
    
    // State Service 측 — 단일 Reader
    ChannelReader<IInputMessage> Reader { get; }
}

public sealed class InputChannel : IInputChannel
{
    private readonly Channel<IInputMessage> _channel;

    public InputChannel()
    {
        // Unbounded — 입력 손실 없이 받음. 실 운영에서는 capacity 설정 검토.
        _channel = Channel.CreateUnbounded<IInputMessage>(new UnboundedChannelOptions
        {
            SingleReader = true,    // State Service 단일 Consumer로 설정
            SingleWriter = false    // Adapter 다수 Writer 허용
        });
    }

    public ValueTask WriteAsync(IInputMessage msg, CancellationToken ct = default)
        => _channel.Writer.WriteAsync(msg, ct);

    public ChannelReader<IInputMessage> Reader => _channel.Reader;
}
```

### 2.3 인프라 — 도메인 Event 버스

State Service가 발행, Transfer/AMMR Service가 구독.

```csharp
// Infra/DomainEventBus.cs
using System.Threading.Channels;
using Core.Domain.Events;

namespace Core.Infra;

public interface IDomainEventBus
{
    ValueTask PublishAsync(IDomainEvent evt, CancellationToken ct = default);
    ChannelReader<IDomainEvent> Subscribe();  // 구독자가 자기 reader 받음
}

public sealed class DomainEventBus : IDomainEventBus
{
    // 구독자별 Channel 생성 — 한 Event가 모든 구독자에 fan-out
    private readonly List<Channel<IDomainEvent>> _subscribers = new();
    private readonly object _lock = new();

    public async ValueTask PublishAsync(IDomainEvent evt, CancellationToken ct = default)
    {
        Channel<IDomainEvent>[] subs;
        lock (_lock) { subs = _subscribers.ToArray(); }

        foreach (var ch in subs)
            await ch.Writer.WriteAsync(evt, ct);
    }

    public ChannelReader<IDomainEvent> Subscribe()
    {
        var ch = Channel.CreateUnbounded<IDomainEvent>();
        lock (_lock) { _subscribers.Add(ch); }
        return ch.Reader;
    }
}
```

> **단순화**: 실 구현에서는 구독자 unsubscribe·예외 격리·backpressure 처리 방식 추가 필요. 이 샘플은 구조 학습용 최소 형태.

### 2.4 Adapter — IGmAdapter 인터페이스

DI로 교체할 Port (§1.1 단일 Port 구현 교체 방식).

```csharp
// Adapters/Gm/IGmAdapter.cs
using Core.Domain;

namespace Core.Adapters.Gm;

public interface IGmAdapter
{
    // GM 시스템에서 Recipe 목록 가져오기
    Task<IReadOnlyList<Recipe>> FetchRecipesAsync(CancellationToken ct);
}
```

### 2.5 Adapter — GmAdapter 실 구현

REST 호출 → 도메인 객체 변환만 담당 (§4.4 Adapter 책임 원칙).

```csharp
// Adapters/Gm/GmAdapter.cs
using System.Net.Http.Json;
using Core.Domain;

namespace Core.Adapters.Gm;

public sealed class GmAdapter : IGmAdapter
{
    private readonly HttpClient _http;
    private readonly ILogger<GmAdapter> _log;

    public GmAdapter(HttpClient http, ILogger<GmAdapter> log)
    {
        _http = http;
        _log = log;
    }

    public async Task<IReadOnlyList<Recipe>> FetchRecipesAsync(CancellationToken ct)
    {
        // 외부 시스템 응답 DTO
        var dtos = await _http.GetFromJsonAsync<List<GmRecipeDto>>(
            "/api/recipes", ct) ?? new();

        // 도메인 객체로 변환 — Adapter의 핵심 책임
        return dtos.Select(d => new Recipe(d.Id, d.Name, d.Version)).ToList();
    }

    private record GmRecipeDto(string Id, string Name, string Version);
}
```

### 2.6 Adapter — GmAdapterMock 테스트용 구현

같은 Port의 다른 구현. DI로 환경별 교체 (§1.1 단일 Port 구현 교체 방식).

```csharp
// Adapters/Gm/GmAdapterMock.cs
using Core.Domain;

namespace Core.Adapters.Gm;

public sealed class GmAdapterMock : IGmAdapter
{
    public Task<IReadOnlyList<Recipe>> FetchRecipesAsync(CancellationToken ct)
    {
        IReadOnlyList<Recipe> fake = new List<Recipe>
        {
            new("R001", "Recipe A", "v1"),
            new("R002", "Recipe B", "v1"),
        };
        return Task.FromResult(fake);
    }
}
```

### 2.7 Adapter — GmPollingWorker (BackgroundService)

Pull 방식 raw 입력을 Push 방식 입력 Channel로 정규화 (§4.4 SM Adapter 방식과 동일).

```csharp
// Adapters/Gm/GmPollingWorker.cs
using Core.Infra;
using Core.Infra.Inputs;

namespace Core.Adapters.Gm;

public sealed class GmPollingWorker : BackgroundService
{
    private readonly IGmAdapter _gm;
    private readonly IInputChannel _input;
    private readonly ILogger<GmPollingWorker> _log;
    private readonly TimeSpan _interval = TimeSpan.FromSeconds(30);

    public GmPollingWorker(IGmAdapter gm, IInputChannel input, ILogger<GmPollingWorker> log)
    {
        _gm = gm;
        _input = input;
        _log = log;
    }

    protected override async Task ExecuteAsync(CancellationToken stop)
    {
        while (!stop.IsCancellationRequested)
        {
            try
            {
                var recipes = await _gm.FetchRecipesAsync(stop);

                // 각 Recipe를 입력 Channel에 넣음 — State Service가 단일 Consumer로 처리
                foreach (var recipe in recipes)
                {
                    var msg = new GmRecipeUpdate(recipe, DateTime.UtcNow);
                    await _input.WriteAsync(msg, stop);
                }
            }
            catch (OperationCanceledException) { break; }
            catch (Exception ex)
            {
                _log.LogError(ex, "GM Polling 실패. 다음 주기에 재시도.");
                // 두절 감지 시 도메인 Event로 State Service에 보냄 (§4.4·§5.11)
                // 이 샘플은 단순화 — 실 구현에서는 두절 사실 메시지 보냄
            }

            await Task.Delay(_interval, stop);
        }
    }
}
```

### 2.8 서비스 — IStateService + StateService

핵심 구조. 입력 Channel 단일 Consumer + atomic 처리 + Event 발행.

```csharp
// Services/State/IStateService.cs
using Core.Domain;

namespace Core.Services.State;

public interface IStateService
{
    // 다른 서비스가 Pull 조회 (§4.6 Read Only 참조)
    IReadOnlyDictionary<string, Recipe> Recipes { get; }
}
```

```csharp
// Services/State/StateService.cs
using System.Collections.Concurrent;
using Core.Domain;
using Core.Domain.Events;
using Core.Infra;
using Core.Infra.Inputs;

namespace Core.Services.State;

public sealed class StateService : BackgroundService, IStateService
{
    private readonly IInputChannel _input;
    private readonly IDomainEventBus _bus;
    private readonly ILogger<StateService> _log;

    // InMemory 권위 (§4.6 ConcurrentDictionary)
    private readonly ConcurrentDictionary<string, Recipe> _recipes = new();

    // Read Only 참조 노출 — 다른 서비스가 Pull 조회 (§5장 조회 채널 원칙)
    public IReadOnlyDictionary<string, Recipe> Recipes => _recipes;

    public StateService(IInputChannel input, IDomainEventBus bus, ILogger<StateService> log)
    {
        _input = input;
        _bus = bus;
        _log = log;
    }

    // BackgroundService — 단일 Consumer 루프
    protected override async Task ExecuteAsync(CancellationToken stop)
    {
        // 입력 Channel을 FIFO 순으로 단일 Consumer가 소비
        await foreach (var msg in _input.Reader.ReadAllAsync(stop))
        {
            try
            {
                // 한 입력 = 한 atomic 단위 처리 (§1.3 순서·atomic 보존 축)
                await ProcessAsync(msg, stop);
            }
            catch (Exception ex)
            {
                _log.LogError(ex, "입력 처리 실패: {Type}", msg.GetType().Name);
                // 실 구현에서는 실패 메시지 별도 처리로 보냄
            }
        }
    }

    // 메시지 타입별 분기 — switch expression으로 처리
    private async Task ProcessAsync(IInputMessage msg, CancellationToken ct)
    {
        switch (msg)
        {
            case GmRecipeUpdate gm:
                await HandleGmRecipeAsync(gm, ct);
                break;
            // case AmmrJobResult ammr: ...
            // case WipUnitArrival wip: ...
            default:
                _log.LogWarning("미지원 입력 타입: {Type}", msg.GetType().Name);
                break;
        }
    }

    // atomic 단위 처리: InMemory 갱신 → 정합성 검사 → Event 발행
    // (실 시스템은 + 이력 DB 기록 + SignalR Push, §5.6 [수신] atomic 처리)
    private async Task HandleGmRecipeAsync(GmRecipeUpdate msg, CancellationToken ct)
    {
        // ① InMemory 갱신
        _recipes[msg.Recipe.RecipeId] = msg.Recipe;

        // ② 정합성 검사 (이 샘플은 생략, 실 시스템은 Slot 두 필드 검사 등)

        // ③ 이력 DB 기록 (이 샘플은 생략, 실 시스템은 EF Core 비동기 Worker)

        // ④ 도메인 Event 발행 — 구독자(Transfer Service 등)가 받음
        await _bus.PublishAsync(new RecipeUpdatedEvent(msg.Recipe, DateTime.UtcNow), ct);

        // ⑤ SignalR Push (이 샘플은 생략)

        _log.LogDebug("Recipe 갱신: {Id}", msg.Recipe.RecipeId);
    }
}
```

> **핵심 구조**: 이 Class가 `BackgroundService`(단일 Consumer 루프) + `IStateService`(Pull 조회 노출)를 동시에 구현. DI 등록은 한 인스턴스를 두 인터페이스로 묶음 (§2.10 Program.cs 참조).

### 2.9 서비스 — TransferService (Event 구독자)

State Service Event 구독 + 필요 시 State Service Pull 조회.

```csharp
// Services/Transfer/TransferService.cs
using Core.Domain.Events;
using Core.Infra;
using Core.Services.State;

namespace Core.Services.Transfer;

public sealed class TransferService : BackgroundService
{
    private readonly IDomainEventBus _bus;
    private readonly IStateService _state;
    private readonly ILogger<TransferService> _log;

    public TransferService(IDomainEventBus bus, IStateService state, ILogger<TransferService> log)
    {
        _bus = bus;
        _state = state;
        _log = log;
    }

    protected override async Task ExecuteAsync(CancellationToken stop)
    {
        // State Service Event 구독 — 자기 reader 받음
        var reader = _bus.Subscribe();

        await foreach (var evt in reader.ReadAllAsync(stop))
        {
            try
            {
                await HandleEventAsync(evt, stop);
            }
            catch (Exception ex)
            {
                _log.LogError(ex, "Event 처리 실패: {Type}", evt.GetType().Name);
            }
        }
    }

    private Task HandleEventAsync(IDomainEvent evt, CancellationToken ct)
    {
        switch (evt)
        {
            case RecipeUpdatedEvent r:
                _log.LogInformation("새 Recipe 수신: {Id}", r.Recipe.RecipeId);

                // 필요 시 State Service Pull 조회 — Read Only 참조로 읽음
                // (§5장 조회 채널 원칙 — Push 받은 Event + Pull로 최신 상태 확인)
                var totalRecipes = _state.Recipes.Count;
                _log.LogInformation("현재 Recipe 총 {N}개", totalRecipes);
                break;

            // case SlotOccupiedEvent slot: Transfer 생성 처리
        }
        return Task.CompletedTask;
    }
}
```

### 2.10 Program.cs — DI 등록 + 환경별 Adapter 교체

```csharp
// Program.cs
using Core.Adapters.Gm;
using Core.Infra;
using Core.Services.State;
using Core.Services.Transfer;

var builder = Host.CreateApplicationBuilder(args);

// ── 인프라 계층 ───────────────────────────────────────
builder.Services.AddSingleton<IInputChannel, InputChannel>();
builder.Services.AddSingleton<IDomainEventBus, DomainEventBus>();

// ── 서비스 계층 ───────────────────────────────────────
// StateService 한 인스턴스를 IStateService(Pull 조회) + BackgroundService(루프) 둘 다로 등록
builder.Services.AddSingleton<StateService>();
builder.Services.AddSingleton<IStateService>(p => p.GetRequiredService<StateService>());
builder.Services.AddHostedService(p => p.GetRequiredService<StateService>());

builder.Services.AddSingleton<TransferService>();
builder.Services.AddHostedService(p => p.GetRequiredService<TransferService>());

// ── Adapter 계층 — 환경별 구현 교체 (§1.1 단일 Port 구현 교체) ──
var useMock = builder.Configuration.GetValue<bool>("Adapters:Gm:UseMock");

if (useMock)
{
    builder.Services.AddSingleton<IGmAdapter, GmAdapterMock>();
}
else
{
    builder.Services.AddHttpClient<IGmAdapter, GmAdapter>(http =>
    {
        http.BaseAddress = new Uri(builder.Configuration["Gm:BaseUrl"]!);
        http.Timeout = TimeSpan.FromSeconds(10);
    });
}

builder.Services.AddHostedService<GmPollingWorker>();

// ── DB ────────────────────────────────────────────
// builder.Services.AddDbContextFactory<CoreDbContext>(opt =>
//     opt.UseNpgsql(builder.Configuration.GetConnectionString("Core")));

var host = builder.Build();
await host.RunAsync();
```

```jsonc
// appsettings.Development.json
{
  "Adapters": {
    "Gm": { "UseMock": true }     // 개발 환경은 Mock
  }
}

// appsettings.json (운영)
{
  "Adapters": {
    "Gm": { "UseMock": false }    // 운영은 진짜 GM REST
  },
  "Gm": { "BaseUrl": "http://gm.internal/" }
}
```

---

## 3. 흐름 B — 발신 흐름 ((Z) 방식, AMMR Service → State Service → MQTT Adapter)

이 흐름은 본체 d118 구조를 따름 — MQTT 발신은 State Service 단일 게이트웨이 경유 (§5.6 [발신] · §5.10).

### 3.1 Adapter — IMqttSendPort 인터페이스

State Service가 호출하는 발신 Port.

```csharp
// Adapters/Mqtt/IMqttSendPort.cs
using Core.Domain;

namespace Core.Adapters.Mqtt;

public interface IMqttSendPort
{
    // 도메인 객체 → MQTT Payload 변환·발신 (§4.4 MQTT Adapter 책임)
    Task SendJobInstructionAsync(JobInstruction instr, CancellationToken ct);
}

// 도메인 객체 (단순화)
public record JobInstruction(string AmmrId, string JobType, string TargetNodeId);
```

### 3.2 Adapter — MqttAdapter (수신·발신 양방향)

수신: 외부 메시지 → State Service 입력 Channel.
발신: State Service 발신 게이트웨이가 호출 → MQTT publish.

```csharp
// Adapters/Mqtt/MqttAdapter.cs
using Core.Domain;
using Core.Infra;
using Core.Infra.Inputs;

namespace Core.Adapters.Mqtt;

public sealed class MqttAdapter : BackgroundService, IMqttSendPort
{
    private readonly IInputChannel _input;
    private readonly ILogger<MqttAdapter> _log;
    // private readonly IMqttClient _client;  // MQTTnet 등 실 라이브러리

    public MqttAdapter(IInputChannel input, ILogger<MqttAdapter> log)
    {
        _input = input;
        _log = log;
    }

    // ── 수신 측 (BackgroundService) ───────────────────────
    protected override async Task ExecuteAsync(CancellationToken stop)
    {
        // _client.ConnectAsync(...);
        // _client.ApplicationMessageReceivedAsync += async args => {
        //     var payload = ParsePayload(args.ApplicationMessage.PayloadSegment);
        //     var msg = new AmmrJobResult(payload, DateTime.UtcNow);  // 도메인 객체 변환
        //     await _input.WriteAsync(msg, stop);                     // State Service 입력 Channel
        // };

        await Task.Delay(Timeout.Infinite, stop);
    }

    // ── 발신 측 (IMqttSendPort) ───────────────────────────
    // State Service 발신 게이트웨이가 호출 — AMMR Service가 직접 호출 안 함 ((Z) 방식)
    public Task SendJobInstructionAsync(JobInstruction instr, CancellationToken ct)
    {
        var payload = SerializeInstruction(instr);
        var topic = $"ammr/{instr.AmmrId}/cmd";
        // return _client.PublishAsync(topic, payload, ct);
        _log.LogDebug("MQTT publish: {Topic}", topic);
        return Task.CompletedTask;
    }

    private static byte[] SerializeInstruction(JobInstruction i)
        => System.Text.Json.JsonSerializer.SerializeToUtf8Bytes(i);
}
```

### 3.3 서비스 — StateService 발신 게이트웨이 메서드

(Z) 방식 핵심 구조 — atomic 단위로 이력·SignalR Push·MQTT 발신 묶음 (§5.6 [발신] atomic 구조).

```csharp
// Services/State/IStateService.cs (확장)
using Core.Adapters.Mqtt;
using Core.Domain;

namespace Core.Services.State;

public interface IStateService
{
    IReadOnlyDictionary<string, Recipe> Recipes { get; }

    // 발신 게이트웨이 — AMMR Service가 호출 ((Z) 방식)
    Task SendJobInstructionAsync(JobInstruction instr, CancellationToken ct);
}
```

```csharp
// Services/State/StateService.cs (발신 메서드 추가)
public sealed class StateService : BackgroundService, IStateService
{
    private readonly IMqttSendPort _mqttSend;
    // ... (기존 필드 + 생성자에 IMqttSendPort 추가)

    // (Z) 방식 — MQTT 발신 단일 게이트웨이
    public async Task SendJobInstructionAsync(JobInstruction instr, CancellationToken ct)
    {
        // §5.6 [발신] atomic 처리:

        // ① Job 발행 이력 기록 (실 시스템은 EF Core 비동기 Worker)
        _log.LogInformation("Job 발행 이력: {Ammr} {Type}", instr.AmmrId, instr.JobType);

        // ② SignalR Hub Push (Dashboard 실시간 반영) — 이 샘플은 생략
        // await _signalR.Clients.All.SendAsync("JobIssued", instr, ct);

        // ③ MQTT Adapter 호출 — Adapter는 Payload 변환·발신만
        await _mqttSend.SendJobInstructionAsync(instr, ct);
    }
}
```

> **핵심 구조**: 이 메서드는 단일 atomic 묶음. 발신·이력·Push가 한 흐름에 담겨 정합성 균열 지점 봉쇄. 실 시스템에서는 이 묶음을 입력 Channel Consumer 루프 안에서 호출하느냐, 별도 호출 경로로 두느냐는 코드 레벨 결정 (이 샘플은 별도 메서드로 단순화).

### 3.4 서비스 — AmmrService (발신 호출자)

AMMR 배정·Job Sequence 결정 → State Service 발신 게이트웨이 호출 (§4.5).

```csharp
// Services/Ammr/AmmrService.cs
using Core.Adapters.Mqtt;
using Core.Infra;
using Core.Services.State;

namespace Core.Services.Ammr;

public sealed class AmmrService : BackgroundService
{
    private readonly IDomainEventBus _bus;
    private readonly IStateService _state;
    private readonly ILogger<AmmrService> _log;

    public AmmrService(IDomainEventBus bus, IStateService state, ILogger<AmmrService> log)
    {
        _bus = bus;
        _state = state;
        _log = log;
    }

    protected override async Task ExecuteAsync(CancellationToken stop)
    {
        var reader = _bus.Subscribe();

        await foreach (var evt in reader.ReadAllAsync(stop))
        {
            // State Event 구독 → 운영 결정 (§4.5 AMMR Service 담당)
            // 예: TransferEvent 받으면 다음 Job 결정
        }
    }

    // 단위 지시 결정 방식 — State Service 발신 게이트웨이 호출 ((Z) 방식)
    private async Task IssueNextJobAsync(string ammrId, CancellationToken ct)
    {
        // ① InMemory Pull 조회 — 단위 지시 직전 검증 (§5.6)
        // var slot = _state.SomeSlotLookup(ammrId);
        // if (!IsValid(slot)) return;

        // ② Job 결정
        var instr = new JobInstruction(ammrId, "Pickup", "Node-A1");

        // ③ State Service 발신 게이트웨이 호출 — MQTT Adapter 직접 호출 안 함 ((Z) 방식)
        await _state.SendJobInstructionAsync(instr, ct);
    }
}
```

---

## 4. 단위 테스트 샘플

DI 교체 방식 검증 — Mock Adapter로 외부 의존성 없이 테스트.

```csharp
// Tests/StateServiceTests.cs
using Core.Adapters.Gm;
using Core.Domain.Events;
using Core.Infra;
using Core.Infra.Inputs;
using Core.Services.State;
using Microsoft.Extensions.Logging.Abstractions;
using Xunit;

public class StateServiceTests
{
    [Fact]
    public async Task GmRecipe_Input_Updates_InMemory_And_Publishes_Event()
    {
        // Arrange — 실제 인프라 구성, 외부 의존 없음
        var input = new InputChannel();
        var bus = new DomainEventBus();
        var state = new StateService(input, bus, NullLogger<StateService>.Instance);

        // Event 구독자 등록
        var reader = bus.Subscribe();

        // State Service 백그라운드 시작
        using var cts = new CancellationTokenSource();
        var task = state.StartAsync(cts.Token);

        // Act — 입력 Channel에 메시지 넣음
        var recipe = new Core.Domain.Recipe("R001", "Test", "v1");
        await input.WriteAsync(new GmRecipeUpdate(recipe, DateTime.UtcNow));

        // Assert — Event 수신 확인
        var evt = await reader.ReadAsync(cts.Token);
        Assert.IsType<RecipeUpdatedEvent>(evt);
        Assert.Equal("R001", ((RecipeUpdatedEvent)evt).Recipe.RecipeId);

        // InMemory 갱신 확인
        Assert.True(state.Recipes.ContainsKey("R001"));

        // Cleanup
        cts.Cancel();
        await state.StopAsync(CancellationToken.None);
    }
}
```

---

## 5. 핵심 구조 정리

이 샘플 코드가 보여주는 본체 구조 Mapping:

| 본체 구조 | 코드 구현 |
|---|---|
| §1.1 다Port 격리 (Adapter 6종 분리) | 폴더 구조 `Adapters/Gm/`, `Adapters/Mqtt/` 등 분리 |
| §1.1 단일 Port 구현 교체 | `IGmAdapter` + `GmAdapter` / `GmAdapterMock` DI 교체 |
| §1.3 순서·atomic 보존 (입력 직렬화) | `Channel.SingleReader=true` + State Service 단일 Consumer 루프 |
| §1.3 atomic 5단계 | `HandleGmRecipeAsync` (InMemory → 검증 → 이력 → Event → Push) |
| §4.6 InMemory 권위 (ConcurrentDictionary) | `StateService._recipes` (쓰기 State Service 한정) |
| §4.6 Read Only 참조 노출 | `IStateService.Recipes` 반환 타입 `IReadOnlyDictionary` |
| §5장 조회 채널 원칙 (Pull) | TransferService에서 `_state.Recipes.Count` 호출 |
| §5.6 [발신] 구조 | `IStateService.SendJobInstructionAsync` 단일 게이트웨이 |
| §4.4 MQTT Adapter 양방향 | `MqttAdapter` 수신(BackgroundService) + 발신(`IMqttSendPort`) 구현 |
| §4.4 Adapter 책임 (변환만) | `MqttAdapter.SendJobInstructionAsync`이 Payload 직렬화만 |

---

## 학습 진행 추천

1. **읽기**: 이 파일을 처음부터 끝까지 한 번 읽음 — 흐름 A 끝까지 따라가기.
2. **타이핑**: 빈 ASP.NET Core Worker 프로젝트 만들고 흐름 A를 직접 타이핑 (복붙 금지 — 손으로 쳐야 손맛 남음).
3. **실행**: 콘솔에 GM Polling Log + Recipe 갱신 + Transfer Service Event 수신 Log 뜨는지 확인.
4. **변경**: GmAdapterMock 응답을 바꿔보고 흐름 따라가기. Channel 용량을 Bounded로 바꿔보고 backpressure 동작 살펴보기.
5. **확장**: SM Adapter 추가 (GM과 동일 패턴). Adapter 격리 구조 직접 체감.
6. **흐름 B**: 발신 흐름 (Z) 방식 구현. atomic 묶음 직접 짜보기.

막히면 클로드한테 코드 보여주면서 "이 구조가 §X.Y랑 맞나?" 물어보세요. 이 샘플은 학습 출발점일 뿐, 실 프로젝트는 위 구조를 바탕으로 점진 구현.
