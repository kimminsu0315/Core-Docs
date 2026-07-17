# Core 구현참고 Channels 해설 v1.0_d14

이 파일은 Core 아키 본체 §4.6 인프라 계층의 Channels 두 구조(FIFO 파이프 / Event 버스)을 실제 구현 코드 수준으로 풀어낸 학습 자료다. 본체와 독립 수명이며, Core 프로젝트 zip 묶음에 동반 자료로 포함된다.


> **기준 본체**: Core_구현참고_Channels해설_v1_0_d14.md (학습 원본 변경 시 사용자 명시 때만 sync)
> 이 문서는 `Core_구현참고_Channels해설_v1_0_d14.md` 기준으로 작성되었습니다.
> 최종 업데이트: 2026-07-17 13:00
---

## 1. Channels 개요

`System.Threading.Channels`(.NET Core 3.0+)는 **producer-consumer Queue 추상**을 제공하는 표준 라이브러리. 핵심 구조:

- **FIFO 보존** — Writer가 넣은 순서대로 Reader가 꺼냄
- **비동기 backpressure** — `await WriteAsync` / `await ReadAsync`로 비동기 신호 전달
- **단일/다수 옵션** — `SingleReader` / `SingleWriter` 옵션으로 최적화 + 정합성 구조 명시
- **Bounded / Unbounded** — 용량 한계 유무

라이브러리 자체는 한 종류이나, **Writer/Reader 분포에 따라 두 구조로 갈라짐**:

| 구조 | Writer | Reader | 용도 |
|---|---|---|---|
| **FIFO 파이프** | 다수 | 단일 | 다수 입력원의 직렬화·순서 보존 |
| **Event 버스** | 단일 | 다수(fan-out 필요) | 한 발행자의 사건을 여러 구독자에 전달 |

이 시스템은 **두 구조를 모두 사용**하며, 각각 다른 곳에 들어가 있다.

---

## 2. 구조 1 — 입력 Channel (FIFO 파이프)

이 시스템 §4.6 **State Service 입력 Channel**의 구현 구조.

위상: Adapter 6종(GM/SM/General Process WIP/Restack Process WIP/MQTT/Command API) → Channel → State Service.

### 2.1 도메인 입력 객체 정의

Adapter마다 다른 외부 신호를 **공통 봉투**(sealed record 계층)로 정규화한다. Adapter 책임 원칙(§5장 공통 원칙)의 코드 구조.

```csharp
// 모든 입력의 공통 base
public abstract record StateServiceInput
{
    public DateTimeOffset ReceivedAt { get; init; } = DateTimeOffset.UtcNow;
}

// SM Adapter 입력
public record SmSensorInput(int CncId, int SlotId, bool InOccupied, bool OutOccupied)
    : StateServiceInput;

// WIP Adapter 입력 — 일반 공정 / 되담기 공통
public record WipQrInput(int WipId, int SlotId, string UnitId, bool RebundleDone)
    : StateServiceInput;

public record WipSensorInput(int WipId, int SlotId, bool Occupied)
    : StateServiceInput;

// MQTT Adapter 입력 — AMMR HW 보고
public record AmmrStateInput(int AmmrId, AmmrStatus Status, int CurrentNode)
    : StateServiceInput;

public record AmmrSlotSensorInput(int AmmrId, int SlotIndex, bool Occupied)
    : StateServiceInput;

public record AmmrJobResultInput(
    int AmmrId, int JobId, JobKind Kind, bool Success,
    string? UnitId, FailureReason? Reason) : StateServiceInput;

public record AmmrBatteryInput(int AmmrId, double Percent) : StateServiceInput;

// GM Adapter 입력
public record GmMasterDataInput(IReadOnlyList<ProductDto> Products) : StateServiceInput;

// Command API 입력 — Dashboard 사용자 명령
public record CommandInput(CommandKind Kind, string Operator, IDictionary<string, object> Payload)
    : StateServiceInput;
```

**핵심 구조**:
- 모든 Adapter 입력이 **`StateServiceInput`** 한 타입 계층으로 수렴 → 입력 Channel 단일 타입 매개변수로 처리 가능
- 각 record는 **immutable** — 채널 통과 중 변형 차단
- `ReceivedAt` 공통 필드로 입력 도착 시각 보존 (이력 DB 기록용)

### 2.2 DI 등록

`Program.cs`에서 Channel 인스턴스를 한 번 만들고, Writer/Reader를 **별도 DI 등록**한다.

```csharp
var builder = WebApplication.CreateBuilder(args);

// === 입력 Channel 생성 — 한 인스턴스 ===
var inputChannel = Channel.CreateUnbounded<StateServiceInput>(
    new UnboundedChannelOptions
    {
        SingleReader = true,    // State Service 단일 Consumer
        SingleWriter = false,   // Adapter 다수
        AllowSynchronousContinuations = false
    });

// Writer / Reader를 따로 DI 등록 — 인터페이스 분리 원칙
builder.Services.AddSingleton<ChannelWriter<StateServiceInput>>(inputChannel.Writer);
builder.Services.AddSingleton<ChannelReader<StateServiceInput>>(inputChannel.Reader);

// Adapter 등록 — Writer만 주입받음
builder.Services.AddSingleton<SmAdapter>();
builder.Services.AddSingleton<WipAdapter>();
builder.Services.AddSingleton<MqttAdapter>();
builder.Services.AddSingleton<GmAdapter>();
// CommandAPI는 Controller라 자동 등록

// State Service 등록 — Reader만 주입받음
builder.Services.AddSingleton<StateService>();
builder.Services.AddHostedService<StateServiceWorker>();   // BackgroundService

var app = builder.Build();
app.Run();
```

**Writer/Reader 분리 등록 구조의 의미**:

- Adapter 측에 `ChannelWriter<T>`만 주입 → Adapter가 **Read 권한 없음**(컴파일 타임 차단)
- State Service에 `ChannelReader<T>`만 주입 → State Service가 **Write 권한 없음**(자기 자신에게 다시 못 넣음)
- 의존성 역전 원칙(DIP) + 인터페이스 분리 원칙(ISP) 준수
- 테스트 시 mock 채널 주입 용이

### 2.3 Writer 측 — Adapter

각 Adapter는 자기 외부 프로토콜 수신 후 **도메인 객체 변환 + Channel write**만 담당한다.

```csharp
// === SM Adapter — REST Polling ===
public class SmAdapter : BackgroundService
{
    private readonly ChannelWriter<StateServiceInput> _writer;
    private readonly HttpClient _http;
    private readonly ILogger<SmAdapter> _logger;

    public SmAdapter(
        ChannelWriter<StateServiceInput> writer,
        HttpClient http,
        ILogger<SmAdapter> logger)
    {
        _writer = writer;
        _http = http;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromSeconds(5));
        while (await timer.WaitForNextTickAsync(ct))
        {
            try
            {
                var dto = await _http.GetFromJsonAsync<SmPollResponse>("/poll", ct);
                if (dto is null) continue;

                foreach (var slot in dto.Slots)
                {
                    // 외부 dto → 도메인 입력 객체 변환 (Adapter 책임 원칙)
                    var input = new SmSensorInput(
                        slot.CncId, slot.SlotId, slot.InOccupied, slot.OutOccupied);

                    // Channel write — 비교·변화 감지·Event 발행은 State Service 책임
                    await _writer.WriteAsync(input, ct);
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "SM Polling 실패");
                // 두절 감지는 별도 (§5.11 — Timeout 누적 시 두절 입력 발행)
            }
        }
    }
}

// === WIP Adapter — REST 수신/응답 ===
[ApiController]
[Route("wip")]
public class WipAdapter : ControllerBase
{
    private readonly ChannelWriter<StateServiceInput> _writer;
    private readonly IUnitInfoReplyService _reply;

    public WipAdapter(
        ChannelWriter<StateServiceInput> writer,
        IUnitInfoReplyService reply)
    {
        _writer = writer;
        _reply = reply;
    }

    [HttpPost("qr")]
    public async Task<UnitInfoResponse> OnQr(WipQrDto dto, CancellationToken ct)
    {
        var input = new WipQrInput(dto.WipId, dto.SlotId, dto.UnitId, dto.RebundleDone);
        await _writer.WriteAsync(input, ct);

        // REST 응답은 별도 — State Service 처리 완료까지 대기 후 Unit 정보 회신
        // (응답 본문에 의미 있는 도메인 데이터 — §4.4 WIP Adapter)
        return await _reply.WaitForReply(dto.RequestId, ct);
    }

    [HttpPost("sensor")]
    public async Task OnSensor(WipSensorDto dto, CancellationToken ct)
    {
        var input = new WipSensorInput(dto.WipId, dto.SlotId, dto.Occupied);
        await _writer.WriteAsync(input, ct);
    }
}
```

**핵심 구조**:
- Adapter는 `_writer.WriteAsync(input, ct)` 한 줄로 끝
- 도메인 비교·판단 일체 없음 (§5장 공통 Adapter 책임 원칙의 코드 구조)
- WIP Adapter REST 응답은 별도 채널 — 입력 Channel은 입력만 흘리고, 응답은 `IUnitInfoReplyService`가 별도 처리

### 2.4 Reader 측 — State Service BackgroundService

State Service는 **단일 Reader BackgroundService**로 입력을 한 번에 하나씩 꺼내 처리한다.

```csharp
// === State Service Worker — 단일 Consumer ===
public class StateServiceWorker : BackgroundService
{
    private readonly ChannelReader<StateServiceInput> _reader;
    private readonly StateService _state;
    private readonly ILogger<StateServiceWorker> _logger;

    public StateServiceWorker(
        ChannelReader<StateServiceInput> reader,
        StateService state,
        ILogger<StateServiceWorker> logger)
    {
        _reader = reader;
        _state = state;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        // ReadAllAsync — Channel이 닫힐 때까지 무한 루프
        await foreach (var input in _reader.ReadAllAsync(ct))
        {
            try
            {
                // === atomic 단위 시작 ===
                await _state.Process(input);
                // === atomic 단위 끝 ===
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "State Service 처리 실패: {Input}", input);
                // 이 입력 처리는 실패해도 다음 입력으로 진행 — 직렬화 유지
            }
        }
    }
}

// === State Service 본체 ===
public class StateService
{
    private readonly ConcurrentDictionary<int, SlotState> _slots;
    private readonly ConcurrentDictionary<int, AmmrState> _ammrs;
    private readonly DomainEventBus _bus;
    private readonly IHistoryWriter _history;
    private readonly ISignalRPushGateway _push;

    public async Task Process(StateServiceInput input)
    {
        // 1) InMemory 갱신
        var changes = UpdateInMemory(input);

        // 2) 정합성 검사 (§5.7)
        var integrityResults = CheckIntegrity(changes);

        // 3) 도메인 Event 발행
        foreach (var evt in DeriveDomainEvents(changes, integrityResults))
        {
            await _bus.Publish(evt);
        }

        // 4) 이력 영속화 요청 (비동기 Worker 위임 — DB 왕복 비대기)
        _history.Enqueue(BuildHistoryRecord(input, changes, integrityResults));

        // 5) Dashboard push (자기 권위 변화)
        await _push.Push(BuildSnapshot(changes));
    }
}
```

### 2.5 atomic 단위 경계

`await foreach (var input in _reader.ReadAllAsync(ct))` 한 바퀴가 곧 **atomic 단위**.

```
[입력 N 처리 시작]
  ├─ InMemory 갱신
  ├─ 정합성 검사
  ├─ 도메인 Event 발행
  ├─ 이력 영속화 요청
  └─ Dashboard push
[입력 N 처리 끝] ←── 여기까지 한 덩어리
[입력 N+1 처리 시작]  ←── 비로소 다음 입력 꺼냄
```

**보장되는 구조**:
- 입력 N 처리 도중 다른 입력 N+1이 끼어들지 못함 → §1.2 시나리오 1(Interleave 상태 역전) 방어
- Event 발행 시점에는 InMemory 갱신·정합성 검사가 **완료된 상태** → §1.2 시나리오 2(Event 발행 선행) 방어
- 다수 Adapter 입력이 한 줄로 직렬화 → §1.2 시나리오 3(다수 Writer 경합) 방어

이게 §1.3 **순서·atomic 보존 축**의 인프라 측 핵심 구현.

---

## 3. 구조 2 — 도메인 Event Channels (Event 버스)

이 시스템 §4.6 **Channels (Event 버스)**의 구현 구조.

위상: State Service → 도메인 Event → Transfer Service / AMMR Service.

### 3.1 단일 Reader 한계와 fan-out 필요성

`System.Threading.Channels`의 본질은 **Queue**다. 한 메시지는 **한 Reader가 가져가면 끝**. 동일 메시지를 여러 Reader가 받으려면 **fan-out**(메시지 복제 분배)이 필요.

이 시스템 §5.2 다소비처 분기 예:

```
CNC 작업대 Slot 해제 Event
    ├─→ Transfer Service (자기 대기 Transfer 무효화)
    └─→ AMMR Service (만재대기 AMMR 깨우기)
```

같은 Event를 둘이 모두 받아야 함. Channel 한 개로는 안 됨 — fan-out 필요.

---

## 4. Event 버스 구현 갈래

세 갈래로 갈리며, 이 시스템은 **갈래 (b)**를 메인으로 한다.

### 4.1 갈래 (a) — 종류별 Channel + 단일 Reader (한계)

Event 종류마다 Channel 하나, 한 Reader만. 즉 **다소비처 구조 표현 불가**.

```csharp
var cncEnd = Channel.CreateUnbounded<OutboundSlotSensorOnEvent>();
builder.Services.AddSingleton(cncEnd.Writer);
builder.Services.AddSingleton(cncEnd.Reader);

// Transfer Service만 구독 가능 — AMMR Service는 받을 수 없음
public class TransferServiceWorker : BackgroundService
{
    public TransferServiceWorker(ChannelReader<OutboundSlotSensorOnEvent> reader) { ... }
}
```

**한계**: 이 시스템 §5 다소비처 분기 결과 충돌. 이 시스템에서는 부적합.

### 4.2 갈래 (b) — Dispatcher + 구독자별 Channel (메인)

발행자(State Service)는 Dispatcher에 **한 번 발행**. Dispatcher가 등록된 모든 구독자별 Channel에 fan-out. 각 구독자는 자기 Channel을 단일 Reader로 읽음.

```csharp
// === 도메인 Event base ===
public abstract record DomainEvent
{
    public DateTimeOffset OccurredAt { get; init; } = DateTimeOffset.UtcNow;
    public Guid EventId { get; init; } = Guid.NewGuid();
}

// === Dispatcher ===
public class DomainEventBus
{
    // Type → 등록된 구독자 ChannelWriter 목록
    private readonly Dictionary<Type, List<object>> _writers = new();
    private readonly object _lock = new();

    public void Subscribe<T>(ChannelWriter<T> writer) where T : DomainEvent
    {
        lock (_lock)
        {
            if (!_writers.TryGetValue(typeof(T), out var list))
                _writers[typeof(T)] = list = new();
            list.Add(writer);
        }
    }

    public async Task Publish<T>(T evt) where T : DomainEvent
    {
        List<object>? snapshot;
        lock (_lock)
        {
            if (!_writers.TryGetValue(typeof(T), out var list)) return;
            snapshot = new List<object>(list);   // 발행 중 변경 차단
        }

        // 모든 구독자에 fan-out
        foreach (var w in snapshot)
        {
            await ((ChannelWriter<T>)w).WriteAsync(evt);
        }
    }
}

// === DI 등록 ===
var bus = new DomainEventBus();
builder.Services.AddSingleton(bus);

// 구독자별 Channel 생성 + Dispatcher 등록
// 같은 Event 타입에 두 구독자가 등록 → fan-out 방식
var transferSlotRel = Channel.CreateUnbounded<SlotReleasedEvent>(
    new UnboundedChannelOptions { SingleReader = true });
var ammrSlotRel = Channel.CreateUnbounded<SlotReleasedEvent>(
    new UnboundedChannelOptions { SingleReader = true });

bus.Subscribe(transferSlotRel.Writer);
bus.Subscribe(ammrSlotRel.Writer);

// Reader는 keyed DI로 주입 (.NET 8+)
builder.Services.AddKeyedSingleton<ChannelReader<SlotReleasedEvent>>(
    "transfer", transferSlotRel.Reader);
builder.Services.AddKeyedSingleton<ChannelReader<SlotReleasedEvent>>(
    "ammr", ammrSlotRel.Reader);

// === Transfer Service Worker ===
public class TransferServiceWorker : BackgroundService
{
    private readonly ChannelReader<SlotReleasedEvent> _slotRel;
    private readonly ChannelReader<OutboundSlotSensorOnEvent> _cncEnd;
    private readonly TransferService _service;

    public TransferServiceWorker(
        [FromKeyedServices("transfer")] ChannelReader<SlotReleasedEvent> slotRel,
        [FromKeyedServices("transfer")] ChannelReader<OutboundSlotSensorOnEvent> cncEnd,
        TransferService service)
    {
        _slotRel = slotRel;
        _cncEnd = cncEnd;
        _service = service;
    }

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        // 두 Channel 동시 구독 — Task.WhenAll로 병렬 루프
        var t1 = ReadLoop(_slotRel, _service.OnSlotReleased, ct);
        var t2 = ReadLoop(_cncEnd, _service.OnOutboundSlotSensorOn, ct);
        await Task.WhenAll(t1, t2);
    }

    private static async Task ReadLoop<T>(
        ChannelReader<T> reader, Func<T, Task> handler, CancellationToken ct)
    {
        await foreach (var evt in reader.ReadAllAsync(ct))
            await handler(evt);
    }
}

// === AMMR Service Worker — 같은 SlotReleasedEvent 구독 ===
public class AmmrServiceWorker : BackgroundService
{
    public AmmrServiceWorker(
        [FromKeyedServices("ammr")] ChannelReader<SlotReleasedEvent> slotRel,
        ...)
    { ... }
}
```

**장점**:
- 진정한 pub/sub 구조
- 각 구독자가 자기 처리 속도로 소비(backpressure 독립)
- 한 구독자 장애가 다른 구독자에 전이 안 됨
- 단일 Reader 보존 → 구독자 측 처리도 직렬화

**단점**:
- 구독자 등록 boilerplate
- keyed DI 또는 별도 wrapper 필요

### 4.3 갈래 (c) — MediatR (대안)

[MediatR](https://github.com/jbogard/MediatR) 라이브러리는 in-process pub/sub을 제공.

```csharp
// === 발행 ===
public class StateService
{
    private readonly IMediator _mediator;

    public async Task Process(StateServiceInput input)
    {
        // ... atomic 처리 ...
        await _mediator.Publish(new SlotReleasedEvent(slotId, SlotKind.Cnc));
    }
}

// === 구독 ===
public class TransferServiceSlotReleasedHandler : INotificationHandler<SlotReleasedEvent>
{
    public Task Handle(SlotReleasedEvent evt, CancellationToken ct)
    {
        // 자기 대기 Transfer 무효화
        return Task.CompletedTask;
    }
}

public class AmmrServiceSlotReleasedHandler : INotificationHandler<SlotReleasedEvent>
{
    public Task Handle(SlotReleasedEvent evt, CancellationToken ct)
    {
        // 만재대기 AMMR 깨우기
        return Task.CompletedTask;
    }
}
```

**갈래 (c)의 구조**:
- **동기 in-process** — `Publish` 호출이 모든 Handler 완료까지 await
- Handler 한 개가 느리면 전체 발행 지연
- 비동기 fan-out 구조 아님 — 갈래 (b)와 본질이 다름

이 시스템처럼 **구독자 처리를 발행자와 분리**해 atomic 단위 경계를 보호해야 하는 구조에는 갈래 (b)가 더 정합. MediatR는 코드 간결성이 강점이나, 발행자(State Service) atomic 단위 안에서 구독자 처리까지 대기하면 atomic 시간이 길어져 이 시스템 구조와 어긋남.

---

## 5. Event 생성·발행 상세

### 5.1 도메인 Event 정의

Event는 **immutable record**로 정의. 이 시스템 §5에 등장하는 Event 종류:

```csharp
// === Slot 관련 ===
public record SlotOccupiedEvent(int WipId, int SlotId, SlotKind Kind) : DomainEvent;
public record SlotReleasedEvent(int WipId, int SlotId, SlotKind Kind) : DomainEvent;

public record QrRecognizedEvent(
    int WipId, int SlotId, string UnitId, QrRole Role)   // 입고/출고
    : DomainEvent;

public record InboundSlotSensorOnEvent(int CncId, int SlotId, string UnitId) : DomainEvent;
public record OutboundSlotSensorOnEvent(int CncId, int SlotId, string UnitId) : DomainEvent;

public record SlotBlockedEvent(
    int WipId, int SlotId, SlotKind Kind, BlockReason Reason)
    : DomainEvent;
public record SlotUnblockedEvent(
    int WipId, int SlotId, SlotKind Kind, UnblockMethod Method)
    : DomainEvent;

// === AMMR 관련 ===
public record AmmrStateChangedEvent(
    int AmmrId, AmmrStatus OldStatus, AmmrStatus NewStatus, int CurrentNode)
    : DomainEvent;

public record AmmrSlotOccupiedEvent(int AmmrId, int SlotIndex) : DomainEvent;
public record AmmrSlotReleasedEvent(int AmmrId, int SlotIndex) : DomainEvent;

public record JobResultEvent(
    int AmmrId, int JobId, JobKind Kind, bool Success,
    string? UnitId, FailureReason? Reason)
    : DomainEvent;

public record BatteryThresholdEvent(
    int AmmrId, double Percent, ThresholdKind Kind)   // 25/75% (저전력 25 / 충전 중 대기 75 — 본체 §5.5)
    : DomainEvent;

// === Transfer 관련 — Transfer Service가 발행 ===
public record TransferEvent(int TransferId, TransferAction Action) : DomainEvent;
```

**설계 구조**:
- 모든 Event는 `DomainEvent` base 상속 → `OccurredAt`·`EventId` 공통
- record라 immutable + 값 비교 자동
- payload는 **Event 발생 시점의 결정 정보만 포함** — 구독자가 추가 조회 필요 시 State Service Pull(§5장 공통 조회 채널 원칙)
- 연속값(Battery)은 임계값 통과 시점에만 발행 — 매 tick 발행 금지(§5장 공통 도메인 Event 발행 원칙)

### 5.2 발행 시점 — atomic 처리 끝 구조

발행은 **InMemory 갱신·정합성 검사가 완료된 직후, atomic 단위 경계 내**에서.

```csharp
public class StateService
{
    public async Task Process(StateServiceInput input)
    {
        // === atomic 단위 시작 ===

        // 1) InMemory 갱신 — 권위 상태 갱신
        var changes = UpdateInMemory(input);
        // 예: WIP Slot 센서 ON 입력 → _slots[slotId] = state with { SensorOn = true }

        // 2) 정합성 검사 — Block 마킹/해제 결정
        var integrity = CheckIntegrity(changes);

        // 3) 도메인 Event 도출
        var events = DeriveDomainEvents(input, changes, integrity);

        // 4) 발행 — 이 시점에 InMemory는 이미 최신 상태
        //    구독자가 Event 수신 후 Pull 조회 시 모든 갱신이 반영된 상태 관찰
        foreach (var evt in events)
        {
            await _bus.Publish(evt);
        }

        // 5) 이력 영속화 Queue 적재 (비동기 Worker 위임)
        _history.Enqueue(BuildHistoryRecord(input, changes, integrity));

        // 6) Dashboard push (자기 권위 변화 + 이력)
        await _push.Push(BuildSnapshot(changes, events));

        // === atomic 단위 끝 ===
    }

    private IReadOnlyList<DomainEvent> DeriveDomainEvents(
        StateServiceInput input, IReadOnlyList<StateChange> changes, IntegrityResult integrity)
    {
        var events = new List<DomainEvent>();

        // 입력 종류별 도메인 Event Mapping
        switch (input)
        {
            case WipQrInput qr when qr.RebundleDone == false:
                events.Add(new QrRecognizedEvent(qr.WipId, qr.SlotId, qr.UnitId, QrRole.Out));
                break;

            case SmSensorInput sm:
                // 4상태 Mapping 결과에 따른 Event
                var transition = MapCncTransition(sm, changes);
                events.AddRange(transition.ToEvents());
                break;

            case AmmrJobResultInput job when job.Success && job.Kind == JobKind.Pickup:
                events.Add(new JobResultEvent(
                    job.AmmrId, job.JobId, job.Kind, job.Success, job.UnitId, null));
                events.Add(new AmmrSlotOccupiedEvent(job.AmmrId, FindAmmrSlot(job)));
                break;

            // ... 기타 입력 종류
        }

        // 정합성 검사 결과 Event
        foreach (var blocked in integrity.NewlyBlocked)
            events.Add(new SlotBlockedEvent(blocked.WipId, blocked.SlotId, blocked.Kind, blocked.Reason));

        foreach (var unblocked in integrity.NewlyUnblocked)
            events.Add(new SlotUnblockedEvent(unblocked.WipId, unblocked.SlotId, unblocked.Kind, UnblockMethod.Auto));

        return events;
    }
}
```

**핵심 구조**:
- 발행은 **갱신 후·이력·push 전** 위치
- atomic 단위 안에서 발행이 완료되어야 다음 입력 처리 시 구독자의 Event 처리 구조가 정렬됨
- 발행 자체는 빠름 — `_bus.Publish`가 Dispatcher로 fan-out하는 구조에서 각 구독자 Channel에 `WriteAsync`만 함, 구독자 측 Handler 실행은 비동기 (갈래 b)

### 5.3 payload 설계 원칙

이 시스템 §5장 공통 구조로 정리:

**1. 결정 시점의 결과만 포함**
```csharp
// OK — Pickup 성공 결과 자체
public record JobResultEvent(int AmmrId, int JobId, JobKind Kind, bool Success, ...);

// 부적절 — 결정 근거 raw 데이터 포함
public record JobResultEvent(int AmmrId, AmmrRawSensorData[] SensorHistory, ...);
```

**2. 구독자가 다시 Pull해야 할 정보는 포함 안 함**
```csharp
// OK — Slot 식별자만
public record SlotReleasedEvent(int WipId, int SlotId, SlotKind Kind);
// 구독자가 필요시 State Service Pull로 현재 Slot 상태 조회 (§5.6 조회 채널 원칙)

// 부적절 — Slot 전체 Snapshot 포함
public record SlotReleasedEvent(int WipId, int SlotId, SlotState FullSnapshot);
// 시점 차로 stale 위험
```

**3. 권위 갱신 결과만 발행, 추정·중간값 발행 금지**
```csharp
// OK — Pickup 성공으로 적재 추정값 갱신된 결과
public record AmmrSlotOccupiedEvent(int AmmrId, int SlotIndex);

// 부적절 — Pickup 시도 중 중간 상태
public record AmmrPickupAttemptingEvent(int AmmrId, int SlotIndex);
// AMMR HW 책임 영역, Core가 본 것 아님
```

### 5.4 발행 순서 보존 구조

발행 순서는 다음 두 구조로 보존된다:

**1. atomic 단위 안에서의 순서**

`DeriveDomainEvents`가 list로 반환 + foreach로 순차 `WriteAsync` → 한 atomic 안의 Event 순서 보장.

```csharp
foreach (var evt in events)
    await _bus.Publish(evt);
```

**2. atomic 단위 간의 순서**

입력 Channel이 단일 Reader라 atomic 단위가 직렬 실행 → atomic 결과인 Event도 직렬 발행 → 구독자 측 Channel에도 순서대로 도착.

따라서 **입력 도착 순서 = atomic 처리 순서 = Event 발행 순서 = 구독자 수신 순서**가 자연 보장. §1.3 순서·atomic 보존 축의 핵심 구조.

### 5.5 구독자 측 처리 패턴

구독자(Transfer/AMMR Service)는 자기 Channel Reader로 한 번에 하나씩 처리.

```csharp
public class TransferService
{
    private readonly List<Transfer> _transferList;   // 공유 TransferList
    private readonly IStateServicePullApi _statePull;
    private readonly object _lock = new();

    // SlotReleasedEvent 구독자 — 자기 대기 Transfer 무효화
    public async Task OnSlotReleased(SlotReleasedEvent evt)
    {
        // 시점 검증 — Pull로 현재 Slot 상태 재조회 (§5장 조회 채널 원칙)
        var currentState = _statePull.GetSlot(evt.WipId, evt.SlotId);
        if (currentState.SensorOn)
            return;   // 이미 다시 점유 — 무효화 불필요

        // TransferList에서 해당 출발 Slot 대기 Transfer 찾아 무효화
        lock (_lock)
        {
            var target = _transferList
                .FirstOrDefault(t => t.SourceSlot == (evt.WipId, evt.SlotId)
                                  && t.Status == TransferStatus.Pending);
            if (target != null)
            {
                target.Invalidate();
                _transferList.Remove(target);
            }
        }

        // State Service에 무효화 사실 전달 (이력 DB 기록)
        await _statePull.RecordTransferInvalidated(target);
    }

    // OutboundSlotSensorOnEvent 구독자 — Transfer 생성
    public async Task OnOutboundSlotSensorOn(OutboundSlotSensorOnEvent evt)
    {
        // Pickup 시점 검증은 6.6 단위 지시 직전이고, 이 단계는 Transfer 생성만
        var transfer = new Transfer(
            sourceSlot: (evt.CncId, evt.SlotId),
            unitId: evt.UnitId,
            targetSlot: ResolveTargetSlot(evt.UnitId));

        lock (_lock)
        {
            // 멱등성 — 동일 Unit 미수행 Transfer 존재 시 skip (§5.3)
            if (_transferList.Any(t => t.UnitId == evt.UnitId
                                    && t.Status == TransferStatus.Pending))
            {
                _ = _statePull.RecordTransferSkipped(evt.UnitId);
                return;
            }

            _transferList.Add(transfer);
            SortByPriority();
        }

        // AMMR Service에 신규 Transfer Event 발행
        await _bus.Publish(new TransferEvent(transfer.Id, TransferAction.Created));

        // State Service에 Transfer 생성 사실 전달 (이력 DB 기록)
        await _statePull.RecordTransferCreated(transfer);
    }
}
```

**핵심 구조**:
- 구독자는 **Event 수신 즉시 Pull로 시점 검증** (§5.6 조회 채널 원칙)
- 자기 권위 자료구조(TransferList) 변경은 lock으로 보호
- 자기 권위 변경 결과를 State Service에 전달 → 이력 DB 기록 + Dashboard push 게이트웨이 호출

---

## 6. 두 구조 비교 정리표

| 항목 | 입력 Channel (FIFO 파이프) | 도메인 Event Channels (Event 버스) |
|---|---|---|
| 라이브러리 | `System.Threading.Channels` 단일 인스턴스 | `System.Threading.Channels` + Dispatcher (갈래 b) |
| 위상 | Adapter → State Service | State Service → Transfer/AMMR Service |
| Writer | 다수 (Adapter 6종) | 단일 (State Service) |
| Reader | **단일** (State Service Worker) | **다수** (각 구독자 별 단일) |
| 본질 구조 | 입력 직렬화·순서 보존·atomic 단위 경계 | pub/sub fan-out·구독자 분리 |
| §1.3 3축 Mapping | 순서·atomic 보존 축 핵심 인프라 | 순서·atomic 보존 축 보조 (Event 발행 순서 구조) |
| §1.2 시나리오 방어 | 1·2·3 (Interleave / 발행 선행 / Writer 경합) | 2 (Event 발행 순서) 보조 |
| DI 등록 | Writer/Reader 각각 등록, 인스턴스 1개 | Dispatcher + 구독자별 Channel 등록 |
| 이 시스템 위치 | §4.6 State Service 입력 Channel | §4.6 Channels (Event 버스) |

---

## 7. 이 시스템 §5 다소비처 분기 코드 Mapping

§5 본문의 "다소비처 분기" 표가 코드로 어떻게 들어가는지 Mapping.

### 7.1 §5.2 SM Polling — CNC 출고 Slot Sensor On Event

본문 구조 (본체 §5.2 방식):

| Event                       | 소비자                  | 처리                              |
|------------------------------|-------------------------|-----------------------------------|
| 출고 Slot Sensor On Event    | State Service 자체      | Unit 출고 시간 기록                |
| 출고 Slot Sensor On Event    | Transfer Service (Push) | Transfer 생성, 우선순위 정렬       |

> CNC 출고 감지는 입고 Slot Sensor / 출고 Slot Sensor를 각각 별도 Sensor On/Off로 다룬다. (본체 §5.2)

코드 Mapping:

```csharp
// === State Service — atomic 단위 안에서 자기 처리 + Event 발행 ===
private async Task ProcessCncSlotSensorChange(SmSensorInput sm)
{
    // 입고 Slot On / 출고 Slot On / Off 분기
    if (sm.SensorType == SensorType.Outbound && sm.State == SensorState.On)
    {
        var unit = _units[GetUnitIdAt(sm.CncId, sm.SlotId)];

        // (a) State Service 자체 처리 — 같은 atomic 단위 내 함수 호출
        //     "도메인 Event 발행 원칙" — 같은 서비스 내 후속은 함수 호출 (§5장 공통)
        unit.OutTime = DateTimeOffset.UtcNow;
        _history.Enqueue(new UnitOutTimeRecord(unit.Id, unit.OutTime));

        // (b) 외부 서비스 알림 — 도메인 Event로 발행
        var evt = new OutboundSlotSensorOnEvent(sm.CncId, sm.SlotId, unit.Id);
        await _bus.Publish(evt);
    }
}

// === Transfer Service — Event 구독자 ===
public async Task OnOutboundSlotSensorOn(OutboundSlotSensorOnEvent evt)
{
    // 멱등 — 동일 Unit 미수행 Transfer 존재 시 skip (§5.3)
    // 위 5.5 코드 방식
}
```

핵심: 같은 서비스 내 처리는 **함수 호출**, 서비스 경계 넘는 처리는 **도메인 Event**. §5장 공통 도메인 Event 발행 원칙의 코드 구조.

### 7.2 §5.3 WIP 출고 Slot QR 인식 Event

본문 구조:

| Event | 소비자 | 처리 |
|---|---|---|
| 출고 Slot QR 인식 Event | State Service 자체 | Unit 출고 시간 기록 |
| 출고 Slot QR 인식 Event | Transfer Service (Push) | Transfer 생성(멱등) |

코드 Mapping:

```csharp
private async Task ProcessWipQr(WipQrInput qr)
{
    // (a) 자기 처리
    var unit = _units[qr.UnitId];
    if (qr.Role == QrRole.Out)
    {
        unit.OutTime = DateTimeOffset.UtcNow;
        _history.Enqueue(new UnitOutTimeRecord(unit.Id, unit.OutTime));

        // CNC WIP 추가 — Tray 형태 갱신
        if (qr.RebundleDone)
        {
            unit.TrayShape = TrayShape.Stacked;
            _history.Enqueue(new TrayShapeChangedRecord(unit.Id, unit.TrayShape));
        }
    }

    // (b) 외부 알림
    await _bus.Publish(new QrRecognizedEvent(qr.WipId, qr.SlotId, qr.UnitId, qr.Role));
}

// Transfer Service 측은 5.5 OnOutboundSlotSensorOn와 같은 멱등 방식 적용
```

### 7.3 §5.5 AMMR Job 수행 결과 Event

본문 구조 (다소비처):

| Event | 소비자 | 처리 |
|---|---|---|
| Job 수행 결과 Event | AMMR Service | 다음 단위 지시 / 완료/실패 |
| Job 수행 결과 Event | State Service 자체 | Job 이력 DB 기록 |
| Job 수행 결과 Event | State Service 자체 | Unit 위치 변경 이력 DB 기록 |

코드 Mapping:

```csharp
private async Task ProcessJobResult(AmmrJobResultInput job)
{
    // (a) State Service 자체 처리 — InMemory 갱신
    if (job.Success)
    {
        switch (job.Kind)
        {
            case JobKind.Pickup:
                _ammrs[job.AmmrId].LoadEstimate[FindFreeSlot(job.AmmrId)] = job.UnitId;
                _units[job.UnitId!].Location = new AmmrSlotLocation(job.AmmrId, ...);
                break;
            case JobKind.Dropoff:
                var ammrSlot = FindLoadedSlot(job.AmmrId, job.UnitId!);
                _ammrs[job.AmmrId].LoadEstimate[ammrSlot] = null;
                _units[job.UnitId!].Location = ResolveDropoffLocation(job);
                break;
        }
    }

    // (b) 자체 처리 — 이력 DB Queue 적재 (Job 이력 + Unit 위치 변경 이력)
    _history.Enqueue(new JobResultRecord(job.AmmrId, job.JobId, job.Kind, job.Success));
    if (job.Success && (job.Kind == JobKind.Pickup || job.Kind == JobKind.Dropoff))
        _history.Enqueue(new UnitLocationChangedRecord(job.UnitId!, ...));

    // (c) 외부 알림 — AMMR Service
    var evt = new JobResultEvent(
        job.AmmrId, job.JobId, job.Kind, job.Success, job.UnitId, job.Reason);
    await _bus.Publish(evt);
}

// === AMMR Service ===
public async Task OnJobResult(JobResultEvent evt)
{
    var ammr = _ammrList.Find(evt.AmmrId);

    if (evt.Success)
    {
        // 다음 Job 진행 (§5.6 결과 수신 처리 표)
        var next = _jobSequencer.Next(ammr, evt);
        if (next != null)
        {
            // Pickup 직전 검증 — State Service Pull (§5.6 조회 채널 원칙)
            var ok = await VerifyBeforeJob(next);
            if (ok)
                await _mqttGateway.Publish(next);
            else
                await HandleVerificationFail(next);
        }
        else
        {
            // Transfer 완료
            await _statePull.RecordTransferCompleted(ammr.CurrentTransfer!);
            ammr.CurrentTransfer = null;
            _ammrList.Return(ammr);   // AmmrList 복귀
        }
    }
    else
    {
        // 실패 처리 — §5.6 결과 수신 처리 표 분기
        await HandleJobFailure(evt);
    }
}
```

**핵심 구조**:
- "State Service 자체"는 **함수 호출**(같은 atomic 단위 내)
- "AMMR Service (Push)"는 **도메인 Event 발행**(서비스 경계)
- 두 소비처가 한 Event(`JobResultEvent`)를 받으나, **State Service 자체 구조는 이미 atomic 안에서 처리되었고 외부로 발행하는 Event는 외부 서비스용** — fan-out은 외부 구독자 사이의 구조

이 구조로 §5.5 다소비처 표의 "State Service 자체" 항목은 사실 **Event 발행 전에 이미 atomic 안에서 처리됨**. 본문이 "State Service 자체"를 다소비처 표에 담은 것은 구조의 가시화이며, 코드 구조에서는 자기 처리는 함수 호출 구조.

(끝)
