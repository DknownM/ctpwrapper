# TraderApiPy 函数字典（交易接口备查）

本文档面向 `ctpwrapper.TraderApiPy` 的日常开发与排障，按“连接生命周期、认证登录、查询、交易、撤单、回调处理”的顺序整理常用函数。交易接口与行情接口一样采用 **主动请求 + 异步回调** 模式：你主动调用 `Req...` 方法，CTP 柜台随后通过 `OnRsp...`、`OnRtn...`、`OnErrRtn...` 回调异步返回结果或推送状态。

> 重要提示：本文示例以 SimNow 常见参数为例。真实生产交易前，请以期货公司给出的 `BrokerID`、前置地址、`AppID`、`AuthCode`、合约交易规则、交易所规则和风控要求为准。

## 1. 核心工作模式

### 1.1 主动请求 + 异步回调

`TraderApiPy` 的函数大致分为两类：

- **主动请求函数**：例如 `ReqAuthenticate()`、`ReqUserLogin()`、`ReqQryTradingAccount()`、`ReqOrderInsert()`、`ReqOrderAction()`。调用后通常会立即返回一个整数，表示本地调用是否被 API 接受。
- **异步回调函数**：例如 `OnRspAuthenticate()`、`OnRspUserLogin()`、`OnRspQryTradingAccount()`、`OnRtnOrder()`、`OnRtnTrade()`。这些函数需要在你继承 `TraderApiPy` 后按需重写，用来接收柜台响应、报单状态推送、成交推送和错误推送。

请求函数的返回值只表示 **请求是否成功送入 API 队列**，不代表业务成功。业务是否成功要看回调里的 `pRspInfo.ErrorID`、订单回报状态、成交回报等。

### 1.2 推荐调用顺序

1. 创建自定义交易类，继承 `TraderApiPy`。
2. 重写 `OnFrontConnected()`：连接成功后发起 `ReqAuthenticate()`。
3. 重写 `OnRspAuthenticate()`：认证成功后发起 `ReqUserLogin()`。
4. 重写 `OnRspUserLogin()`：登录成功后记录 `FrontID`、`SessionID`、`MaxOrderRef`，再进行结算单确认、查询或交易。
5. 调用 `Create()` 创建 API 实例。
6. 调用 `RegisterFront()` 或 `RegisterNameServer()` 注册交易前置地址。
7. 如需私有流/公共流，在 `Init()` 前调用 `SubscribePrivateTopic()`、`SubscribePublicTopic()`。
8. 调用 `Init()` 启动接口线程。
9. 登录成功后调用 `ReqSettlementInfoConfirm()`、`ReqQry...`、`ReqOrderInsert()`、`ReqOrderAction()` 等。
10. 在 `OnRsp...`、`OnRtn...`、`OnErrRtn...` 回调中处理响应、推送和错误。
11. 退出前调用 `ReqUserLogout()`、`Release()`，或使用 `Join()` 阻塞等待接口线程结束。

### 1.3 请求编号、报单引用和流控

- `nRequestID`：客户端自增的请求编号，用于在响应回调中匹配请求。建议封装 `next_request_id()`，每次请求递增。
- `OrderRef`：报单引用，需要在同一交易会话内唯一。登录响应中的 `MaxOrderRef` 可作为起点，后续递增生成。
- CTP 查询类请求常见流控限制，连续查询太快可能收到流控错误。实盘中建议在查询之间加入 0.2 到 1 秒间隔，或维护请求队列。
- 报单与撤单属于交易请求，必须在登录成功、必要的结算单确认完成、账户状态正常后再发起。

## 2. 最小交易登录示例

```python
# encoding:utf-8
import time
from ctpwrapper import ApiStructure, TraderApiPy


class MyTraderApi(TraderApiPy):
    """自定义交易 API：负责认证、登录与交易回调处理。"""

    def __init__(self, broker_id, user_id, password, app_id, auth_code):
        # 中文注释：保存交易登录和认证参数，生产环境不要硬编码真实密码。
        self.broker_id = broker_id
        self.user_id = user_id
        self.password = password
        self.app_id = app_id
        self.auth_code = auth_code
        self.request_id = 0
        self.login = False
        self.front_id = None
        self.session_id = None
        self.order_ref = 0

    def next_request_id(self):
        # 中文注释：每次请求使用新的请求编号，便于在回调中定位请求。
        self.request_id += 1
        return self.request_id

    def next_order_ref(self):
        # 中文注释：报单引用通常使用数字字符串，长度以 CTP 字段定义为准。
        self.order_ref += 1
        return f"{self.order_ref:012d}"

    def OnFrontConnected(self):
        # 中文注释：连接交易前置成功后，先做客户端认证。
        req = ApiStructure.ReqAuthenticateField(
            BrokerID=self.broker_id,
            UserID=self.user_id,
            AppID=self.app_id,
            AuthCode=self.auth_code,
        )
        self.ReqAuthenticate(req, self.next_request_id())

    def OnRspAuthenticate(self, pRspAuthenticateField, pRspInfo, nRequestID, bIsLast):
        # 中文注释：认证成功后再登录。
        if pRspInfo.ErrorID == 0:
            req = ApiStructure.ReqUserLoginField(
                BrokerID=self.broker_id,
                UserID=self.user_id,
                Password=self.password,
            )
            self.ReqUserLogin(req, self.next_request_id())
        else:
            print("认证失败", pRspInfo)

    def OnRspUserLogin(self, pRspUserLogin, pRspInfo, nRequestID, bIsLast):
        # 中文注释：登录成功后才可以查询、确认结算单、报单或撤单。
        if pRspInfo.ErrorID == 0:
            print("交易登录成功", pRspUserLogin)
            self.login = True
            self.front_id = pRspUserLogin.FrontID
            self.session_id = pRspUserLogin.SessionID
            self.order_ref = int(pRspUserLogin.MaxOrderRef or "0")
        else:
            print("交易登录失败", pRspInfo)

    def OnRspQryTradingAccount(self, pTradingAccount, pRspInfo, nRequestID, bIsLast):
        # 中文注释：查询资金账户响应，bIsLast=True 表示本次查询响应结束。
        print("资金账户", pTradingAccount, "是否最后一条", bIsLast)


trader = MyTraderApi("9999", "你的账号", "你的密码", "你的AppID", "你的AuthCode")
trader.Create()
trader.RegisterFront("tcp://交易前置地址:端口")

# 中文注释：私有流和公共流订阅必须在 Init 前调用。
trader.SubscribePrivateTopic(2, 0)  # 2 表示 THOST_TERT_QUICK，只接收登录后的私有流。
trader.SubscribePublicTopic(2)      # 2 表示 THOST_TERT_QUICK，只接收登录后的公共流。

trader.Init()

while not trader.login:
    time.sleep(0.2)

# 中文注释：登录成功后查询资金账户。
req = ApiStructure.QryTradingAccountField(
    BrokerID="9999",
    InvestorID="你的账号",
    BizType="1",
)
trader.ReqQryTradingAccount(req, trader.next_request_id())

trader.Join()
```

## 3. 生命周期与连接函数

| 函数 | 逻辑/用途 | 什么时候使用 | 示例 |
| --- | --- | --- | --- |
| `Create(pszFlowPath="", bIsProductionMode=True)` | 创建交易 API 实例，并指定流文件目录。流文件用于保存私有流、公共流续传信息。 | 注册前置地址和 `Init()` 前调用一次。多账号通常每个账号独立实例和独立流目录。 | `trader.Create("./flow/trader/")` |
| `RegisterFront(pszFrontAddress)` | 注册交易前置机地址。 | 已知固定交易前置地址时使用，必须在 `Init()` 前调用。 | `trader.RegisterFront("tcp://127.0.0.1:17001")` |
| `RegisterNameServer(pszNsAddress)` | 注册名字服务器地址，名字服务器可下发可用前置。 | 期货公司提供名字服务器而非固定前置时使用，优先级高于 `RegisterFront()`。 | `trader.RegisterNameServer("tcp://127.0.0.1:12001")` |
| `RegisterFensUserInfo(pFensUserInfo)` | 注册 FENS 用户信息，配合名字服务器使用。 | 使用 FENS/名字服务器接入模式时，在 `Init()` 前调用。 | `trader.RegisterFensUserInfo(ApiStructure.FensUserInfoField(...))` |
| `SubscribePrivateTopic(nResumeType, nSeqNo)` | 订阅私有流，私有流包含订单、成交等与本用户相关的数据。 | 必须在 `Init()` 前调用；需要接收订单和成交推送时建议调用。 | `trader.SubscribePrivateTopic(2, 0)` |
| `SubscribePublicTopic(nResumeType)` | 订阅公共流，公共流包含合约状态、公告等公共数据。 | 必须在 `Init()` 前调用；需要合约状态、公告推送时调用。 | `trader.SubscribePublicTopic(2)` |
| `Init()` | 初始化接口并启动后台线程，开始连接前置。 | 前置、流订阅配置完成后调用。 | `trader.Init()` |
| `Join()` | 阻塞当前线程直到 API 线程退出。 | 示例程序或常驻进程中用于防止主线程退出。 | `trader.Join()` |
| `Release()` | 释放 API 资源。 | 程序退出或需要销毁实例时调用。 | `trader.Release()` |
| `GetTradingDay()` | 获取当前交易日字符串。 | 登录成功后获取才可靠，可用于日志、文件分区或交易日判断。 | `day = trader.GetTradingDay()` |
| `GetFrontInfo(pFrontInfo)` | 获取当前连接前置信息。 | 连接后排障或记录前置流控信息时使用。 | `trader.GetFrontInfo(front_info)` |

`nResumeType` 常见取值：`0` 表示从本交易日开始重传，`1` 表示从上次收到处续传，`2` 表示只接收登录后的新流。普通策略和测试环境通常使用 `2`，灾备恢复或需要补齐当日回报时再考虑 `0`/`1`。

## 4. 认证、登录、登出与密码函数

| 函数 | 请求结构体 | 主要回调 | 逻辑/用途 | 什么时候使用 |
| --- | --- | --- | --- | --- |
| `ReqAuthenticate(req, request_id)` | `ReqAuthenticateField` | `OnRspAuthenticate` | 客户端认证，校验 `BrokerID`、`UserID`、`AppID`、`AuthCode`。 | 多数 CTP 环境登录前必须先认证。 |
| `ReqUserLogin(req, request_id)` | `ReqUserLoginField` | `OnRspUserLogin` | 用户登录，获取 `FrontID`、`SessionID`、`MaxOrderRef` 等会话信息。 | 认证成功后调用。 |
| `ReqUserLogout(req, request_id)` | `UserLogoutField` | `OnRspUserLogout` | 用户登出。 | 程序正常退出、切换账号前使用。 |
| `ReqUserPasswordUpdate(req, request_id)` | `UserPasswordUpdateField` | `OnRspUserPasswordUpdate` | 修改交易用户密码。 | 柜台要求改密或用户主动改密时使用。 |
| `ReqTradingAccountPasswordUpdate(req, request_id)` | `TradingAccountPasswordUpdateField` | `OnRspTradingAccountPasswordUpdate` | 修改资金账户密码。 | 银期或资金账户密码维护场景使用。 |
| `ReqUserAuthMethod(req, request_id)` | `ReqUserAuthMethodField` | `OnRspUserAuthMethod` | 查询当前用户支持的认证方式。 | 需要验证码、短信、OTP 等扩展登录流程前使用。 |
| `ReqGenUserCaptcha(req, request_id)` | `ReqGenUserCaptchaField` | `OnRspGenUserCaptcha` | 获取图形验证码。 | 柜台要求图形验证码登录时使用。 |
| `ReqGenUserText(req, request_id)` | `ReqGenUserTextField` | `OnRspGenUserText` | 获取短信验证码。 | 柜台要求短信验证码登录时使用。 |
| `ReqUserLoginWithCaptcha(req, request_id)` | `ReqUserLoginWithCaptchaField` | `OnRspUserLogin` | 带图形验证码登录。 | 查询认证方式后确认需要图片验证码时使用。 |
| `ReqUserLoginWithText(req, request_id)` | `ReqUserLoginWithTextField` | `OnRspUserLogin` | 带短信验证码登录。 | 查询认证方式后确认需要短信验证码时使用。 |
| `ReqUserLoginWithOTP(req, request_id)` | `ReqUserLoginWithOTPField` | `OnRspUserLogin` | 带动态口令登录。 | 需要 OTP 登录的柜台环境使用。 |

认证登录示例：

```python
# 中文注释：连接成功后发起认证。
def OnFrontConnected(self):
    req = ApiStructure.ReqAuthenticateField(
        BrokerID=self.broker_id,
        UserID=self.user_id,
        AppID=self.app_id,
        AuthCode=self.auth_code,
    )
    self.ReqAuthenticate(req, self.next_request_id())

# 中文注释：认证通过后发起登录。
def OnRspAuthenticate(self, pRspAuthenticateField, pRspInfo, nRequestID, bIsLast):
    if pRspInfo.ErrorID == 0:
        req = ApiStructure.ReqUserLoginField(
            BrokerID=self.broker_id,
            UserID=self.user_id,
            Password=self.password,
        )
        self.ReqUserLogin(req, self.next_request_id())
```

## 5. 终端信息采集相关函数

| 函数 | 逻辑/用途 | 什么时候使用 |
| --- | --- | --- |
| `RegisterUserSystemInfo(pUserSystemInfo)` | 注册用户终端信息，用于中继服务器多连接模式。 | 终端认证成功后、用户登录前调用。 |
| `SubmitUserSystemInfo(pUserSystemInfo)` | 上报用户终端信息，用于中继服务器操作员登录模式。 | 操作员登录后，可多次上报客户终端信息。 |
| `RegisterWechatUserSystemInfo(pUserSystemInfo)` | 注册微信小程序等应用的用户终端信息。 | 微信小程序等接入模式，登录前注册。 |
| `SubmitWechatUserSystemInfo(pUserSystemInfo)` | 上报微信小程序等应用的用户终端信息。 | 微信小程序等接入模式，登录后上报。 |

这类函数是否必需取决于期货公司接入要求。普通 SimNow 示例通常不需要；生产环境如柜台要求采集终端信息，应按期货公司文档构造对应结构体。

## 6. 结算单函数

| 函数 | 请求结构体 | 主要回调 | 逻辑/用途 | 什么时候使用 |
| --- | --- | --- | --- | --- |
| `ReqQrySettlementInfo(req, request_id)` | `QrySettlementInfoField` | `OnRspQrySettlementInfo` | 查询结算单内容。 | 登录后、确认前查看上一交易日结算单。 |
| `ReqSettlementInfoConfirm(req, request_id)` | `SettlementInfoConfirmField` | `OnRspSettlementInfoConfirm` | 确认结算单。 | 很多柜台要求每日首次登录后确认，否则不能交易。 |
| `ReqQrySettlementInfoConfirm(req, request_id)` | `QrySettlementInfoConfirmField` | `OnRspQrySettlementInfoConfirm` | 查询结算单确认记录。 | 判断当日是否已经确认过结算单。 |

结算单确认示例：

```python
# 中文注释：登录成功后确认结算单，部分柜台不确认不能报单。
confirm = ApiStructure.SettlementInfoConfirmField(
    BrokerID="9999",
    InvestorID="你的账号",
)
trader.ReqSettlementInfoConfirm(confirm, trader.next_request_id())
```

## 7. 常用查询函数字典

查询类函数一般遵循同一模式：构造 `Qry...Field`，调用 `ReqQry...()`，在 `OnRspQry...()` 中逐条接收结果。若结果有多条，回调会触发多次，`bIsLast=True` 表示本次查询结束。

| 函数 | 请求结构体 | 主要回调 | 用途 | 常用时机 |
| --- | --- | --- | --- | --- |
| `ReqQryTradingAccount` | `QryTradingAccountField` | `OnRspQryTradingAccount` | 查询资金账户、可用资金、保证金、手续费等。 | 登录后初始化账户状态；下单前检查资金。 |
| `ReqQryInvestorPosition` | `QryInvestorPositionField` | `OnRspQryInvestorPosition` | 查询合约维度持仓。 | 策略启动初始化持仓；报单前判断可平数量。 |
| `ReqQryInvestorPositionDetail` | `QryInvestorPositionDetailField` | `OnRspQryInvestorPositionDetail` | 查询持仓明细。 | 需要区分开仓日期、逐笔持仓、平今平昨逻辑时。 |
| `ReqQryOrder` | `QryOrderField` | `OnRspQryOrder` | 查询委托。 | 启动后恢复未完成委托；排查订单状态。 |
| `ReqQryTrade` | `QryTradeField` | `OnRspQryTrade` | 查询成交。 | 启动后恢复当日成交；对账。 |
| `ReqQryInvestor` | `QryInvestorField` | `OnRspQryInvestor` | 查询投资者信息。 | 初始化账号资料、排障。 |
| `ReqQryTradingCode` | `QryTradingCodeField` | `OnRspQryTradingCode` | 查询交易编码。 | 检查各交易所交易编码是否开通。 |
| `ReqQryInstrument` | `QryInstrumentField` | `OnRspQryInstrument` | 查询合约基础信息。 | 启动时加载合约、价格跳动、合约乘数、交割月等。 |
| `ReqQryDepthMarketData` | `QryDepthMarketDataField` | `OnRspQryDepthMarketData` | 查询最新深度行情快照。 | 交易接口侧临时查询行情快照；常规行情建议用行情 API。 |
| `ReqQryInstrumentMarginRate` | `QryInstrumentMarginRateField` | `OnRspQryInstrumentMarginRate` | 查询合约保证金率。 | 下单前估算保证金。 |
| `ReqQryInstrumentCommissionRate` | `QryInstrumentCommissionRateField` | `OnRspQryInstrumentCommissionRate` | 查询合约手续费率。 | 成本计算、策略回测参数校验。 |
| `ReqQryInstrumentOrderCommRate` | `QryInstrumentOrderCommRateField` | `OnRspQryInstrumentOrderCommRate` | 查询报单手续费率。 | 需要细分报单相关费用时。 |
| `ReqQryExchange` | `QryExchangeField` | `OnRspQryExchange` | 查询交易所信息。 | 初始化基础资料。 |
| `ReqQryProduct` | `QryProductField` | `OnRspQryProduct` | 查询品种信息。 | 初始化品种、交易所、产品类型。 |
| `ReqQryProductGroup` | `QryProductGroupField` | `OnRspQryProductGroup` | 查询产品组。 | 组合保证金、产品组风控场景。 |
| `ReqQryExchangeMarginRate` | `QryExchangeMarginRateField` | `OnRspQryExchangeMarginRate` | 查询交易所保证金率。 | 风控或保证金参数核对。 |
| `ReqQryExchangeMarginRateAdjust` | `QryExchangeMarginRateAdjustField` | `OnRspQryExchangeMarginRateAdjust` | 查询保证金率调整。 | 检查柜台保证金调整参数。 |
| `ReqQryExchangeRate` | `QryExchangeRateField` | `OnRspQryExchangeRate` | 查询汇率。 | 外币或特定品种资金折算。 |
| `ReqQryMaxOrderVolume` | `QryMaxOrderVolumeField` | `OnRspQryMaxOrderVolume` | 查询最大报单数量。 | 大单拆单前判断单笔最大量。 |
| `ReqQryUserSession` | `QryUserSessionField` | `OnRspQryUserSession` | 查询用户会话。 | 运维排障、会话跟踪。 |
| `ReqQryNotice` | `QryNoticeField` | `OnRspQryNotice` | 查询通知。 | 登录后查看柜台通知。 |
| `ReqQryTradingNotice` | `QryTradingNoticeField` | `OnRspQryTradingNotice` | 查询交易通知。 | 接收或补查交易通知。 |
| `ReqQryBrokerTradingParams` | `QryBrokerTradingParamsField` | `OnRspQryBrokerTradingParams` | 查询经纪公司交易参数。 | 初始化柜台参数。 |
| `ReqQryBrokerTradingAlgos` | `QryBrokerTradingAlgosField` | `OnRspQryBrokerTradingAlgos` | 查询经纪公司交易算法。 | 算法或柜台规则检查。 |

资金、持仓、合约查询示例：

```python
# 中文注释：查询投资者信息。
qry_investor = ApiStructure.QryInvestorField(
    BrokerID="9999",
    InvestorID="你的账号",
)
trader.ReqQryInvestor(qry_investor, trader.next_request_id())

# 中文注释：查询持仓。InstrumentID 可为空，具体支持情况以柜台为准。
qry_position = ApiStructure.QryInvestorPositionField(
    BrokerID="9999",
    InvestorID="你的账号",
    InstrumentID="rb2410",
)
trader.ReqQryInvestorPosition(qry_position, trader.next_request_id())

# 中文注释：查询合约基础资料。
qry_instrument = ApiStructure.QryInstrumentField(
    InstrumentID="rb2410",
)
trader.ReqQryInstrument(qry_instrument, trader.next_request_id())
```

## 8. 报单、撤单与订单状态

### 8.1 报单函数

| 函数 | 请求结构体 | 主要响应/推送 | 逻辑/用途 | 常用时机 |
| --- | --- | --- | --- | --- |
| `ReqOrderInsert(req, request_id)` | `InputOrderField` | `OnRspOrderInsert`、`OnRtnOrder`、`OnRtnTrade`、`OnErrRtnOrderInsert` | 普通报单录入。 | 策略发出开仓、平仓、平今、平昨等交易指令时。 |
| `ReqParkedOrderInsert(req, request_id)` | `ParkedOrderField` | `OnRspParkedOrderInsert` | 预埋单录入。 | 非交易时段预先提交，待条件满足或人工触发。 |
| `ReqRemoveParkedOrder(req, request_id)` | `RemoveParkedOrderField` | `OnRspRemoveParkedOrder` | 删除预埋单。 | 预埋单不再需要时。 |
| `ReqQryParkedOrder(req, request_id)` | `QryParkedOrderField` | `OnRspQryParkedOrder` | 查询预埋单。 | 恢复或管理预埋单时。 |

限价买开示例：

```python
# 中文注释：构造限价买开报单请求。真实交易前必须确认合约、方向、开平、价格、数量和风控。
order = ApiStructure.InputOrderField(
    BrokerID="9999",
    InvestorID="你的账号",
    InstrumentID="rb2410",
    OrderRef=trader.next_order_ref(),
    UserID="你的账号",
    Direction="0",              # 中文注释：0=买，1=卖。
    CombOffsetFlag="0",         # 中文注释：0=开仓，1=平仓，3=平今等，以 CTP 枚举为准。
    CombHedgeFlag="1",          # 中文注释：1=投机。
    LimitPrice=3500.0,
    VolumeTotalOriginal=1,
    OrderPriceType="2",         # 中文注释：2=限价。
    TimeCondition="3",          # 中文注释：3=当日有效。
    VolumeCondition="1",        # 中文注释：1=任意数量。
    MinVolume=1,
    ContingentCondition="1",    # 中文注释：1=立即。
    ForceCloseReason="0",       # 中文注释：0=非强平。
    IsAutoSuspend=0,
)

# 中文注释：业务结果在 OnRspOrderInsert / OnRtnOrder / OnRtnTrade / OnErrRtnOrderInsert 中处理。
ret = trader.ReqOrderInsert(order, trader.next_request_id())
print("本地报单调用返回值", ret)
```

### 8.2 撤单函数

| 函数 | 请求结构体 | 主要响应/推送 | 逻辑/用途 | 常用时机 |
| --- | --- | --- | --- | --- |
| `ReqOrderAction(req, request_id)` | `InputOrderActionField` | `OnRspOrderAction`、`OnRtnOrder`、`OnErrRtnOrderAction` | 撤销普通委托。 | 未成交、部成未撤订单需要撤单时。 |
| `ReqParkedOrderAction(req, request_id)` | `ParkedOrderActionField` | `OnRspParkedOrderAction` | 预埋撤单录入。 | 需要预埋撤单指令时。 |
| `ReqRemoveParkedOrderAction(req, request_id)` | `RemoveParkedOrderActionField` | `OnRspRemoveParkedOrderAction` | 删除预埋撤单。 | 预埋撤单不再需要时。 |
| `ReqBatchOrderAction(req, request_id)` | `InputBatchOrderActionField` | `OnRspBatchOrderAction`、`OnErrRtnBatchOrderAction` | 批量撤单。 | 按交易所或条件批量撤单，需确认柜台支持。 |

撤单通常需要能唯一定位原委托。常见做法是使用 `FrontID + SessionID + OrderRef`，也可根据柜台和交易所回报使用 `ExchangeID + OrderSysID`。

```python
# 中文注释：使用 FrontID、SessionID、OrderRef 撤单。
action = ApiStructure.InputOrderActionField(
    BrokerID="9999",
    InvestorID="你的账号",
    InstrumentID="rb2410",
    UserID="你的账号",
    ActionFlag="0",             # 中文注释：0=删除，即撤单。
    FrontID=trader.front_id,
    SessionID=trader.session_id,
    OrderRef="000000000001",
)
trader.ReqOrderAction(action, trader.next_request_id())
```

### 8.3 报单回调的处理逻辑

- `OnRspOrderInsert`：报单录入请求的同步错误响应。并不是所有拒单都会在这里出现。
- `OnErrRtnOrderInsert`：报单错误回报，通常表示柜台或交易所拒绝。
- `OnRtnOrder`：订单状态推送，成功报单、排队、部分成交、全部成交、撤单、拒单等状态都应以该回调更新本地订单状态机。
- `OnRtnTrade`：成交推送。更新成交明细、持仓、资金和策略状态时必须处理。
- `OnRspOrderAction` / `OnErrRtnOrderAction`：撤单错误响应/错误回报。

建议生产代码以 `OnRtnOrder` 和 `OnRtnTrade` 为订单状态和成交状态的主数据源，`OnRsp...` 与 `OnErrRtn...` 主要用于记录错误和辅助排障。

## 9. 期权、询价、报价、组合与高级交易函数

| 函数 | 请求结构体 | 主要回调 | 用途 | 常用时机 |
| --- | --- | --- | --- | --- |
| `ReqExecOrderInsert` | `InputExecOrderField` | `OnRspExecOrderInsert`、`OnRtnExecOrder` | 期权执行宣告录入。 | 期权行权/履约相关业务。 |
| `ReqExecOrderAction` | `InputExecOrderActionField` | `OnRspExecOrderAction`、`OnErrRtnExecOrderAction` | 期权执行宣告撤销。 | 撤销未完成执行宣告。 |
| `ReqForQuoteInsert` | `InputForQuoteField` | `OnRspForQuoteInsert`、`OnRtnForQuoteRsp` | 询价录入。 | 期权或做市询价业务。 |
| `ReqQuoteInsert` | `InputQuoteField` | `OnRspQuoteInsert`、`OnRtnQuote` | 报价录入。 | 做市商报价业务。 |
| `ReqQuoteAction` | `InputQuoteActionField` | `OnRspQuoteAction`、`OnErrRtnQuoteAction` | 报价撤销。 | 撤销做市报价。 |
| `ReqOptionSelfCloseInsert` | `InputOptionSelfCloseField` | `OnRspOptionSelfCloseInsert`、`OnRtnOptionSelfClose` | 期权自对冲录入。 | 期权自对冲业务。 |
| `ReqOptionSelfCloseAction` | `InputOptionSelfCloseActionField` | `OnRspOptionSelfCloseAction`、`OnErrRtnOptionSelfCloseAction` | 期权自对冲撤销。 | 撤销自对冲申请。 |
| `ReqCombActionInsert` | `InputCombActionField` | `OnRspCombActionInsert`、`OnRtnCombAction` | 组合指令录入。 | 组合持仓申请、拆分等业务。 |
| `ReqQryExecOrder` | `QryExecOrderField` | `OnRspQryExecOrder` | 查询执行宣告。 | 期权执行宣告恢复/对账。 |
| `ReqQryForQuote` | `QryForQuoteField` | `OnRspQryForQuote` | 查询询价。 | 做市/询价业务恢复。 |
| `ReqQryQuote` | `QryQuoteField` | `OnRspQryQuote` | 查询报价。 | 做市报价恢复。 |
| `ReqQryOptionSelfClose` | `QryOptionSelfCloseField` | `OnRspQryOptionSelfClose` | 查询期权自对冲。 | 自对冲业务恢复。 |
| `ReqQryCombAction` | `QryCombActionField` | `OnRspQryCombAction` | 查询组合指令。 | 组合业务恢复。 |
| `ReqQryCombInstrumentGuard` | `QryCombInstrumentGuardField` | `OnRspQryCombInstrumentGuard` | 查询组合合约安全系数。 | 组合保证金或组合策略风控。 |

这些函数一般不是普通期货策略的第一优先级，只有在期权、做市商、询价、组合业务明确需要时使用。使用前务必确认交易权限、柜台支持情况和字段填法。

## 10. 银期转账与银行相关函数

| 函数 | 请求结构体 | 主要回调/推送 | 用途 | 常用时机 |
| --- | --- | --- | --- | --- |
| `ReqQryTransferBank` | `QryTransferBankField` | `OnRspQryTransferBank` | 查询可转账银行。 | 初始化银期银行列表。 |
| `ReqQryTransferSerial` | `QryTransferSerialField` | `OnRspQryTransferSerial` | 查询转账流水。 | 对账、排查转账状态。 |
| `ReqQryAccountregister` | `QryAccountregisterField` | `OnRspQryAccountregister` | 查询银期签约关系。 | 检查银行卡签约状态。 |
| `ReqQryContractBank` | `QryContractBankField` | `OnRspQryContractBank` | 查询签约银行。 | 银期业务初始化。 |
| `ReqQueryCFMMCTradingAccountToken` | `QueryCFMMCTradingAccountTokenField` | `OnRspQueryCFMMCTradingAccountToken`、`OnRtnCFMMCTradingAccountToken` | 查询保证金监控中心令牌。 | 需要 CFMMC 令牌场景。 |
| `ReqFromBankToFutureByFuture` | `ReqTransferField` | `OnRspFromBankToFutureByFuture`、相关 `OnRtn...`/`OnErrRtn...` | 银行转期货。 | 入金。 |
| `ReqFromFutureToBankByFuture` | `ReqTransferField` | `OnRspFromFutureToBankByFuture`、相关 `OnRtn...`/`OnErrRtn...` | 期货转银行。 | 出金。 |
| `ReqQueryBankAccountMoneyByFuture` | `ReqQueryAccountField` | `OnRspQueryBankAccountMoneyByFuture`、`OnRtnQueryBankBalanceByFuture` | 查询银行余额。 | 出入金前检查银行余额。 |

银期转账涉及真实资金，生产环境必须加入人工确认、权限控制、日志留痕和异常重试保护。

## 11. 投资单元、二级代理、做市和风控参数查询

下列函数主要用于机构账户、二级代理、做市商或组合保证金/风控模型相关场景。普通个人期货策略通常较少使用，但在需要完整柜台资料或机构风控时很有价值。

| 函数 | 请求结构体 | 主要回调 | 用途 |
| --- | --- | --- | --- |
| `ReqQryInvestUnit` | `QryInvestUnitField` | `OnRspQryInvestUnit` | 查询投资单元。 |
| `ReqQrySecAgentTradingAccount` | `QryTradingAccountField` | `OnRspQrySecAgentTradingAccount` | 查询二级代理资金账户。 |
| `ReqQrySecAgentCheckMode` | `QrySecAgentCheckModeField` | `OnRspQrySecAgentCheckMode` | 查询二级代理检查模式。 |
| `ReqQrySecAgentTradeInfo` | `QrySecAgentTradeInfoField` | `OnRspQrySecAgentTradeInfo` | 查询二级代理交易信息。 |
| `ReqQrySecAgentACIDMap` | `QrySecAgentACIDMapField` | `OnRspQrySecAgentACIDMap` | 查询二级代理资金账户映射。 |
| `ReqQryMMInstrumentCommissionRate` | `QryMMInstrumentCommissionRateField` | `OnRspQryMMInstrumentCommissionRate` | 查询做市商合约手续费率。 |
| `ReqQryMMOptionInstrCommRate` | `QryMMOptionInstrCommRateField` | `OnRspQryMMOptionInstrCommRate` | 查询做市商期权手续费率。 |
| `ReqQryOptionInstrTradeCost` | `QryOptionInstrTradeCostField` | `OnRspQryOptionInstrTradeCost` | 查询期权交易成本。 |
| `ReqQryOptionInstrCommRate` | `QryOptionInstrCommRateField` | `OnRspQryOptionInstrCommRate` | 查询期权手续费率。 |
| `ReqQryProductExchRate` | `QryProductExchRateField` | `OnRspQryProductExchRate` | 查询产品汇率。 |
| `ReqQryClassifiedInstrument` | `QryClassifiedInstrumentField` | `OnRspQryClassifiedInstrument` | 查询分类合约。 |
| `ReqQryCombPromotionParam` | `QryCombPromotionParamField` | `OnRspQryCombPromotionParam` | 查询组合优惠参数。 |
| `ReqQryRiskSettleInvstPosition` | `QryRiskSettleInvstPositionField` | `OnRspQryRiskSettleInvstPosition` | 查询风险结算投资者持仓。 |
| `ReqQryRiskSettleProductStatus` | `QryRiskSettleProductStatusField` | `OnRspQryRiskSettleProductStatus` | 查询风险结算产品状态。 |
| `ReqQryTraderOffer` | `QryTraderOfferField` | `OnRspQryTraderOffer` | 查询交易员报盘机。 |

## 12. 组合保证金、SPBM、SPMM、RCAMS、RULE 与价差申请函数

这些函数通常服务于新一代组合保证金、期权组合、价差申请、套保确认等高级业务。是否可用取决于 CTP 版本、交易所、柜台和账户权限。

| 分类 | 函数 | 用途 |
| --- | --- | --- |
| SPBM 查询 | `ReqQrySPBMFutureParameter`、`ReqQrySPBMOptionParameter`、`ReqQrySPBMIntraParameter`、`ReqQrySPBMInterParameter`、`ReqQrySPBMPortfDefinition`、`ReqQrySPBMInvestorPortfDef`、`ReqQrySPBMAddOnInterParameter` | 查询 SPBM 组合保证金相关参数和投资者组合定义。 |
| 投资者组合保证金 | `ReqQryInvestorPortfMarginRatio`、`ReqQryInvestorProdSPBMDetail`、`ReqQryInvestorProdRCAMSMargin`、`ReqQryInvestorProdRULEMargin`、`ReqQryInvestorPortfSetting` | 查询投资者组合保证金比例、产品明细和设置。 |
| SPMM 查询 | `ReqQryInvestorCommoditySPMMMargin`、`ReqQryInvestorCommodityGroupSPMMMargin`、`ReqQrySPMMInstParam`、`ReqQrySPMMProductParam` | 查询商品/商品组 SPMM 保证金和参数。 |
| RCAMS 查询 | `ReqQryRCAMSCombProductInfo`、`ReqQryRCAMSInstrParameter`、`ReqQryRCAMSIntraParameter`、`ReqQryRCAMSInterParameter`、`ReqQryRCAMSShortOptAdjustParam`、`ReqQryRCAMSInvestorCombPosition` | 查询 RCAMS 组合产品、合约参数、跨/内品种参数、空头期权调整、组合持仓。 |
| RULE 查询 | `ReqQryRULEInstrParameter`、`ReqQryRULEIntraParameter`、`ReqQryRULEInterParameter` | 查询 RULE 风控/组合保证金参数。 |
| 组合腿与互抵 | `ReqQryCombLeg`、`ReqOffsetSetting`、`ReqCancelOffsetSetting`、`ReqQryOffsetSetting` | 查询组合腿，申请/取消/查询互抵设置。 |
| 价差申请 | `ReqGenSMSCode`、`ReqSpdApply`、`ReqSpdApplyAction`、`ReqQrySpdApply` | 生成短信验证码、价差申请、价差申请撤销、查询价差申请。 |
| 套保确认 | `ReqHedgeCfm`、`ReqHedgeCfmAction`、`ReqQryHedgeCfm` | 套保确认、撤销和查询。 |

高级函数示例格式与普通请求一致：构造对应 `ApiStructure.*Field`，调用 `Req...()`，在对应 `OnRsp...`、`OnRtn...`、`OnErrRtn...` 回调里处理结果。

## 13. 常用回调函数字典

### 13.1 连接与通用错误回调

| 回调 | 触发时机 | 建议处理 |
| --- | --- | --- |
| `OnFrontConnected()` | 与交易前置建立 TCP 连接。 | 发起认证或登录；记录连接成功日志。 |
| `OnFrontDisconnected(nReason)` | 与交易前置断开。 | 标记离线，暂停发单；等待 API 自动重连；记录 `nReason`。 |
| `OnHeartBeatWarning(nTimeLapse)` | 心跳超时警告。 | 记录告警，必要时暂停交易。 |
| `OnRspError(pRspInfo, nRequestID, bIsLast)` | 通用错误响应。 | 统一记录错误码和错误信息，关联请求编号。 |
| `OnRtnPrivateSeqNo(pSettlementInfo)` | 私有流序号通知。 | 需要精细流恢复时记录。 |

### 13.2 登录与查询回调

| 回调 | 对应请求 | 触发时机 | 建议处理 |
| --- | --- | --- | --- |
| `OnRspAuthenticate` | `ReqAuthenticate` | 认证响应。 | 成功后登录；失败时停止后续交易。 |
| `OnRspUserLogin` | `ReqUserLogin` / 验证码登录函数 | 登录响应。 | 保存 `FrontID`、`SessionID`、`MaxOrderRef`，进入可查询/可交易状态。 |
| `OnRspUserLogout` | `ReqUserLogout` | 登出响应。 | 标记退出或允许重新登录。 |
| `OnRspSettlementInfoConfirm` | `ReqSettlementInfoConfirm` | 结算单确认响应。 | 成功后允许交易；失败时记录原因。 |
| `OnRspQryTradingAccount` | `ReqQryTradingAccount` | 资金查询响应。 | 更新资金快照，`bIsLast` 后完成本次查询。 |
| `OnRspQryInvestorPosition` | `ReqQryInvestorPosition` | 持仓查询响应。 | 聚合多条持仓记录，`bIsLast` 后替换本地持仓快照。 |
| `OnRspQryOrder` | `ReqQryOrder` | 委托查询响应。 | 恢复当日订单状态。 |
| `OnRspQryTrade` | `ReqQryTrade` | 成交查询响应。 | 恢复当日成交明细。 |
| `OnRspQryInstrument` | `ReqQryInstrument` | 合约查询响应。 | 缓存合约基础资料。 |

### 13.3 订单、成交和交易错误回调

| 回调 | 触发时机 | 建议处理 |
| --- | --- | --- |
| `OnRspOrderInsert(pInputOrder, pRspInfo, nRequestID, bIsLast)` | 报单录入请求错误响应。 | 记录错误；将对应本地订单标记为失败或待核查。 |
| `OnErrRtnOrderInsert(pInputOrder, pRspInfo)` | 报单被柜台/交易所拒绝的错误回报。 | 记录拒单原因；释放策略冻结量。 |
| `OnRtnOrder(pOrder)` | 订单状态变化推送。 | 更新订单状态机，是委托状态的核心数据源。 |
| `OnRtnTrade(pTrade)` | 成交推送。 | 更新成交、持仓、资金、策略信号状态。 |
| `OnRspOrderAction(pInputOrderAction, pRspInfo, nRequestID, bIsLast)` | 撤单请求错误响应。 | 记录撤单错误；根据后续 `OnRtnOrder` 校正状态。 |
| `OnErrRtnOrderAction(pOrderAction, pRspInfo)` | 撤单失败错误回报。 | 标记撤单失败或重新评估是否继续撤。 |
| `OnRtnErrorConditionalOrder(pErrorConditionalOrder)` | 条件单错误推送。 | 记录条件单错误并通知策略。 |

订单回调示例：

```python
# 中文注释：生产代码建议维护本地订单字典，以下仅演示关键字段。
def OnRtnOrder(self, pOrder):
    print("订单状态", pOrder.InstrumentID, pOrder.OrderRef, pOrder.OrderStatus, pOrder.StatusMsg)

# 中文注释：成交推送是更新持仓和成交明细的核心入口。
def OnRtnTrade(self, pTrade):
    print("成交", pTrade.InstrumentID, pTrade.Direction, pTrade.OffsetFlag, pTrade.Price, pTrade.Volume)

# 中文注释：报单错误响应不能代替 OnRtnOrder，但要记录错误原因。
def OnRspOrderInsert(self, pInputOrder, pRspInfo, nRequestID, bIsLast):
    if pRspInfo.ErrorID != 0:
        print("报单错误响应", pInputOrder.OrderRef, pRspInfo.ErrorID, pRspInfo.ErrorMsg)
```

### 13.4 公共推送与通知回调

| 回调 | 触发时机 | 建议处理 |
| --- | --- | --- |
| `OnRtnInstrumentStatus(pInstrumentStatus)` | 合约交易状态变化。 | 更新合约是否可交易，避免非交易状态发单。 |
| `OnRtnBulletin(pBulletin)` | 公告推送。 | 记录公告，必要时告警。 |
| `OnRtnTradingNotice(pTradingNoticeInfo)` | 交易通知推送。 | 记录通知并推送给运维或交易员。 |
| `OnRtnCFMMCTradingAccountToken(pCFMMCTradingAccountToken)` | 保证金监控中心令牌推送。 | 需要 CFMMC 令牌时保存。 |

### 13.5 银期与银行推送回调

银期相关回调包括 `OnRtnFromBankToFutureByBank`、`OnRtnFromFutureToBankByBank`、`OnRtnFromBankToFutureByFuture`、`OnRtnFromFutureToBankByFuture`、`OnRtnQueryBankBalanceByFuture`、`OnErrRtnBankToFutureByFuture`、`OnErrRtnFutureToBankByFuture`、`OnRspFromBankToFutureByFuture`、`OnRspFromFutureToBankByFuture`、`OnRspQueryBankAccountMoneyByFuture` 等。处理原则是：

1. 所有出入金请求都要先落库或写日志。
2. `OnRsp...` 只代表请求响应，最终状态还要结合 `OnRtn...` 或 `OnErrRtn...`。
3. 出金失败、重复流水、银行超时等情况必须人工可追踪。

## 14. 实战建议与常见坑

1. **不要只看请求函数返回值**：`ret == 0` 只表示本地 API 接受请求，业务结果必须看回调。
2. **查询要限速**：启动时同时查资金、持仓、合约、手续费、保证金，建议排队串行发送。
3. **先确认结算单再交易**：很多柜台未确认结算单会拒绝报单。
4. **保存登录会话信息**：撤单常用 `FrontID + SessionID + OrderRef`，登录成功后必须保存。
5. **报单引用要单调递增且避免重复**：重启后可从 `MaxOrderRef` 继续递增。
6. **订单状态以回报为准**：本地策略发单后不要假定已成交，要等待 `OnRtnOrder` 和 `OnRtnTrade`。
7. **区分平今和平昨**：上期所、能源中心等品种常涉及平今/平昨差异，`CombOffsetFlag` 必须按交易所规则填写。
8. **异常断线会自动重连**：断线期间应暂停发单，重连登录后重新查询订单、成交、持仓做状态恢复。
9. **真实账号信息不要写入仓库**：配置文件、日志脱敏，避免泄露密码、认证码和资金信息。
10. **生产交易前做仿真验证**：至少覆盖登录、结算单确认、查资金、查持仓、报单、撤单、拒单、断线重连、成交恢复等流程。

## 15. 常用结构体速查

| 场景 | 结构体 |
| --- | --- |
| 认证 | `ApiStructure.ReqAuthenticateField` |
| 登录 | `ApiStructure.ReqUserLoginField` |
| 登出 | `ApiStructure.UserLogoutField` |
| 结算单确认 | `ApiStructure.SettlementInfoConfirmField` |
| 查询资金 | `ApiStructure.QryTradingAccountField` |
| 查询持仓 | `ApiStructure.QryInvestorPositionField` |
| 查询委托 | `ApiStructure.QryOrderField` |
| 查询成交 | `ApiStructure.QryTradeField` |
| 查询合约 | `ApiStructure.QryInstrumentField` |
| 普通报单 | `ApiStructure.InputOrderField` |
| 普通撤单 | `ApiStructure.InputOrderActionField` |
| 预埋单 | `ApiStructure.ParkedOrderField` |
| 预埋撤单 | `ApiStructure.ParkedOrderActionField` |
| 银期转账 | `ApiStructure.ReqTransferField` |

