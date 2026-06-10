# CAN DBC 解析模块

## 一、整改后总体架构

### 1.1 新架构数据流

```text
RX Path:

CAN Bus
  |
  v
ICanReceiver.Receive(timeout)
  |
  v
CanFrame
  |
  v
MessageResolver
  |  - Classic CAN: CanMessageKey
  |  - CAN FD     : CanMessageKey + length
  |  - J1939      : PGN / SA / DA
  v
FrameDecoder
  |
  +--> BitCodec
  +--> PhysicalCodec
  +--> MultiplexResolver
  |
  v
DecodedFrame
  |
  v
SignalRepository.Update(RuntimeSignalValue)
  |
  v
Subscriber / Application / UI / Logger
```

```text
TX Path:

Application / Control Logic
  |
  v
TxSignalStore.SetSignal(SignalId, value)
  |
  v
TxMessageSnapshot
  |
  v
FrameEncoder.BuildFrame(MessageKey)
  |
  +--> DefaultValueResolver
  +--> MultiplexResolver
  +--> PhysicalCodec
  +--> BitCodec
  |
  v
CanFrame
  |
  v
PeriodicScheduler / Application Send
  |
  v
ICanSender.Send(frame, timeout)
  |
  v
CAN Bus
```

### 1.2 为什么要引入 `MessageResolver`

原设计由 `FrameDecoder` 直接调用：

```cpp
FindMessageById(frame.id)
```

整改后改成：

```cpp
Result<const Message*> MessageResolver::Resolve(const CanFrame& frame) const;
```

原因：

1. Classic CAN、Extended CAN、CAN FD、J1939 的匹配规则不同。
2. J1939 不能简单按完整 29-bit ID 匹配，因为 Source Address 可能变化。
3. DBC 中 Extended ID 可能带 `0x80000000` 标记，需要加载时 normalize。
4. Decoder 不应该承担协议 ID 解析细节，否则后续扩展 J1939 TP、UDS、CAN FD 会污染 Decoder。

---

## 二、核心类型整改设计

### 2.1 CAN 报文 key

```cpp
namespace can_dbc {

struct CanMessageKey {
    FrameFormat format {FrameFormat::Standard11Bit};
    std::uint32_t id {};   // normalized 11-bit or 29-bit CAN ID

    friend bool operator==(const CanMessageKey&, const CanMessageKey&) = default;
};

struct CanMessageKeyHash {
    std::size_t operator()(const CanMessageKey& key) const noexcept {
        const auto fmt = static_cast<std::uint32_t>(key.format);
        return std::hash<std::uint32_t>{}((fmt << 29U) ^ key.id);
    }
};

}
```

修改原因：

- 原来只用 `uint32_t id`，无法表达 Standard / Extended。
- DBC 中 11-bit ID `0x123` 和 29-bit ID `0x123` 在数值上相同，但语义不同。
- Vector DBC 常用最高位标记扩展帧，必须 normalize 后保存。

DBC 加载时处理：

```cpp
static constexpr std::uint32_t DbcExtendedFrameFlag = 0x80000000U;
static constexpr std::uint32_t CanExtendedMask      = 0x1FFFFFFFU;
static constexpr std::uint32_t CanStandardMask      = 0x7FFU;

Result<CanMessageKey> NormalizeDbcMessageId(std::uint32_t dbcId) {
    if ((dbcId & DbcExtendedFrameFlag) != 0U) {
        return CanMessageKey{
            .format = FrameFormat::Extended29Bit,
            .id = dbcId & CanExtendedMask
        };
    }

    if (dbcId <= CanStandardMask) {
        return CanMessageKey{
            .format = FrameFormat::Standard11Bit,
            .id = dbcId
        };
    }

    // 可配置策略：严格模式下报错；兼容模式下视为 29-bit。
    return Error{ErrorCode::SemanticError, "DBC message id is not normalized"};
}
```

---

### 2.2 CAN FD frame 模型

```cpp
struct CanFrame {
    CanMessageKey key;
    FrameType type {FrameType::Data};

    bool isFd {false};
    bool brs {false};
    bool esi {false};

    std::uint8_t dlcCode {};  // 0..15，硬件/总线层 DLC code
    std::uint8_t length {};   // 0..64，真实 payload 字节数

    std::array<std::byte, 64> data {};
    std::chrono::steady_clock::time_point timestamp {};
};
```

DLC 转换工具：

```cpp
constexpr std::uint8_t DlcToLength(std::uint8_t dlc) {
    constexpr std::array<std::uint8_t, 16> table {
        0, 1, 2, 3, 4, 5, 6, 7,
        8, 12, 16, 20, 24, 32, 48, 64
    };
    return dlc < table.size() ? table[dlc] : 0;
}

Result<std::uint8_t> LengthToDlc(std::uint8_t length) {
    if (length <= 8) return length;
    if (length <= 12) return 9;
    if (length <= 16) return 10;
    if (length <= 20) return 11;
    if (length <= 24) return 12;
    if (length <= 32) return 13;
    if (length <= 48) return 14;
    if (length <= 64) return 15;
    return Error{ErrorCode::InvalidFrameLength, "CAN FD payload length exceeds 64 bytes"};
}
```

修改原因：

- Classic CAN 中 DLC 0~8 可直接等于字节数。
- CAN FD 中 DLC 9~15 与字节数不相等。
- 编码和解码必须使用 `length` 创建 payload span，而不是直接使用 `dlcCode`。

---

### 2.3 J1939 ID 模型

```cpp
struct J1939Id {
    std::uint8_t priority {};
    std::uint8_t edp {};
    std::uint8_t dp {};
    std::uint8_t pf {};
    std::uint8_t ps {};
    std::uint8_t sa {};

    [[nodiscard]] bool IsPdu1() const noexcept {
        return pf < 240U;
    }

    [[nodiscard]] std::uint32_t pgn() const noexcept {
        const std::uint32_t edpDp =
            (static_cast<std::uint32_t>(edp & 0x1U) << 17U) |
            (static_cast<std::uint32_t>(dp  & 0x1U) << 16U);

        if (IsPdu1()) {
            return edpDp | (static_cast<std::uint32_t>(pf) << 8U);
        }

        return edpDp |
               (static_cast<std::uint32_t>(pf) << 8U) |
               static_cast<std::uint32_t>(ps);
    }

    [[nodiscard]] std::optional<std::uint8_t> destinationAddress() const noexcept {
        if (IsPdu1()) {
            return ps;
        }
        return std::nullopt;
    }

    [[nodiscard]] std::uint8_t sourceAddress() const noexcept {
        return sa;
    }
};

class J1939IdCodec final {
public:
    static Result<J1939Id> Decode(CanMessageKey key);
    static Result<CanMessageKey> Encode(const J1939Id& id);
};
```

修改原因：

- J1939 的 Message 匹配通常基于 PGN，而不是完整 CAN ID。
- PDU1 中 PS 是 DA，不属于 PGN 低 8 位。
- PDU2 中 PS 是 Group Extension，属于 PGN。
- SA 会因 Address Claim 动态变化，不能把完整 29-bit ID 当作唯一 Message key。

---

### 2.4 Signal 唯一标识

```cpp
struct MessageId {
    std::uint32_t index {};
};

struct SignalId {
    std::uint32_t messageIndex {};
    std::uint32_t signalIndex {};

    friend bool operator==(const SignalId&, const SignalId&) = default;
};

struct SignalIdHash {
    std::size_t operator()(const SignalId& id) const noexcept {
        return (static_cast<std::size_t>(id.messageIndex) << 32U) ^ id.signalIndex;
    }
};
```

对外解析接口：

```cpp
Result<SignalId> ResolveSignal(
    CanMessageKey messageKey,
    std::string_view signalName
) const;

Result<SignalId> ResolveSignal(
    std::string_view messageName,
    std::string_view signalName
) const;
```

修改原因：

- 不再认为 signal name 全局唯一。
- 同名信号在不同报文中是合法且常见的。
- Repository、Encoder、Subscriber 使用 `SignalId` 可以避免歧义和字符串热路径查找。

---

### 2.5 Runtime signal value

```cpp
enum class SignalQuality {
    Valid,
    NotAvailable,
    ErrorIndicator,
    Reserved,
    Timeout,
    OutOfRange,
    DecodeError
};

using SignalPhysicalValue = std::variant<
    std::int64_t,
    std::uint64_t,
    double,
    std::string
>;

struct RuntimeSignalValue {
    SignalPhysicalValue physical {};
    std::uint64_t raw {};
    SignalQuality quality {SignalQuality::Valid};
    std::chrono::steady_clock::time_point timestamp {};
    std::optional<std::string_view> enumName {};
};
```

修改原因：

- 原 `SignalValue` 只能表达物理值，无法表达 Not Available、Timeout、Error Indicator。
- J1939 SPN 常见特殊值不能直接当成普通物理量。
- 上层业务需要判断信号是否新鲜、是否有效、是否来自错误指示。

---

## 三、DBC 数据模型整改

### 3.1 Message

```cpp
class Message final {
public:
    Message(
        CanMessageKey key,
        std::string name,
        std::uint8_t payloadLength,
        std::string sender,
        std::vector<Signal> signals,
        MessageAttributes attributes
    );

    Message(const Message&) = delete;
    Message& operator=(const Message&) = delete;
    Message(Message&&) noexcept = default;
    Message& operator=(Message&&) noexcept = default;

    [[nodiscard]] CanMessageKey key() const noexcept;
    [[nodiscard]] std::uint8_t payloadLength() const noexcept;
    [[nodiscard]] std::span<const Signal> signals() const noexcept;
    [[nodiscard]] const Signal* FindSignal(std::string_view name) const;

private:
    void BuildSignalIndex();

private:
    CanMessageKey key_ {};
    std::string name_;
    std::uint8_t payloadLength_ {};
    std::string sender_;
    std::vector<Signal> signals_;
    MessageAttributes attributes_;

    // 使用 owning string 消除 string_view 悬空风险。
    std::unordered_map<std::string, std::size_t> signalByName_;
};
```

### 3.2 DbcDatabase

```cpp
class DbcDatabase final {
public:
    DbcDatabase(const DbcDatabase&) = delete;
    DbcDatabase& operator=(const DbcDatabase&) = delete;
    DbcDatabase(DbcDatabase&&) noexcept = default;
    DbcDatabase& operator=(DbcDatabase&&) noexcept = default;

    static Result<std::shared_ptr<const DbcDatabase>> Load(
        const std::filesystem::path& path,
        const DbcLoadOptions& options
    );

    [[nodiscard]] const Message* FindMessage(CanMessageKey key) const;
    [[nodiscard]] const Message* FindMessageByName(std::string_view name) const;
    [[nodiscard]] const Message* FindJ1939MessageByPgn(std::uint32_t pgn) const;

    [[nodiscard]] Result<SignalId> ResolveSignal(
        CanMessageKey key,
        std::string_view signalName
    ) const;

    [[nodiscard]] const Signal& GetSignal(SignalId id) const;
    [[nodiscard]] const Message& GetMessage(MessageId id) const;

private:
    void BuildIndexes();

private:
    std::vector<Message> messages_;

    std::unordered_map<CanMessageKey, std::size_t, CanMessageKeyHash> messageByKey_;
    std::unordered_map<std::string, std::size_t> messageByName_;
    std::unordered_map<std::uint32_t, std::vector<std::size_t>> j1939MessageByPgn_;
};
```

修改原因：

- 数据库加载完成后是不可变对象，多线程只读无需加锁。
- 禁止 copy，避免内部索引与对象存储脱钩。
- J1939 PGN 可能对应多个 Message variant，因此 PGN 索引使用 `vector<size_t>`，后续可按 SA、DA、sender、attribute 二次筛选。

---

## 四、DBC Parser 整改

### 4.1 Parser 分阶段目标

```text
阶段 1：基础可用
- VERSION
- NS_
- BU_
- BO_
- SG_
- CM_
- VAL_
- VAL_TABLE_
- simple multiplex: M / m0 / m1

阶段 2：工程可用
- BA_DEF_
- BA_DEF_DEF_
- BA_
- BO_TX_BU_
- SIG_VALTYPE_
- GenMsgCycleTime
- GenMsgSendType
- GenSigStartValue
- VFrameFormat
- BusType
- ProtocolType

阶段 3：J1939 / 高级 DBC
- SPN attribute
- J1939 PGN attribute
- SG_MUL_VAL_
- extended multiplexing
- vendor-specific attributes
```

### 4.2 SemanticAnalyzer 新增规则

必须校验：

```text
1. Message ID 是否合法，是否完成 standard/extended normalize。
2. Classic CAN payload length <= 8。
3. CAN FD payload length ∈ {0..8,12,16,20,24,32,48,64}。
4. Signal length ∈ [1,64]。
5. Signal bit range 不超过 message payload length。
6. factor != 0。
7. minimum <= maximum。
8. signed signal length 支持 1..64。
9. Float32 length == 32，Float64 length == 64。
10. 同一 message 内 signal name 不重复。
11. 全局 signal name 可重复，但不能用于无上下文解析。
12. 非 multiplex 分支内信号不能非法重叠。
13. 不同 multiplex branch 可复用 bit 区域。
14. mux signal 必须存在，mux range 必须可解析。
15. J1939 报文必须是 Extended29Bit。
```

---

## 五、BitCodec 整改

### 5.1 保留原则

`BitCodec` 继续保持纯函数：

```text
无状态
无 I/O
无数据库依赖
无驱动依赖
输入相同，输出相同
```

### 5.2 新增 compiled layout

```cpp
struct CompiledSignalLayout {
    std::array<std::uint16_t, 64> frameBits {};
    std::array<std::uint8_t, 64> rawBits {};
    std::uint8_t length {};
};
```

数据库构建时预计算：

```cpp
Result<CompiledSignalLayout> CompileLayout(
    const Signal& signal,
    std::uint8_t payloadLength
);
```

运行时提取：

```cpp
Result<std::uint64_t> ExtractSignal(
    std::span<const std::byte> data,
    const CompiledSignalLayout& layout
);
```

修改原因：

- Motorola 位跳转逻辑复杂，预编译可减少热路径分支。
- 编译阶段可集中做 range 校验。
- 更容易做 golden vector 测试。

### 5.3 64-bit signed 修正

```cpp
struct SignedRawRange {
    std::int64_t min;
    std::int64_t max;
};

constexpr SignedRawRange GetSignedRawRange(std::uint16_t length) {
    if (length == 64) {
        return {
            std::numeric_limits<std::int64_t>::min(),
            std::numeric_limits<std::int64_t>::max()
        };
    }

    const auto max = static_cast<std::int64_t>((1ULL << (length - 1U)) - 1ULL);
    const auto min = -static_cast<std::int64_t>(1ULL << (length - 1U));
    return {min, max};
}
```

修改原因：

- 避免 `1LL << 63`。
- 使用无符号移位构造范围，再转换到有符号。

---

## 六、PhysicalCodec 整改

### 6.1 不再抛异常

```cpp
static Result<double> ToDouble(const SignalPhysicalValue& value);
```

### 6.2 编码流程

```cpp
Result<std::uint64_t> ToRaw(const Signal& signal, SignalPhysicalValue value) {
    auto physical = ToDouble(value);
    if (!physical) {
        return std::unexpected(physical.error());
    }

    if (!std::isfinite(*physical)) {
        return Error{ErrorCode::InvalidSignalValue, "physical value is NaN or Inf"};
    }

    if (signal.factor() == 0.0) {
        return Error{ErrorCode::SemanticError, "factor must not be zero"};
    }

    if (*physical < signal.minimum() || *physical > signal.maximum()) {
        return Error{ErrorCode::PhysicalValueOutOfRange, "physical value out of range"};
    }

    // raw = round((physical - offset) / factor)
}
```

### 6.3 Float32 / Float64

```cpp
Result<SignalPhysicalValue> DecodeFloat(
    const Signal& signal,
    std::uint64_t raw
) {
    if (signal.valueType() == ValueType::Float32) {
        std::uint32_t bits = static_cast<std::uint32_t>(raw & 0xFFFF'FFFFULL);
        float f = std::bit_cast<float>(bits);
        return static_cast<double>(f);
    }

    if (signal.valueType() == ValueType::Float64) {
        double d = std::bit_cast<double>(raw);
        return d;
    }

    return Error{ErrorCode::SemanticError, "not a float signal"};
}
```

修改原因：

- DBC 浮点信号应按 IEEE754 bit pattern 解析。
- 浮点信号通常不应再走整数 raw scaling 流程，除非项目明确约定。

---

## 七、Multiplex 整改

### 7.1 数据模型

```cpp
enum class MultiplexKind {
    None,
    Multiplexor,
    Multiplexed
};

struct MuxRange {
    std::int64_t from {};
    std::int64_t to {};
};

struct MultiplexInfo {
    MultiplexKind kind {MultiplexKind::None};
    std::optional<std::string> switchSignal;
    std::vector<MuxRange> activeRanges;
};
```

### 7.2 active 判断

```cpp
bool IsSignalActive(
    const Signal& signal,
    std::optional<std::int64_t> muxValue
) {
    const auto& mux = signal.multiplex();

    if (mux.kind == MultiplexKind::None ||
        mux.kind == MultiplexKind::Multiplexor) {
        return true;
    }

    if (!muxValue.has_value()) {
        return false;
    }

    for (const auto& range : mux.activeRanges) {
        if (*muxValue >= range.from && *muxValue <= range.to) {
            return true;
        }
    }

    return false;
}
```

修改原因：

- 支持 simple multiplex 和后续 `SG_MUL_VAL_` range。
- 编码时不能写 inactive branch。
- 解码时不能把 inactive branch 的 bit 区域误解释成信号。

---

## 八、FrameDecoder 整改

```cpp
class FrameDecoder final {
public:
    FrameDecoder(
        std::shared_ptr<const DbcDatabase> db,
        MessageResolver resolver
    );

    Result<DecodedFrame> Decode(const CanFrame& frame) const {
        auto message = resolver_.Resolve(frame);
        if (!message) {
            return std::unexpected(message.error());
        }

        if (frame.length < (*message)->payloadLength()) {
            return Error{ErrorCode::InvalidFrameLength, "frame shorter than DBC message"};
        }

        std::span<const std::byte> payload {
            frame.data.data(),
            static_cast<std::size_t>((*message)->payloadLength())
        };

        auto muxValue = DecodeMuxValue(payload, **message);

        DecodedFrame decoded;
        decoded.key = frame.key;
        decoded.messageName = (*message)->name();
        decoded.j1939 = TryDecodeJ1939(frame.key);

        for (const auto& signal : (*message)->signals()) {
            if (!IsSignalActive(signal, muxValue)) {
                continue;
            }

            auto raw = BitCodec::ExtractSignal(payload, signal.layout());
            if (!raw) {
                decoded.signals.push_back(MakeDecodeErrorSignal(signal, raw.error()));
                continue;
            }

            auto physical = PhysicalCodec::ToPhysical(signal, *raw);
            if (!physical) {
                decoded.signals.push_back(MakeDecodeErrorSignal(signal, physical.error()));
                continue;
            }

            decoded.signals.push_back(MakeDecodedSignal(signal, *raw, *physical));
        }

        return decoded;
    }

private:
    std::shared_ptr<const DbcDatabase> db_;
    MessageResolver resolver_;
};
```

修改原因：

- Message 查找逻辑从 Decoder 中剥离。
- frame payload 使用 `length` 而不是 `dlcCode`。
- multiplex inactive signal 不再解码。
- DecodeError 可以按 signal 记录，而不是整帧直接失败，具体策略可配置。

---

## 九、FrameEncoder 与 TxSignalStore 整改

### 9.1 为什么拆出 TxSignalStore

原设计中 `FrameEncoder` 从 `SignalRepository` 逐个读取信号。问题是：

1. 同一报文的多个 signal 可能来自不同时间点，构造出混合帧。
2. RX 状态仓库和 TX 发送状态职责不同。
3. TX 需要默认值、上一次发送值、initial value、counter/checksum hook。

整改后引入：

```cpp
class TxSignalStore final {
public:
    Result<void> SetSignal(SignalId id, SignalPhysicalValue value);
    Result<TxMessageSnapshot> SnapshotMessage(MessageId messageId) const;
};

struct TxMessageSnapshot {
    MessageId messageId;
    std::vector<std::optional<SignalPhysicalValue>> values;
    std::uint64_t version {};
};
```

### 9.2 默认值策略

```cpp
enum class MissingSignalPolicy {
    UseDbcInitialValue,
    UseLastTxValue,
    UseProtocolDefault,
    UseZero,
    Error
};

class DefaultValueResolver final {
public:
    Result<SignalPhysicalValue> Resolve(
        const Signal& signal,
        const std::optional<SignalPhysicalValue>& explicitValue,
        MissingSignalPolicy policy
    ) const;
};
```

默认推荐策略：

```text
1. 应用显式设置值
2. TxShadow 上一次发送值
3. DBC GenSigStartValue
4. 项目配置默认值
5. Error，不建议静默置零
```

### 9.3 Encoder active mux 逻辑

```cpp
Result<CanFrame> FrameEncoder::BuildFrame(MessageId id) const {
    const auto& message = db_->GetMessage(id);
    auto snapshot = txStore_->SnapshotMessage(id);
    if (!snapshot) {
        return std::unexpected(snapshot.error());
    }

    CanFrame frame;
    frame.key = message.key();
    frame.isFd = message.attributes().isCanFd;
    frame.length = message.payloadLength();
    frame.dlcCode = *LengthToDlc(frame.length);

    std::span<std::byte> payload {frame.data.data(), frame.length};
    std::ranges::fill(payload, std::byte{0});

    auto muxValue = ResolveMuxValueForEncoding(message, *snapshot);

    for (const auto& signal : message.signals()) {
        if (!IsSignalActive(signal, muxValue)) {
            continue;
        }

        auto value = defaultResolver_.Resolve(signal, FindValue(signal, *snapshot), policy_);
        if (!value) {
            return std::unexpected(value.error());
        }

        auto raw = PhysicalCodec::ToRaw(signal, *value);
        if (!raw) {
            return std::unexpected(raw.error());
        }

        auto inserted = BitCodec::InsertSignal(payload, signal.layout(), *raw);
        if (!inserted) {
            return std::unexpected(inserted.error());
        }
    }

    ApplyTxHooks(message, payload); // counter/checksum/E2E 可选扩展点
    return frame;
}
```

修改原因：

- 保证同一报文使用一致快照。
- 避免 inactive mux branch 污染 payload。
- 默认值策略可配置，避免静默置零带来控制风险。
- 为 rolling counter、checksum、E2E、SecOC 预留 hook。

---

## 十、SignalRepository 整改

### 10.1 接收侧 Repository

```cpp
class SignalRepository final {
public:
    Result<void> Update(SignalId id, RuntimeSignalValue value);
    std::optional<RuntimeSignalValue> Get(SignalId id) const;
    SignalSnapshot Snapshot(std::span<const SignalId> ids) const;

    Subscription Subscribe(
        SignalId id,
        SignalCallback callback,
        CallbackPolicy policy = CallbackPolicy::Immediate
    );
};
```

### 10.2 订阅生命周期安全

```cpp
class Subscription final {
public:
    Subscription() = default;
    explicit Subscription(std::weak_ptr<RepositoryState> state, SignalId id, std::uint64_t cbId);
    ~Subscription();
    void Cancel();

private:
    std::weak_ptr<RepositoryState> state_;
    SignalId id_ {};
    std::uint64_t callbackId_ {};
};
```

修改原因：

- 不捕获裸 `this`。
- Repository 先析构时，Subscription 的 `Cancel()` 只会发现 weak_ptr 失效，不访问悬空对象。

### 10.3 回调执行策略

```cpp
enum class CallbackPolicy {
    Immediate,     // 调用 Set/Update 的线程直接执行
    Dispatch       // 投递到业务线程或 executor
};
```

修改原因：

- 简单场景下 immediate 方便。
- 实车/高频 RX 下，推荐 dispatch，避免业务回调阻塞 RX/Decode 线程。

---

## 十一、PeriodicScheduler 整改

### 11.1 修正 wait_until 唤醒问题

```cpp
class PeriodicScheduler final {
private:
    std::mutex mutex_;
    std::condition_variable cv_;
    std::unordered_map<MessageId, PeriodicTask, MessageIdHash> tasks_;
    std::uint64_t scheduleVersion_ {};
    bool running_ {false};
};
```

Add/Remove：

```cpp
void AddTask(MessageId id, std::chrono::milliseconds period) {
    std::lock_guard lock(mutex_);
    tasks_[id] = PeriodicTask{ id, period, Clock::now() + period, true };
    ++scheduleVersion_;
    cv_.notify_one();
}
```

RunLoop：

```cpp
void RunLoop() {
    std::unique_lock lock(mutex_);

    while (running_) {
        if (tasks_.empty()) {
            cv_.wait(lock, [this] {
                return !running_ || !tasks_.empty();
            });
            continue;
        }

        const auto version = scheduleVersion_;
        const auto nextDue = FindNextDueLocked();

        cv_.wait_until(lock, nextDue, [this, version] {
            return !running_ || scheduleVersion_ != version;
        });

        if (!running_) {
            break;
        }

        if (scheduleVersion_ != version) {
            continue; // 任务变化，重新计算 nextDue
        }

        auto dueMessages = CollectDueTasksLocked(Clock::now());
        lock.unlock();

        for (const auto id : dueMessages) {
            auto frame = encoder_.BuildFrame(id);
            if (frame) {
                static_cast<void>(sender_.Send(*frame, sendTimeout_));
            }
        }

        lock.lock();
    }
}
```

### 11.2 missed period 策略

```cpp
enum class MissedPeriodPolicy {
    SkipMissed,    // 推荐实车使用
    CatchUp,
    AlignToGrid
};
```

推荐默认：`SkipMissed`。

修改原因：

- 实车周期报文不应因线程延迟后连续补发过期帧。
- 总线负载、接收方超时逻辑和控制时序更适合跳过过期周期。

---

## 十二、CAN Driver 接口整改

```cpp
class ICanSender {
public:
    virtual ~ICanSender() = default;
    virtual Result<void> Send(
        const CanFrame& frame,
        std::chrono::milliseconds timeout
    ) = 0;
};

class ICanReceiver {
public:
    virtual ~ICanReceiver() = default;
    virtual Result<CanFrame> Receive(
        std::chrono::milliseconds timeout
    ) = 0;
};

class ICanDriver : public ICanSender, public ICanReceiver {
public:
    ~ICanDriver() override = default;
};
```

可选扩展接口：

```cpp
class ICanFilterConfigurable {
public:
    virtual ~ICanFilterConfigurable() = default;
    virtual Result<void> SetFilter(std::span<const CanFilter> filters) = 0;
};

class ICanBusStatistics {
public:
    virtual ~ICanBusStatistics() = default;
    virtual CanBusStats GetStats() const = 0;
};

class ICanBusStateObserver {
public:
    virtual ~ICanBusStateObserver() = default;
    virtual Result<CanBusState> GetBusState() const = 0;
};
```

修改原因：

- `Receive(timeout)` 保证 `RxWorker::Stop()` 可退出。
- 不把过滤、统计、bus state 全塞进基础接口，保持 ISP。
- 各厂商高级功能通过小接口扩展。

---

## 十三、J1939 扩展设计

### 13.1 单帧 PGN 解码

```cpp
class J1939MessageResolver final {
public:
    Result<const Message*> Resolve(const CanFrame& frame) const {
        auto id = J1939IdCodec::Decode(frame.key);
        if (!id) {
            return std::unexpected(id.error());
        }

        auto candidates = db_->FindJ1939MessagesByPgn(id->pgn());
        if (candidates.empty()) {
            return Error{ErrorCode::MessageNotFound, "J1939 PGN not found"};
        }

        return SelectBestCandidate(candidates, *id);
    }
};
```

### 13.2 TP/BAM/RTS/CTS 扩展点

```cpp
class IJ1939TransportProtocol {
public:
    virtual ~IJ1939TransportProtocol() = default;
    virtual Result<std::optional<CanFrame>> OnFrame(const CanFrame& frame) = 0;
    virtual Result<std::vector<CanFrame>> Segment(const CanFrame& longFrame) = 0;
};
```

整改后 V2.0 范围：

```text
必须支持：J1939 单帧 PGN 解析、PDU1/PDU2、SA/DA 提取
预留接口：BAM、RTS/CTS、多包重组、Address Claim
暂不强制实现：完整 J1939 Network Management
```

修改原因：

- 设计上先把 J1939 ID 模型放对。
- TP 可后续独立实现，不污染 BitCodec / PhysicalCodec。

---

## 十四、Error 设计整改

```cpp
enum class ErrorCode {
    Ok,
    FileNotFound,
    LexicalError,
    SyntaxError,
    SemanticError,
    MessageNotFound,
    SignalNotFound,
    AmbiguousSignalName,
    InvalidFrameId,
    InvalidFrameLength,
    InvalidDlcCode,
    InvalidSignalLength,
    InvalidSignalValue,
    InvalidScalingFactor,
    PhysicalValueOutOfRange,
    UnsupportedValueType,
    DriverError,
    DriverTimeout,
    BusOff,
    QueueFull,
    SchedulerStopped
};

struct Error {
    ErrorCode code {};
    std::string message;

    std::optional<std::filesystem::path> file;
    std::optional<std::uint32_t> line;
    std::optional<std::uint32_t> column;

    std::optional<CanMessageKey> messageKey;
    std::optional<std::string> messageName;
    std::optional<std::string> signalName;

    std::optional<int> nativeErrorCode;
};
```

修改原因：

- 解析 DBC 时必须能定位到文件、行、列。
- 运行时错误必须能定位到 Message 和 Signal。
- 驱动错误需要保留底层 native error code，方便现场诊断。

---

## 十五、测试策略整改

### 15.1 单元测试

|模块|新增重点|
|---|---|
|`CanMessageKey`|Standard / Extended / DBC extended marker normalize|
|`CanFrame`|CAN FD DLC code 与 length 转换|
|`J1939IdCodec`|PDU1 / PDU2 / PGN / SA / DA|
|`BitCodec`|Motorola golden vector、边界 bit、64-bit|
|`PhysicalCodec`|factor=0、NaN、signed 64-bit、Float32、Float64|
|`DbcParser`|BA_、BA_DEF_、SIG_VALTYPE_、SG_MUL_VAL_|
|`FrameDecoder`|J1939 PGN 匹配、mux inactive filter|
|`FrameEncoder`|mux active encoding、default value policy|
|`TxSignalStore`|message snapshot 一致性|
|`SignalRepository`|Subscription 生命周期、回调锁外执行|
|`Scheduler`|动态新增更早任务、Stop、missed period|

### 15.2 Golden tests

必须维护一组固定向量：

```text
classic_intel_8bit.dbc
classic_intel_cross_byte.dbc
classic_motorola_16bit_start7.dbc
classic_motorola_12bit_start23.dbc
signed_12bit_negative.dbc
signed_64bit.dbc
canfd_64bytes.dbc
multiplex_simple.dbc
multiplex_extended_range.dbc
j1939_pdu1.dbc
j1939_pdu2.dbc
value_table.dbc
float32_float64.dbc
```

每个 golden case 包含：

```text
1. DBC 文件
2. 输入 CAN frame bytes
3. 期望 raw value
4. 期望 physical value
5. 期望 enum name / quality
6. 期望重新编码后的 bytes
```

### 15.3 工具对齐测试

建议至少和一个外部成熟工具对齐：

```text
cantools
canmatrix
Vector CANdb++ 导出的 reference
```

修改原因：

- Round-trip 只能证明 encode/decode 自洽，不能证明位序正确。
- Motorola bit numbering 必须有第三方或人工 golden vector 对齐。

### 15.4 Sanitizer / 并发测试

```text
ASAN: 检查 string_view、Subscription 生命周期、越界访问
UBSAN: 检查 shift、integer overflow、bit_cast 边界
TSAN: 检查 Repository、Scheduler、TxStore 并发访问
Fuzz: DBC Lexer / Parser
```

---

## 十六、推荐开发顺序整改

```text
1. 定义 CanMessageKey / CanFrame(length + dlcCode) / Error / Result。
2. 实现 DBC ID normalize，修复 Standard / Extended 区分。
3. 引入 SignalId / MessageId，移除全局 signalName 依赖。
4. 禁止 DbcDatabase / Message copy，修正 string_view 生命周期问题。
5. 实现 CAN FD DLC code <-> length 转换。
6. 实现 J1939IdCodec，完成 PDU1 / PDU2 / PGN 测试。
7. 实现 BitCodec golden vector 测试，尤其 Motorola。
8. 修复 PhysicalCodec：factor=0、64-bit signed、NaN、Float32/Float64。
9. 重构 FrameDecoder，引入 MessageResolver 和 mux active filter。
10. 新增 TxSignalStore，FrameEncoder 基于 message snapshot 编码。
11. 修正 multiplex encoding，不再写 inactive branch。
12. 修正 Scheduler scheduleVersion 唤醒逻辑。
13. 修正 SignalRepository Subscription 生命周期。
14. Driver Send/Receive 增加 timeout。
15. Parser 增加 BA_ / BA_DEF_ / SIG_VALTYPE_ / SG_MUL_VAL_。
16. 增加 J1939 TP/BAM/RTS/CTS 扩展接口。
17. 加入 ASAN / UBSAN / TSAN / fuzz / golden regression。
```

---

## 十七、整改后模块目录建议

```text
can_dbc/
├── include/can_dbc/
│   ├── core/
│   │   ├── CanFrame.hpp
│   │   ├── CanMessageKey.hpp
│   │   ├── Error.hpp
│   │   ├── Result.hpp
│   │   └── Types.hpp
│   │
│   ├── dbc/
│   │   ├── DbcDatabase.hpp
│   │   ├── DbcLoader.hpp
│   │   ├── DbcLexer.hpp
│   │   ├── DbcParser.hpp
│   │   ├── DbcAst.hpp
│   │   ├── SemanticAnalyzer.hpp
│   │   ├── Message.hpp
│   │   ├── Signal.hpp
│   │   ├── SignalId.hpp
│   │   ├── ValueTable.hpp
│   │   └── Attributes.hpp
│   │
│   ├── j1939/
│   │   ├── J1939Id.hpp
│   │   ├── J1939IdCodec.hpp
│   │   ├── J1939MessageResolver.hpp
│   │   └── J1939TransportProtocol.hpp
│   │
│   ├── codec/
│   │   ├── BitCodec.hpp
│   │   ├── CompiledSignalLayout.hpp
│   │   ├── PhysicalCodec.hpp
│   │   ├── MultiplexResolver.hpp
│   │   ├── MessageResolver.hpp
│   │   ├── FrameDecoder.hpp
│   │   └── FrameEncoder.hpp
│   │
│   ├── repository/
│   │   ├── RuntimeSignalValue.hpp
│   │   ├── SignalRepository.hpp
│   │   ├── SignalSnapshot.hpp
│   │   ├── TxSignalStore.hpp
│   │   └── Subscription.hpp
│   │
│   ├── driver/
│   │   ├── ICanSender.hpp
│   │   ├── ICanReceiver.hpp
│   │   ├── ICanDriver.hpp
│   │   ├── CanFilter.hpp
│   │   ├── CanBusStats.hpp
│   │   └── adapters/
│   │       ├── SocketCanDriver.hpp
│   │       ├── VectorCanDriver.hpp
│   │       ├── PeakCanDriver.hpp
│   │       ├── KvaserCanDriver.hpp
│   │       └── ZlgCanDriver.hpp
│   │
│   ├── scheduler/
│   │   ├── PeriodicTask.hpp
│   │   └── PeriodicScheduler.hpp
│   │
│   └── runtime/
│       ├── CanRuntime.hpp
│       ├── RxWorker.hpp
│       └── TxWorker.hpp
│
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── regression/
│   │   └── golden_vectors/
│   └── fuzz/
│
└── examples/
    ├── decode_frame.cpp
    ├── encode_frame.cpp
    ├── periodic_send.cpp
    └── j1939_decode.cpp
```

---

## 十八、最终设计取舍

### 18.1 保留原设计的部分

保留：

```text
1. DBC Database 不负责运行时状态。
2. BitCodec 独立为纯算法。
3. PhysicalCodec 与 BitCodec 分离。
4. Decoder 不直接写 Repository。
5. Encoder 不直接调用 Driver。
6. Scheduler 只负责什么时候发送。
7. ICanSender / ICanReceiver 分离。
8. Repository 回调不在锁内执行。
```

这些设计方向是正确的，整改不是推翻，而是强化生产可用性。

### 18.2 变更较大的部分

变更：

```text
1. Message ID 从 uint32_t 升级为 CanMessageKey。
2. CAN FD 从 dlc 单字段升级为 dlcCode + length。
3. 全局 signalName 访问升级为 SignalId / Message + SignalName。
4. RX Repository 与 TX SignalStore 分离。
5. J1939 从“扩展点”升级为一等协议模型。
6. Multiplex 从单值 m0/m1 升级为 range-capable 模型。
7. Scheduler 增加 scheduleVersion 修复动态任务唤醒。
8. Subscription 生命周期从裸 this 改为 weak control block。
```

---

## 十九、结论

整改后的设计更适合真实汽车电子项目：

```text
Classic CAN: 支持 11-bit / 29-bit 正确索引
CAN FD: 正确处理 DLC code 与 payload length
DBC: 支持工程常用 attribute、float、initial value、multiplex
J1939: 支持 PGN / SA / DA / PDU1 / PDU2 模型
Runtime: 支持信号质量、时间戳、raw value、订阅安全
TX: 支持 message snapshot、默认值策略、mux active encoding
Scheduler: 支持动态任务及时重算和 missed period 策略
Driver: 支持 timeout、可停止 RX、可扩展总线状态和过滤
Testing: 支持 golden vector、fuzz、sanitizer、第三方工具对齐
```

最终目标不是把所有高级功能一次性实现，而是先把底层抽象放对。尤其是 `CanMessageKey`、`SignalId`、`J1939IdCodec`、`dlcCode/length`、`TxSignalStore` 和 `MessageResolver`，这些属于架构地基，必须优先整改。否则后续接入真实整车 DBC、J1939 DBC 或 CAN FD 报文时，会产生高成本返工。
