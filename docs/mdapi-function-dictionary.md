# MdApiPy 行情接口函数字典

本文是 `ctpwrapper.MdApiPy` 的行情接口备查手册，重点说明每个函数的调用逻辑、使用场景、参数含义、返回值和示例。`MdApiPy` 位于 `ctpwrapper.Md`，底层封装 CTP 官方 `CThostFtdcMdApi` 行情接口，使用方式是“主动请求 + 异步回调”。

## 1. 总体逻辑

### 1.1 核心模型

行情接口不是普通的同步函数调用模型，而是事件驱动模型：

1. 客户端调用主动函数，例如 `Create()`、`RegisterFront()`、`Init()`、`ReqUserLogin()`、`SubscribeMarketData()`。
2. CTP API 在后台线程中连接前置机、收发报文。
3. 服务端或底层 API 通过回调函数通知结果，例如 `OnFrontConnected()`、`OnRspUserLogin()`、`OnRspSubMarketData()`、`OnRtnDepthMarketData()`。
4. 使用者通常继承 `MdApiPy`，只重写自己关心的回调。

### 1.2 推荐调用顺序

```text
实例化自定义 MdApiPy 子类
        │
        ▼
Create() 创建底层行情 API
        │
        ▼
RegisterFront() 或 RegisterNameServer() 注册连接地址
        │
        ▼
可选：RegisterFensUserInfo() 注册名字服务器用户信息
        │
        ▼
Init() 启动 API 工作线程
        │
        ▼
OnFrontConnected() 被回调
        │
        ▼
ReqUserLogin() 发起登录
        │
        ▼
OnRspUserLogin() 收到登录响应
        │
        ▼
SubscribeMarketData() 订阅合约行情
        │
        ▼
OnRspSubMarketData() 收到订阅响应
        │
        ▼
OnRtnDepthMarketData() 持续收到行情推送
        │
        ▼
结束前 UnSubscribeMarketData() / ReqUserLogout() / Release()
```

### 1.3 请求编号 request_id

`ReqUserLogin()`、`ReqUserLogout()`、`ReqQryMulticastInstrument()` 这类主动请求都需要传入 `nRequestID`。建议由客户端维护一个递增整数，便于在响应回调中关联请求和响应。

```python
# 中文注释：维护一个简单递增请求编号
self.request_id += 1
request_id = self.request_id
```

### 1.4 响应错误判断

大多数响应回调包含 `pRspInfo`。通常：

- `pRspInfo.ErrorID == 0`：成功。
- `pRspInfo.ErrorID != 0`：失败，错误原因通常在 `pRspInfo.ErrorMsg` 中。

示例：

```python
def is_success(pRspInfo):
    # 中文注释：部分回调理论上可能传空，实际使用时先做保护
    return pRspInfo is not None and pRspInfo.ErrorID == 0
```

## 2. 最小可运行结构

```python
# encoding:utf-8
import time
from ctpwrapper import ApiStructure, MdApiPy


class MyMdApi(MdApiPy):
    """中文注释：自定义行情 API，通过重写回调处理异步事件。"""

    def __init__(self, broker_id, user_id, password, instruments):
        # 中文注释：保存登录参数和需要订阅的合约列表
        self.broker_id = broker_id
        self.user_id = user_id
        self.password = password
        self.instruments = instruments
        self.request_id = 0
        self.login = False

    def next_request_id(self):
        # 中文注释：CTP 请求编号通常由客户端递增维护
        self.request_id += 1
        return self.request_id

    def OnFrontConnected(self):
        # 中文注释：前置连接成功后才能发起登录
        req = ApiStructure.ReqUserLoginField(
            BrokerID=self.broker_id,
            UserID=self.user_id,
            Password=self.password,
        )
        self.ReqUserLogin(req, self.next_request_id())

    def OnRspUserLogin(self, pRspUserLogin, pRspInfo, nRequestID, bIsLast):
        # 中文注释：登录成功后再订阅行情，避免订阅过早导致无推送
        if pRspInfo.ErrorID == 0:
            self.login = True
            print("行情登录成功", pRspUserLogin.TradingDay)
            self.SubscribeMarketData(self.instruments)
        else:
            print("行情登录失败", pRspInfo.ErrorID, pRspInfo.ErrorMsg)

    def OnRspSubMarketData(self, pSpecificInstrument, pRspInfo, nRequestID, bIsLast):
        # 中文注释：订阅响应只代表服务端受理结果，不代表马上一定有行情
        print("订阅响应", pSpecificInstrument.InstrumentID, pRspInfo.ErrorID)

    def OnRtnDepthMarketData(self, pDepthMarketData):
        # 中文注释：行情推送是高频回调，真实项目中不要做耗时操作
        print(pDepthMarketData.InstrumentID, pDepthMarketData.LastPrice)


md = MyMdApi("9999", "你的账号", "你的密码", ["rb2410"])
md.Create()
md.RegisterFront("tcp://行情前置地址:端口")
md.Init()

while not md.login:
    time.sleep(0.2)

md.Join()
```

## 3. 主动方法字典

### 3.1 `Create(pszFlowPath="", bIsUsingUdp=False, bIsMulticast=False, bIsProductionMode=True)`

**作用**：创建底层行情 API 实例，是使用 `MdApiPy` 的第一步。

**逻辑**：该方法会调用底层 `CreateFtdcMdApi`，初始化 CTP 行情 API 对象。未调用 `Create()` 前，不应调用 `Init()`、`ReqUserLogin()`、订阅等方法。

**参数**：

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `pszFlowPath` | `str` | 流文件目录，用于保存订阅信息等运行文件。默认空字符串表示当前目录。多实例建议使用不同目录。 |
| `bIsUsingUdp` | `bool` | 是否使用 UDP 行情。普通 TCP 行情通常为 `False`。 |
| `bIsMulticast` | `bool` | 是否使用组播行情。普通行情通常为 `False`。 |
| `bIsProductionMode` | `bool` | 是否生产模式。通常保持默认 `True`。 |

**什么时候使用**：程序启动、实例化行情 API 后立即调用。每个 API 实例通常只调用一次。

**示例**：

```python
# 中文注释：使用默认流文件路径创建行情 API
md.Create()

# 中文注释：多实例时建议每个实例使用独立流文件目录
md.Create("./flow/md_account_1/")
```

**注意事项**：

- `pszFlowPath` 目录建议提前创建，避免不同实例共用同一流文件目录。
- 不要在同一个实例上反复 `Create()`。

### 3.2 `RegisterFront(pszFrontAddress)`

**作用**：注册行情前置机地址。

**逻辑**：告诉底层 API 要连接哪一个行情前置地址。实际连接动作在 `Init()` 后发生。

**参数**：

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `pszFrontAddress` | `str` | 行情前置地址，格式通常为 `tcp://ip:port`。 |

**什么时候使用**：调用 `Create()` 后、`Init()` 前。适合直接使用期货公司提供的固定前置地址。

**示例**：

```python
# 中文注释：注册期货公司提供的行情前置地址
md.RegisterFront("tcp://180.168.146.187:10131")
```

**注意事项**：

- 只注册地址不会立即连接，必须再调用 `Init()`。
- 如果同时使用 `RegisterNameServer()`，名字服务器优先级更高。

### 3.3 `RegisterNameServer(pszNsAddress)`

**作用**：注册名字服务器地址。

**逻辑**：名字服务器会根据用户和环境返回实际可用的前置地址。底层 CTP API 文档说明其优先级高于 `RegisterFront()`。

**参数**：

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `pszNsAddress` | `str` | 名字服务器地址，格式通常为 `tcp://ip:port`。 |

**什么时候使用**：期货公司提供名字服务器地址，或需要通过 FENS/名字服务动态发现前置地址时使用。

**示例**：

```python
# 中文注释：注册名字服务器，实际前置由名字服务发现
md.RegisterNameServer("tcp://ns.example.com:12001")
```

**注意事项**：

- 需要时配合 `RegisterFensUserInfo()` 使用。
- 如果期货公司只提供普通前置地址，用 `RegisterFront()` 即可。

### 3.4 `RegisterFensUserInfo(pFensUserInfo)`

**作用**：注册名字服务器用户信息。

**逻辑**：在使用名字服务器时，提供经纪公司、用户、登录模式等信息，便于名字服务器返回匹配的前置地址。

**参数**：

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `pFensUserInfo` | `ApiStructure.FensUserInfoField` | FENS 用户信息结构体。 |

**什么时候使用**：使用 `RegisterNameServer()` 且期货公司要求传入 FENS 用户信息时。

**示例**：

```python
from ctpwrapper import ApiStructure

# 中文注释：按期货公司要求填写 FENS 用户信息
fens = ApiStructure.FensUserInfoField(
    BrokerID="9999",
    UserID="你的账号",
    LoginMode="0",
)
md.RegisterFensUserInfo(fens)
```

**注意事项**：

- `LoginMode` 的具体取值以柜台或期货公司说明为准。
- 普通前置连接通常不需要此函数。

### 3.5 `Init()`

**作用**：初始化并启动 API 工作线程。

**逻辑**：调用后底层线程开始连接前置机。连接成功后会触发 `OnFrontConnected()` 回调。`ctpwrapper.MdApiPy.Init()` 内部在调用底层 `Init()` 后会 `sleep(1)` 等待 C++ 初始化。

**什么时候使用**：完成 `Create()` 和地址注册后调用。

**示例**：

```python
# 中文注释：启动行情接口线程，等待后续回调
md.Init()
```

**注意事项**：

- `Init()` 后不要假设已经登录成功，应等待 `OnFrontConnected()` 和 `OnRspUserLogin()`。
- 订阅行情建议放在 `OnRspUserLogin()` 登录成功分支中。

### 3.6 `Join()`

**作用**：阻塞当前线程，等待 API 工作线程结束。

**逻辑**：通常用于脚本主线程保活，否则主进程退出后回调线程也会结束。

**返回值**：线程退出代码，类型为 `int`。

**什么时候使用**：示例程序、长期运行的行情采集程序结尾处使用。

**示例**：

```python
# 中文注释：阻塞主线程，让程序持续接收行情回调
md.Join()
```

**注意事项**：

- `Join()` 会阻塞，放在程序最后。
- 如果你有自己的事件循环或服务框架，可以不用 `Join()`，但要保证进程不退出。

### 3.7 `Release()`

**作用**：释放底层 API 资源。

**逻辑**：通知底层对象释放连接、线程和内部资源。

**什么时候使用**：程序退出、需要主动销毁 API 实例时。

**示例**：

```python
try:
    md.Init()
    md.Join()
finally:
    # 中文注释：程序结束前释放行情 API 资源
    md.Release()
```

**注意事项**：

- 释放后不要继续调用该实例的方法。
- 真实服务中建议做好退出信号处理，避免资源未释放。

### 3.8 `ReqUserLogin(pReqUserLogin, nRequestID)`

**作用**：发起行情登录请求。

**逻辑**：主动请求是异步的，本函数返回值只代表请求是否成功发送到底层 API；真正登录结果通过 `OnRspUserLogin()` 返回。

**参数**：

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `pReqUserLogin` | `ApiStructure.ReqUserLoginField` | 登录请求结构体，常用字段有 `BrokerID`、`UserID`、`Password`。 |
| `nRequestID` | `int` | 请求编号，由客户端递增维护。 |

**返回值**：`int`。通常 `0` 表示底层 API 接收请求成功，非 `0` 表示发送失败或接口状态不允许。

**什么时候使用**：在 `OnFrontConnected()` 中调用最常见，因为此时已经建立前置连接。

**示例**：

```python
def OnFrontConnected(self):
    # 中文注释：前置连接成功后发起登录
    req = ApiStructure.ReqUserLoginField(
        BrokerID=self.broker_id,
        UserID=self.user_id,
        Password=self.password,
    )
    ret = self.ReqUserLogin(req, self.next_request_id())
    print("发送登录请求返回值", ret)
```

**注意事项**：

- 不要在 `RegisterFront()` 后立即登录，通常应等待 `OnFrontConnected()`。
- 登录成功后再订阅，稳定性更好。

### 3.9 `ReqUserLogout(pUserLogout, nRequestID)`

**作用**：发起行情登出请求。

**逻辑**：异步请求，登出响应通过 `OnRspUserLogout()` 返回。

**参数**：

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `pUserLogout` | `ApiStructure.UserLogoutField` | 登出请求结构体，常用字段有 `BrokerID`、`UserID`。 |
| `nRequestID` | `int` | 请求编号。 |

**返回值**：`int`，含义同其他请求方法。

**什么时候使用**：主动停止行情程序、切换账号、释放资源前希望正常登出时。

**示例**：

```python
# 中文注释：退出前主动登出行情账号
req = ApiStructure.UserLogoutField(BrokerID="9999", UserID="你的账号")
md.ReqUserLogout(req, 2)
```

### 3.10 `ReqQryMulticastInstrument(pQryMulticastInstrument, nRequestID)`

**作用**：请求查询组播合约。

**逻辑**：用于组播行情相关场景，服务端返回通过 `OnRspQryMulticastInstrument()` 分批通知。

**参数**：

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `pQryMulticastInstrument` | `ApiStructure.QryMulticastInstrumentField` | 查询组播合约请求结构体。 |
| `nRequestID` | `int` | 请求编号。 |

**什么时候使用**：使用 UDP/组播行情，且需要查询组播合约信息时。普通 TCP 行情订阅一般不需要。

**示例**：

```python
# 中文注释：组播行情场景下查询组播合约
req = ApiStructure.QryMulticastInstrumentField()
md.ReqQryMulticastInstrument(req, self.next_request_id())
```

### 3.11 `GetTradingDay()`

**作用**：获取当前交易日。

**逻辑**：从底层 API 读取当前交易日字符串。官方说明通常要求登录成功后才能得到可靠结果。

**返回值**：`str`，例如 `"20260710"`。

**什么时候使用**：登录成功后记录行情交易日、生成日志或数据文件分区时。

**示例**：

```python
def OnRspUserLogin(self, pRspUserLogin, pRspInfo, nRequestID, bIsLast):
    if pRspInfo.ErrorID == 0:
        # 中文注释：登录成功后获取交易日更可靠
        trading_day = self.GetTradingDay()
        print("当前交易日", trading_day)
```

**注意事项**：

- 不建议在登录前依赖该值。
- 夜盘场景下交易日可能与自然日不同。

### 3.12 `SubscribeMarketData(pInstrumentID)`

**作用**：订阅指定合约的深度行情。

**逻辑**：传入合约 ID 列表，包装层会将 Python 字符串转为底层 bytes 数组。订阅请求的响应通过 `OnRspSubMarketData()` 返回；真实行情数据通过 `OnRtnDepthMarketData()` 持续推送。

**参数**：

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `pInstrumentID` | `list[str]` | 合约 ID 列表，例如 `["rb2410", "cu2409"]`。 |

**返回值**：`int`。`0` 通常表示请求发送成功，具体订阅是否成功看 `OnRspSubMarketData()`。

**什么时候使用**：登录成功后需要接收某些合约行情时。最常放在 `OnRspUserLogin()` 成功分支中。

**示例**：

```python
def OnRspUserLogin(self, pRspUserLogin, pRspInfo, nRequestID, bIsLast):
    if pRspInfo.ErrorID == 0:
        # 中文注释：登录成功后订阅多个合约行情
        ret = self.SubscribeMarketData(["rb2410", "cu2409"])
        print("发送订阅请求返回值", ret)
```

**注意事项**：

- 合约 ID 必须是柜台当前识别的合约，过期合约或拼写错误通常会订阅失败。
- 不要在行情回调中执行耗时订阅逻辑；需要动态订阅时可以交给队列或单独控制线程。
- 订阅成功不代表当前市场一定有成交，非交易时段可能没有行情推送。

### 3.13 `UnSubscribeMarketData(pInstrumentID)`

**作用**：退订指定合约的深度行情。

**逻辑**：退订请求响应通过 `OnRspUnSubMarketData()` 返回。退订后通常不再收到对应合约的 `OnRtnDepthMarketData()` 推送。

**参数**：

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `pInstrumentID` | `list[str]` | 要退订的合约 ID 列表。 |

**返回值**：`int`。

**什么时候使用**：策略不再关注某些合约、切换主力合约、减少带宽和回调压力时。

**示例**：

```python
# 中文注释：不再需要 rb2410 行情时主动退订
md.UnSubscribeMarketData(["rb2410"])
```

### 3.14 `SubscribeForQuoteRsp(pInstrumentID)`

**作用**：订阅询价响应。

**逻辑**：订阅指定合约的询价响应事件。订阅结果通过 `OnRspSubForQuoteRsp()` 返回，询价通知通过 `OnRtnForQuoteRsp()` 推送。

**参数**：

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `pInstrumentID` | `list[str]` | 合约 ID 列表。 |

**返回值**：`int`。

**什么时候使用**：需要接收询价相关通知的业务，例如期权、做市或特定询价业务。普通行情订阅通常不需要。

**示例**：

```python
# 中文注释：订阅指定合约的询价响应
md.SubscribeForQuoteRsp(["IO2409-C-3500"])
```

### 3.15 `UnSubscribeForQuoteRsp(pInstrumentID)`

**作用**：退订询价响应。

**逻辑**：退订结果通过 `OnRspUnSubForQuoteRsp()` 返回。

**参数**：

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `pInstrumentID` | `list[str]` | 合约 ID 列表。 |

**返回值**：`int`。

**什么时候使用**：不再需要某些合约的询价通知时。

**示例**：

```python
# 中文注释：退订询价响应，降低无关回调
md.UnSubscribeForQuoteRsp(["IO2409-C-3500"])
```

## 4. 回调函数字典

### 4.1 `OnFrontConnected()`

**触发时机**：客户端与行情前置机建立通信连接时触发，此时尚未登录。

**常见用途**：发起 `ReqUserLogin()`。

**示例**：

```python
def OnFrontConnected(self):
    # 中文注释：连接建立后立刻登录
    req = ApiStructure.ReqUserLoginField(
        BrokerID=self.broker_id,
        UserID=self.user_id,
        Password=self.password,
    )
    self.ReqUserLogin(req, self.next_request_id())
```

**注意事项**：

- 断线重连成功后可能再次触发，需要考虑重复登录和重复订阅。
- 不要在该回调里做耗时操作。

### 4.2 `OnFrontDisconnected(nReason)`

**触发时机**：客户端与行情前置机通信连接断开时触发。

**参数**：

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `nReason` | `int` | 断开原因代码。 |

**常见原因码**：

| 原因码 | 十六进制 | 含义 |
| --- | --- | --- |
| `4097` | `0x1001` | 网络读失败。 |
| `4098` | `0x1002` | 网络写失败。 |
| `8193` | `0x2001` | 读心跳超时。 |
| `8194` | `0x2002` | 发送心跳超时。 |
| `8195` | `0x2003` | 收到不能识别的错误消息。 |

**常见用途**：记录日志、更新连接状态、暂停依赖行情的策略。

**示例**：

```python
def OnFrontDisconnected(self, nReason):
    # 中文注释：断线后底层 API 通常会自动重连，这里只更新状态和记录日志
    self.login = False
    print("行情前置断开", nReason)
```

**注意事项**：

- CTP API 通常会自动重连，客户端一般不需要手动 `Init()`。
- 重连后可能需要重新登录和重新订阅，具体行为应以实际柜台测试为准。

### 4.3 `OnHeartBeatWarning(nTimeLapse)`

**触发时机**：长时间未收到报文，底层 API 发出心跳超时警告。

**参数**：

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `nTimeLapse` | `int` | 距离上次接收报文的时间。 |

**常见用途**：记录网络质量、告警监控。

**示例**：

```python
def OnHeartBeatWarning(self, nTimeLapse):
    # 中文注释：心跳告警通常说明网络或前置响应异常
    print("行情心跳告警，距离上次报文秒数", nTimeLapse)
```

### 4.4 `OnRspUserLogin(pRspUserLogin, pRspInfo, nRequestID, bIsLast)`

**触发时机**：`ReqUserLogin()` 的响应返回时触发。

**参数**：

| 参数 | 说明 |
| --- | --- |
| `pRspUserLogin` | 登录响应结构体，常见字段包含 `TradingDay`、`LoginTime`、`BrokerID`、`UserID` 等。 |
| `pRspInfo` | 响应错误信息，`ErrorID == 0` 表示成功。 |
| `nRequestID` | 对应请求编号。 |
| `bIsLast` | 是否最后一条响应。 |

**常见用途**：确认登录结果；登录成功后订阅行情；记录交易日。

**示例**：

```python
def OnRspUserLogin(self, pRspUserLogin, pRspInfo, nRequestID, bIsLast):
    # 中文注释：只有 ErrorID 为 0 才继续订阅行情
    if pRspInfo.ErrorID == 0:
        self.login = True
        print("登录成功，交易日", self.GetTradingDay())
        self.SubscribeMarketData(self.instruments)
    else:
        print("登录失败", pRspInfo.ErrorID, pRspInfo.ErrorMsg)
```

### 4.5 `OnRspUserLogout(pUserLogout, pRspInfo, nRequestID, bIsLast)`

**触发时机**：`ReqUserLogout()` 的响应返回时触发。

**常见用途**：确认登出结果、更新登录状态、准备释放资源。

**示例**：

```python
def OnRspUserLogout(self, pUserLogout, pRspInfo, nRequestID, bIsLast):
    # 中文注释：登出成功后标记登录状态为 False
    if pRspInfo.ErrorID == 0:
        self.login = False
        print("行情登出成功")
    else:
        print("行情登出失败", pRspInfo.ErrorID, pRspInfo.ErrorMsg)
```

### 4.6 `OnRspQryMulticastInstrument(pMulticastInstrument, pRspInfo, nRequestID, bIsLast)`

**触发时机**：`ReqQryMulticastInstrument()` 的响应返回时触发，可能分多条返回。

**常见用途**：处理组播合约查询结果。

**示例**：

```python
def OnRspQryMulticastInstrument(self, pMulticastInstrument, pRspInfo, nRequestID, bIsLast):
    # 中文注释：bIsLast 为 True 表示本次查询响应结束
    if pRspInfo.ErrorID == 0 and pMulticastInstrument is not None:
        print("组播合约", pMulticastInstrument.InstrumentID)
    if bIsLast:
        print("组播合约查询结束")
```

### 4.7 `OnRspError(pRspInfo, nRequestID, bIsLast)`

**触发时机**：请求处理出现错误时触发。

**常见用途**：统一记录异步错误、排查柜台拒绝原因。

**示例**：

```python
def OnRspError(self, pRspInfo, nRequestID, bIsLast):
    # 中文注释：统一记录接口级错误响应
    print("行情接口错误", nRequestID, pRspInfo.ErrorID, pRspInfo.ErrorMsg)
```

### 4.8 `OnRspSubMarketData(pSpecificInstrument, pRspInfo, nRequestID, bIsLast)`

**触发时机**：`SubscribeMarketData()` 的订阅响应返回时触发。

**参数说明**：

| 参数 | 说明 |
| --- | --- |
| `pSpecificInstrument` | 具体合约结构体，通常可读取 `InstrumentID`。 |
| `pRspInfo` | 响应错误信息。 |
| `nRequestID` | 请求编号。底层订阅接口本身没有显式 request_id，具体值以底层回调为准。 |
| `bIsLast` | 是否最后一条响应。 |

**常见用途**：确认每个合约是否订阅成功。

**示例**：

```python
def OnRspSubMarketData(self, pSpecificInstrument, pRspInfo, nRequestID, bIsLast):
    # 中文注释：订阅响应通常按合约分别返回
    instrument = pSpecificInstrument.InstrumentID
    if pRspInfo.ErrorID == 0:
        print("订阅成功", instrument)
    else:
        print("订阅失败", instrument, pRspInfo.ErrorID, pRspInfo.ErrorMsg)
```

### 4.9 `OnRspUnSubMarketData(pSpecificInstrument, pRspInfo, nRequestID, bIsLast)`

**触发时机**：`UnSubscribeMarketData()` 的退订响应返回时触发。

**常见用途**：确认合约退订结果，维护本地订阅集合。

**示例**：

```python
def OnRspUnSubMarketData(self, pSpecificInstrument, pRspInfo, nRequestID, bIsLast):
    # 中文注释：退订成功后从本地订阅集合移除
    instrument = pSpecificInstrument.InstrumentID
    if pRspInfo.ErrorID == 0:
        self.subscribed.discard(instrument)
        print("退订成功", instrument)
```

### 4.10 `OnRspSubForQuoteRsp(pSpecificInstrument, pRspInfo, nRequestID, bIsLast)`

**触发时机**：`SubscribeForQuoteRsp()` 的响应返回时触发。

**常见用途**：确认询价响应订阅结果。

**示例**：

```python
def OnRspSubForQuoteRsp(self, pSpecificInstrument, pRspInfo, nRequestID, bIsLast):
    # 中文注释：记录询价响应订阅结果
    print("订阅询价响应", pSpecificInstrument.InstrumentID, pRspInfo.ErrorID)
```

### 4.11 `OnRspUnSubForQuoteRsp(pSpecificInstrument, pRspInfo, nRequestID, bIsLast)`

**触发时机**：`UnSubscribeForQuoteRsp()` 的响应返回时触发。

**常见用途**：确认询价响应退订结果。

**示例**：

```python
def OnRspUnSubForQuoteRsp(self, pSpecificInstrument, pRspInfo, nRequestID, bIsLast):
    # 中文注释：记录询价响应退订结果
    print("退订询价响应", pSpecificInstrument.InstrumentID, pRspInfo.ErrorID)
```

### 4.12 `OnRtnDepthMarketData(pDepthMarketData)`

**触发时机**：订阅合约后，服务端推送深度行情时触发。

**核心用途**：处理实时行情，是行情接口最重要的回调。

**常见字段**：

| 字段 | 说明 |
| --- | --- |
| `TradingDay` | 交易日。 |
| `InstrumentID` | 合约 ID。 |
| `ExchangeID` | 交易所 ID。 |
| `LastPrice` | 最新价。 |
| `PreSettlementPrice` | 上次结算价。 |
| `PreClosePrice` | 昨收盘价。 |
| `OpenPrice` | 今开盘价。 |
| `HighestPrice` | 最高价。 |
| `LowestPrice` | 最低价。 |
| `Volume` | 数量/成交量。 |
| `Turnover` | 成交金额。 |
| `OpenInterest` | 持仓量。 |
| `ClosePrice` | 今收盘价。 |
| `SettlementPrice` | 本次结算价。 |
| `UpperLimitPrice` | 涨停板价。 |
| `LowerLimitPrice` | 跌停板价。 |
| `UpdateTime` | 最后修改时间。 |
| `UpdateMillisec` | 最后修改毫秒。 |
| `BidPrice1` / `AskPrice1` | 买一价 / 卖一价。 |
| `BidVolume1` / `AskVolume1` | 买一量 / 卖一量。 |
| `BidPrice2`-`BidPrice5` | 买二到买五价。 |
| `AskPrice2`-`AskPrice5` | 卖二到卖五价。 |
| `BidVolume2`-`BidVolume5` | 买二到买五量。 |
| `AskVolume2`-`AskVolume5` | 卖二到卖五量。 |
| `ActionDay` | 业务日期。 |

**示例：打印行情**：

```python
def OnRtnDepthMarketData(self, pDepthMarketData):
    # 中文注释：只打印最常用字段
    print(
        pDepthMarketData.InstrumentID,
        pDepthMarketData.UpdateTime,
        pDepthMarketData.LastPrice,
        pDepthMarketData.BidPrice1,
        pDepthMarketData.AskPrice1,
    )
```

**示例：推入队列，避免回调阻塞**：

```python
from queue import Queue

class MyMdApi(MdApiPy):
    def __init__(self):
        # 中文注释：行情回调只负责快速入队，计算逻辑交给其他线程
        self.tick_queue = Queue(maxsize=10000)

    def OnRtnDepthMarketData(self, tick):
        # 中文注释：高频回调中不要写数据库、不要发网络请求、不要做复杂计算
        if not self.tick_queue.full():
            self.tick_queue.put_nowait({
                "instrument": tick.InstrumentID,
                "time": tick.UpdateTime,
                "last": tick.LastPrice,
                "bid1": tick.BidPrice1,
                "ask1": tick.AskPrice1,
            })
```

**注意事项**：

- 行情推送频率可能很高，回调中应尽量短小。
- 不建议在此回调中直接执行数据库写入、HTTP 请求、复杂计算。
- CTP 浮点字段可能出现极大值表示无效价，实际入库前建议做合法性过滤。
- 夜盘中 `TradingDay`、`ActionDay`、自然日期之间可能不同，应按业务规则处理。

### 4.13 `OnRtnForQuoteRsp(pForQuoteRsp)`

**触发时机**：订阅询价响应后，收到询价通知时触发。

**常见用途**：处理询价业务通知，普通行情采集通常不需要。

**示例**：

```python
def OnRtnForQuoteRsp(self, pForQuoteRsp):
    # 中文注释：收到询价通知后按业务需要处理
    print("收到询价通知", pForQuoteRsp.InstrumentID)
```

## 5. 常见业务场景

### 5.1 只采集行情

推荐逻辑：

1. `OnFrontConnected()` 中登录。
2. `OnRspUserLogin()` 成功后订阅全部目标合约。
3. `OnRtnDepthMarketData()` 中快速转换 tick 并写入队列。
4. 后台消费线程从队列批量写库或写文件。

### 5.2 动态切换订阅合约

推荐逻辑：

1. 维护本地 `subscribed` 集合。
2. 新增合约时调用 `SubscribeMarketData([instrument])`。
3. 删除合约时调用 `UnSubscribeMarketData([instrument])`。
4. 在 `OnRspSubMarketData()` 和 `OnRspUnSubMarketData()` 中根据成功结果更新集合。

### 5.3 断线重连

推荐逻辑：

1. `OnFrontDisconnected()` 中标记 `login = False`。
2. `OnFrontConnected()` 再次触发时重新 `ReqUserLogin()`。
3. `OnRspUserLogin()` 成功后重新订阅目标合约列表。
4. 订阅列表由业务层保存，不依赖底层 API 自动恢复。

示例：

```python
def OnFrontDisconnected(self, nReason):
    # 中文注释：断线时更新状态，底层一般会自动重连
    self.login = False
    print("断线", nReason)


def OnRspUserLogin(self, pRspUserLogin, pRspInfo, nRequestID, bIsLast):
    if pRspInfo.ErrorID == 0:
        # 中文注释：重连登录成功后，重新订阅业务保存的合约列表
        self.login = True
        self.SubscribeMarketData(self.instruments)
```

## 6. 常见问题排查

### 6.1 `Init()` 后没有行情

优先检查：

- 是否已经触发 `OnFrontConnected()`。
- 是否在 `OnRspUserLogin()` 中确认 `ErrorID == 0`。
- 是否收到 `OnRspSubMarketData()`，订阅响应是否成功。
- 合约代码是否正确、是否已过期。
- 当前是否交易时段。
- 前置地址是否为行情前置，而不是交易前置。

### 6.2 `SubscribeMarketData()` 返回 0，但没有推送

`SubscribeMarketData()` 的返回值只说明请求发送到底层 API 成功，不等于服务端订阅成功。应继续查看：

- `OnRspSubMarketData()` 的 `pRspInfo.ErrorID`。
- 合约是否在当前柜台可订阅。
- 市场是否有实时行情。

### 6.3 回调里能不能直接下单或写库

技术上可以调用其他逻辑，但不推荐在行情回调中做耗时操作。建议行情回调只做：

- 复制必要字段。
- 放入线程安全队列。
- 快速返回。

复杂计算、下单决策、数据库写入交给其他线程或事件循环处理。

## 7. 函数速查表

| 类型 | 函数 | 一般使用时机 |
| --- | --- | --- |
| 生命周期 | `Create()` | 实例化后第一步，创建底层 API。 |
| 生命周期 | `Init()` | 注册前置地址后，启动接口线程。 |
| 生命周期 | `Join()` | 程序末尾阻塞保活。 |
| 生命周期 | `Release()` | 程序退出或销毁实例时。 |
| 连接 | `RegisterFront()` | 使用固定行情前置地址时。 |
| 连接 | `RegisterNameServer()` | 使用名字服务器发现前置地址时。 |
| 连接 | `RegisterFensUserInfo()` | 名字服务器需要用户信息时。 |
| 登录 | `ReqUserLogin()` | `OnFrontConnected()` 中发起登录。 |
| 登录 | `ReqUserLogout()` | 程序退出或切换账号前。 |
| 查询 | `ReqQryMulticastInstrument()` | 组播行情场景查询组播合约。 |
| 状态 | `GetTradingDay()` | 登录成功后获取当前交易日。 |
| 订阅 | `SubscribeMarketData()` | 登录成功后订阅深度行情。 |
| 订阅 | `UnSubscribeMarketData()` | 不再关注合约时退订。 |
| 询价 | `SubscribeForQuoteRsp()` | 需要询价响应通知时。 |
| 询价 | `UnSubscribeForQuoteRsp()` | 不再需要询价响应通知时。 |
| 回调 | `OnFrontConnected()` | 前置连接成功，通常发起登录。 |
| 回调 | `OnFrontDisconnected()` | 前置断开，记录并等待重连。 |
| 回调 | `OnHeartBeatWarning()` | 网络心跳异常告警。 |
| 回调 | `OnRspUserLogin()` | 登录响应，成功后订阅。 |
| 回调 | `OnRspUserLogout()` | 登出响应。 |
| 回调 | `OnRspQryMulticastInstrument()` | 组播合约查询响应。 |
| 回调 | `OnRspError()` | 异步错误响应。 |
| 回调 | `OnRspSubMarketData()` | 行情订阅响应。 |
| 回调 | `OnRspUnSubMarketData()` | 行情退订响应。 |
| 回调 | `OnRspSubForQuoteRsp()` | 询价订阅响应。 |
| 回调 | `OnRspUnSubForQuoteRsp()` | 询价退订响应。 |
| 回调 | `OnRtnDepthMarketData()` | 深度行情推送。 |
| 回调 | `OnRtnForQuoteRsp()` | 询价通知推送。 |
