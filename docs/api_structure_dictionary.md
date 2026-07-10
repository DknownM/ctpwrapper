# ApiStructure 函数字典 / 结构体备查手册

本文档专门说明 `ctpwrapper.ApiStructure` 中 CTP 请求、响应、回报结构体的逻辑、用法、常见使用时机与示例。它适合在编写 `TraderApiPy` / `MdApiPy` 请求参数、处理回调响应、把结构体落库或从配置恢复对象时备查。

> 说明：CTP 官方把请求参数、查询结果、报单回报、成交回报等都定义为 C/C++ 结构体。本项目把这些结构体映射为 Python `ctypes.Structure` 子类，并统一继承 `ctpwrapper.base.Base`。

## 1. 总体逻辑

### 1.1 结构体在项目中的角色

`ApiStructure` 不是主动发起网络请求的 API，而是一组“数据容器”：

- **请求入参**：例如 `ReqAuthenticateField`、`ReqUserLoginField`、`InputOrderField`、`QryTradingAccountField`，通常传给 `TraderApiPy.Req*()` 或 `MdApiPy.Req*()` 方法。
- **响应结果**：例如 `RspUserLoginField`、`TradingAccountField`、`InvestorPositionField`，通常出现在 `OnRsp*()` 回调中。
- **主动回报/推送**：例如 `OrderField`、`TradeField`、`DepthMarketDataField`，通常出现在 `OnRtn*()` 回调中。
- **错误信息**：例如 `RspInfoField`，几乎所有请求响应回调都会带有，用于判断 `ErrorID` 和 `ErrorMsg`。

### 1.2 一般什么时候使用

| 场景 | 典型结构体 | 使用方式 |
| --- | --- | --- |
| 行情登录 | `ReqUserLoginField` | 构造后传给 `MdApiPy.ReqUserLogin()` |
| 交易认证 | `ReqAuthenticateField` | 连接交易前置后传给 `TraderApiPy.ReqAuthenticate()` |
| 交易登录 | `ReqUserLoginField` | 认证成功后传给 `TraderApiPy.ReqUserLogin()` |
| 结算单确认 | `SettlementInfoConfirmField` | 登录成功后、交易前调用 `ReqSettlementInfoConfirm()` |
| 查询资金 | `QryTradingAccountField` | 登录后调用 `ReqQryTradingAccount()` |
| 查询持仓 | `QryInvestorPositionField` | 登录后调用 `ReqQryInvestorPosition()` |
| 查询合约 | `QryInstrumentField` | 登录后调用 `ReqQryInstrument()` |
| 报单 | `InputOrderField` | 构造报单参数后调用 `ReqOrderInsert()` |
| 撤单 | `InputOrderActionField` | 根据报单引用/系统编号等调用 `ReqOrderAction()` |
| 处理错误 | `RspInfoField` | 在回调中检查 `ErrorID == 0` 是否成功 |
| 处理推送 | `OrderField`、`TradeField`、`DepthMarketDataField` | 在 `OnRtnOrder()`、`OnRtnTrade()`、`OnRtnDepthMarketData()` 中读取字段 |

## 2. 创建结构体

### 2.1 构造函数创建

适合代码中直接构造请求对象。每个字段都可以用关键字参数传入；没有传入的字段会使用类构造函数中的默认值：字符串通常为空字符串，整数通常为 `0`，浮点数通常为 `0.0`。

```python
from ctpwrapper import ApiStructure

field = ApiStructure.ExchangeRateField(
    BrokerID="9999",
    FromCurrencyID="CNY",
    FromCurrencyUnit=1.0,
    ToCurrencyID="USD",
    ExchangeRate=7.2,
)

print(field.BrokerID)
print(field.to_dict())
```

### 2.2 字典创建：`from_dict()`

适合从配置、数据库、缓存、消息队列中恢复请求对象。

```python
from ctpwrapper import ApiStructure

data = {
    "BrokerID": "9999",
    "ToCurrencyID": "USD",
    "ExchangeRate": 7.2,
}

field = ApiStructure.ExchangeRateField.from_dict(data)
print(field.to_dict())
```

注意：`from_dict()` 本质是 `cls(**obj)`，因此字典键必须是该结构体构造函数支持的字段名；如果传入不存在的字段名，Python 会抛出 `TypeError`。

### 2.3 转为字典：`to_dict()`

`to_dict()` 会遍历结构体的 `_fields_`，把字段名和值转换成普通 Python 字典，常用于日志、调试、JSON 序列化前处理、落库前清洗。

```python
account = ApiStructure.QryTradingAccountField(
    BrokerID="9999",
    InvestorID="你的账号",
    BizType="1",
)

payload = account.to_dict()
print(payload["BrokerID"])
```

### 2.4 调试输出：`repr()`

直接 `print(field)` 会调用 `__repr__()`，输出结构体名称和字段值，适合在回调中快速定位问题。

```python
print(ApiStructure.UserLogoutField(BrokerID="9999", UserID="demo"))
```

## 3. 编码与解码逻辑

所有结构体继承 `ctpwrapper.base.Base`，核心逻辑如下：

1. **写入字符串字段时**：各结构体构造函数通常调用 `_to_bytes()`，把传入值转换为 `bytes` 后写入 `ctypes.c_char * N` 字段。
2. **读取字符串字段时**：`Base.__getattribute__()` 如果发现字段值是 `bytes`，会优先按 `gbk` 解码。
3. **解码失败时**：如果遇到 `UnicodeDecodeError`，会返回原始 `bytes`，避免策略进程因为异常字节直接崩溃。
4. **数值字段**：构造函数通常会显式转换为 `int()` 或 `float()`，便于接受字符串形式的配置值。
5. **字段长度**：底层是 CTP C 结构体定长字符数组，超长字符串可能被 `ctypes` 或 C 接口截断/拒绝；生产环境应按官方字段长度控制输入。

## 4. 请求结构体常见范式

### 4.1 登录/认证类

常用结构体：

- `ReqAuthenticateField`：交易客户端认证，通常在 `OnFrontConnected()` 后调用。
- `ReqUserLoginField`：行情或交易登录请求，行情端可直接登录，交易端一般先认证再登录。
- `UserLogoutField`：登出请求。
- `ReqUserLoginWithCaptchaField` / `ReqUserLoginWithTextField` / `ReqUserLoginWithOTPField`：验证码、文本验证、动态口令等扩展登录场景。

```python
from ctpwrapper import ApiStructure

req = ApiStructure.ReqAuthenticateField(
    BrokerID="9999",
    UserID="你的账号",
    AppID="你的AppID",
    AuthCode="你的AuthCode",
)
trader.ReqAuthenticate(req, trader.next_request_id())
```

### 4.2 查询类

命名通常以 `Qry` 开头，例如 `QryTradingAccountField`、`QryInvestorPositionField`、`QryInstrumentField`。查询结构体通常只需要填筛选条件；空字符串常表示“不限制该条件”，但具体是否支持要以柜台和 CTP 官方文档为准。

```python
qry = ApiStructure.QryInvestorPositionField(
    BrokerID="9999",
    InvestorID="你的账号",
    InstrumentID="rb2410",
)
trader.ReqQryInvestorPosition(qry, trader.next_request_id())
```

### 4.3 报单/撤单类

- `InputOrderField`：报单录入请求，是最常用的交易请求结构体。
- `InputOrderActionField`：撤单请求，通常需要 `BrokerID`、`InvestorID`、`OrderRef`、`FrontID`、`SessionID` 或交易所系统编号等定位原报单。
- `OrderField`：报单状态回报。
- `TradeField`：成交回报。

```python
order = ApiStructure.InputOrderField(
    BrokerID="9999",
    InvestorID="你的账号",
    InstrumentID="rb2410",
    OrderRef="000000000001",
    UserID="你的账号",
    Direction="0",          # 买
    CombOffsetFlag="0",     # 开仓
    CombHedgeFlag="1",      # 投机
    LimitPrice=3500.0,
    VolumeTotalOriginal=1,
    OrderPriceType="2",     # 限价
    TimeCondition="3",      # 当日有效
    VolumeCondition="1",    # 任意数量
    MinVolume=1,
    ContingentCondition="1", # 立即
    ForceCloseReason="0",   # 非强平
    IsAutoSuspend=0,
)
trader.ReqOrderInsert(order, trader.next_request_id())
```

### 4.4 行情类

- `DepthMarketDataField`：深度行情推送，在 `OnRtnDepthMarketData()` 中使用。
- `SpecificInstrumentField`：订阅/退订响应中的合约信息。
- `ForQuoteRspField`：询价响应推送。
- `QryMulticastInstrumentField`：查询组播合约。

```python
class MyMdApi(MdApiPy):
    def OnRtnDepthMarketData(self, pDepthMarketData):
        data = pDepthMarketData.to_dict()
        print(data["InstrumentID"], data["LastPrice"], data["Volume"])
```

## 5. 响应处理建议

### 5.1 优先检查 `RspInfoField`

多数 `OnRsp*()` 回调都有 `pRspInfo`。一般以 `ErrorID == 0` 表示成功；失败时读取 `ErrorMsg`。

```python
def OnRspUserLogin(self, pRspUserLogin, pRspInfo, nRequestID, bIsLast):
    if pRspInfo and pRspInfo.ErrorID != 0:
        print("登录失败", pRspInfo.ErrorID, pRspInfo.ErrorMsg)
        return
    print("登录成功", pRspUserLogin.to_dict())
```

### 5.2 用 `bIsLast` 聚合多条查询结果

查询持仓、成交、委托、合约等可能分多条返回。建议用 `nRequestID` 建立缓存，用 `bIsLast` 判断本次查询是否结束。

```python
def OnRspQryInvestorPosition(self, pInvestorPosition, pRspInfo, nRequestID, bIsLast):
    if pInvestorPosition:
        self.positions.setdefault(nRequestID, []).append(pInvestorPosition.to_dict())
    if bIsLast:
        print("本次持仓查询完成", self.positions.pop(nRequestID, []))
```

## 6. 备查总表

下面的表按 `ApiStructure.py` 中定义顺序列出全部结构体。字段列只展示前 8 个字段，完整字段请查看对应类的 `_fields_` 定义。

| 结构体类 | 中文说明 | 字段数 | 前几个字段 |
| --- | --- | ---: | --- |
| `DisseminationField` | 信息分发 | 2 | `SequenceSeries, SequenceNo` |
| `ReqUserLoginField` | 用户登录请求 | 14 | `TradingDay, BrokerID, UserID, Password, UserProductInfo, InterfaceProductInfo, ProtocolInfo, MacAddress ...` |
| `RspUserLoginField` | 用户登录应答 | 19 | `TradingDay, LoginTime, BrokerID, UserID, SystemName, FrontID, SessionID, MaxOrderRef ...` |
| `UserLogoutField` | 用户登出请求 | 2 | `BrokerID, UserID` |
| `ForceUserLogoutField` | 强制交易员退出 | 2 | `BrokerID, UserID` |
| `ReqAuthenticateField` | 客户端认证请求 | 5 | `BrokerID, UserID, UserProductInfo, AuthCode, AppID` |
| `RspAuthenticateField` | 客户端认证响应 | 5 | `BrokerID, UserID, UserProductInfo, AppID, AppType` |
| `AuthenticationInfoField` | 客户端认证信息 | 9 | `BrokerID, UserID, UserProductInfo, AuthInfo, IsResult, AppID, AppType, reserve1 ...` |
| `RspUserLogin2Field` | 用户登录应答2 | 14 | `TradingDay, LoginTime, BrokerID, UserID, SystemName, FrontID, SessionID, MaxOrderRef ...` |
| `TransferHeaderField` | 银期转帐报文头 | 13 | `Version, TradeCode, TradeDate, TradeTime, TradeSerial, FutureID, BankID, BankBrchID ...` |
| `TransferBankToFutureReqField` | 银行资金转期货请求，TradeCode=202001 | 6 | `FutureAccount, FuturePwdFlag, FutureAccPwd, TradeAmt, CustFee, CurrencyCode` |
| `TransferBankToFutureRspField` | 银行资金转期货请求响应 | 6 | `RetCode, RetInfo, FutureAccount, TradeAmt, CustFee, CurrencyCode` |
| `TransferFutureToBankReqField` | 期货资金转银行请求，TradeCode=202002 | 6 | `FutureAccount, FuturePwdFlag, FutureAccPwd, TradeAmt, CustFee, CurrencyCode` |
| `TransferFutureToBankRspField` | 期货资金转银行请求响应 | 6 | `RetCode, RetInfo, FutureAccount, TradeAmt, CustFee, CurrencyCode` |
| `TransferQryBankReqField` | 查询银行资金请求，TradeCode=204002 | 4 | `FutureAccount, FuturePwdFlag, FutureAccPwd, CurrencyCode` |
| `TransferQryBankRspField` | 查询银行资金请求响应 | 7 | `RetCode, RetInfo, FutureAccount, TradeAmt, UseAmt, FetchAmt, CurrencyCode` |
| `TransferQryDetailReqField` | 查询银行交易明细请求，TradeCode=204999 | 1 | `FutureAccount` |
| `TransferQryDetailRspField` | 查询银行交易明细请求响应 | 14 | `TradeDate, TradeTime, TradeCode, FutureSerial, FutureID, FutureAccount, BankSerial, BankID ...` |
| `RspInfoField` | 响应信息 | 2 | `ErrorID, ErrorMsg` |
| `ExchangeField` | 交易所 | 3 | `ExchangeID, ExchangeName, ExchangeProperty` |
| `ProductField` | 产品 | 21 | `reserve1, ProductName, ExchangeID, ProductClass, VolumeMultiple, PriceTick, MaxMarketOrderVolume, MinMarketOrderVolume ...` |
| `InstrumentField` | 合约 | 35 | `reserve1, ExchangeID, InstrumentName, reserve2, reserve3, ProductClass, DeliveryYear, DeliveryMonth ...` |
| `BrokerField` | 经纪公司 | 4 | `BrokerID, BrokerAbbr, BrokerName, IsActive` |
| `TraderField` | 交易所交易员 | 9 | `ExchangeID, TraderID, ParticipantID, Password, InstallCount, BrokerID, OrderCancelAlg, TradeInstallCount ...` |
| `InvestorField` | 投资者 | 15 | `InvestorID, BrokerID, InvestorGroupID, InvestorName, IdentifiedCardType, IdentifiedCardNo, IsActive, Telephone ...` |
| `TradingCodeField` | 交易编码 | 9 | `InvestorID, BrokerID, ExchangeID, ClientID, IsActive, ClientIDType, BranchID, BizType ...` |
| `PartBrokerField` | 会员编码和经纪公司编码对照表 | 4 | `BrokerID, ExchangeID, ParticipantID, IsActive` |
| `SuperUserField` | 管理用户 | 4 | `UserID, UserName, Password, IsActive` |
| `SuperUserFunctionField` | 管理用户功能权限 | 2 | `UserID, FunctionCode` |
| `InvestorGroupField` | 投资者组 | 3 | `BrokerID, InvestorGroupID, InvestorGroupName` |
| `TradingAccountField` | 资金账户 | 50 | `BrokerID, AccountID, PreMortgage, PreCredit, PreDeposit, PreBalance, PreMargin, InterestBase ...` |
| `InvestorPositionField` | 投资者持仓 | 51 | `reserve1, BrokerID, InvestorID, PosiDirection, HedgeFlag, PositionDate, YdPosition, Position ...` |
| `InstrumentMarginRateField` | 合约保证金率 | 13 | `reserve1, InvestorRange, BrokerID, InvestorID, HedgeFlag, LongMarginRatioByMoney, LongMarginRatioByVolume, ShortMarginRatioByMoney ...` |
| `InstrumentCommissionRateField` | 合约手续费率 | 14 | `reserve1, InvestorRange, BrokerID, InvestorID, OpenRatioByMoney, OpenRatioByVolume, CloseRatioByMoney, CloseRatioByVolume ...` |
| `DepthMarketDataField` | 深度行情 | 48 | `TradingDay, reserve1, ExchangeID, reserve2, LastPrice, PreSettlementPrice, PreClosePrice, PreOpenInterest ...` |
| `InstrumentTradingRightField` | 投资者合约交易权限 | 6 | `reserve1, InvestorRange, BrokerID, InvestorID, TradingRight, InstrumentID` |
| `BrokerUserField` | 经纪公司用户 | 7 | `BrokerID, UserID, UserName, UserType, IsActive, IsUsingOTP, IsAuthForce` |
| `BrokerUserPasswordField` | 经纪公司用户口令 | 7 | `BrokerID, UserID, Password, LastUpdateTime, LastLoginTime, ExpireDate, WeakExpireDate` |
| `BrokerUserFunctionField` | 经纪公司用户功能权限 | 3 | `BrokerID, UserID, BrokerFunctionCode` |
| `TraderOfferField` | 交易所交易员报盘机 | 20 | `ExchangeID, TraderID, ParticipantID, Password, InstallID, OrderLocalID, TraderConnectStatus, ConnectRequestDate ...` |
| `SettlementInfoField` | 投资者结算结果 | 8 | `TradingDay, SettlementID, BrokerID, InvestorID, SequenceNo, Content, AccountID, CurrencyID` |
| `InstrumentMarginRateAdjustField` | 合约保证金率调整 | 11 | `reserve1, InvestorRange, BrokerID, InvestorID, HedgeFlag, LongMarginRatioByMoney, LongMarginRatioByVolume, ShortMarginRatioByMoney ...` |
| `ExchangeMarginRateField` | 交易所保证金率 | 9 | `BrokerID, reserve1, HedgeFlag, LongMarginRatioByMoney, LongMarginRatioByVolume, ShortMarginRatioByMoney, ShortMarginRatioByVolume, ExchangeID ...` |
| `ExchangeMarginRateAdjustField` | 交易所保证金率调整 | 16 | `BrokerID, reserve1, HedgeFlag, LongMarginRatioByMoney, LongMarginRatioByVolume, ShortMarginRatioByMoney, ShortMarginRatioByVolume, ExchLongMarginRatioByMoney ...` |
| `ExchangeRateField` | 汇率 | 5 | `BrokerID, FromCurrencyID, FromCurrencyUnit, ToCurrencyID, ExchangeRate` |
| `SettlementRefField` | 结算引用 | 2 | `TradingDay, SettlementID` |
| `CurrentTimeField` | 当前时间 | 4 | `CurrDate, CurrTime, CurrMillisec, ActionDay` |
| `CommPhaseField` | 通讯阶段 | 3 | `TradingDay, CommPhaseNo, SystemID` |
| `LoginInfoField` | 登录信息 | 24 | `FrontID, SessionID, BrokerID, UserID, LoginDate, LoginTime, reserve1, UserProductInfo ...` |
| `LogoutAllField` | 登录信息 | 3 | `FrontID, SessionID, SystemName` |
| `FrontStatusField` | 前置状态 | 4 | `FrontID, LastReportDate, LastReportTime, IsActive` |
| `UserPasswordUpdateField` | 用户口令变更 | 4 | `BrokerID, UserID, OldPassword, NewPassword` |
| `InputOrderField` | 输入报单 | 34 | `BrokerID, InvestorID, reserve1, OrderRef, UserID, OrderPriceType, Direction, CombOffsetFlag ...` |
| `OrderField` | 报单 | 68 | `BrokerID, InvestorID, reserve1, OrderRef, UserID, OrderPriceType, Direction, CombOffsetFlag ...` |
| `ExchangeOrderField` | 交易所报单 | 47 | `OrderPriceType, Direction, CombOffsetFlag, CombHedgeFlag, LimitPrice, VolumeTotalOriginal, TimeCondition, GTDDate ...` |
| `ExchangeOrderInsertErrorField` | 交易所报单插入失败 | 7 | `ExchangeID, ParticipantID, TraderID, InstallID, OrderLocalID, ErrorID, ErrorMsg` |
| `InputOrderActionField` | 输入报单操作 | 21 | `BrokerID, InvestorID, OrderActionRef, OrderRef, RequestID, FrontID, SessionID, ExchangeID ...` |
| `OrderActionField` | 报单操作 | 33 | `BrokerID, InvestorID, OrderActionRef, OrderRef, RequestID, FrontID, SessionID, ExchangeID ...` |
| `ExchangeOrderActionField` | 交易所报单操作 | 20 | `ExchangeID, OrderSysID, ActionFlag, LimitPrice, VolumeChange, ActionDate, ActionTime, TraderID ...` |
| `ExchangeOrderActionErrorField` | 交易所报单操作失败 | 8 | `ExchangeID, OrderSysID, TraderID, InstallID, OrderLocalID, ActionLocalID, ErrorID, ErrorMsg` |
| `ExchangeTradeField` | 交易所成交 | 23 | `ExchangeID, TradeID, Direction, OrderSysID, ParticipantID, ClientID, TradingRole, reserve1 ...` |
| `TradeField` | 成交 | 33 | `BrokerID, InvestorID, reserve1, OrderRef, UserID, ExchangeID, TradeID, Direction ...` |
| `UserSessionField` | 用户会话 | 13 | `FrontID, SessionID, BrokerID, UserID, LoginDate, LoginTime, reserve1, UserProductInfo ...` |
| `QryMaxOrderVolumeField` | 查询最大报单数量 | 10 | `BrokerID, InvestorID, reserve1, Direction, OffsetFlag, HedgeFlag, MaxVolume, ExchangeID ...` |
| `SettlementInfoConfirmField` | 投资者结算结果确认信息 | 7 | `BrokerID, InvestorID, ConfirmDate, ConfirmTime, SettlementID, AccountID, CurrencyID` |
| `SyncDepositField` | 出入金同步 | 9 | `DepositSeqNo, BrokerID, InvestorID, Deposit, IsForce, CurrencyID, IsFromSopt, TradingPassword ...` |
| `SyncFundMortgageField` | 货币质押同步 | 6 | `MortgageSeqNo, BrokerID, InvestorID, FromCurrencyID, MortgageAmount, ToCurrencyID` |
| `BrokerSyncField` | 经纪公司同步 | 1 | `BrokerID` |
| `SyncingInvestorField` | 正在同步中的投资者 | 15 | `InvestorID, BrokerID, InvestorGroupID, InvestorName, IdentifiedCardType, IdentifiedCardNo, IsActive, Telephone ...` |
| `SyncingTradingCodeField` | 正在同步中的交易代码 | 6 | `InvestorID, BrokerID, ExchangeID, ClientID, IsActive, ClientIDType` |
| `SyncingInvestorGroupField` | 正在同步中的投资者分组 | 3 | `BrokerID, InvestorGroupID, InvestorGroupName` |
| `SyncingTradingAccountField` | 正在同步中的交易账号 | 49 | `BrokerID, AccountID, PreMortgage, PreCredit, PreDeposit, PreBalance, PreMargin, InterestBase ...` |
| `SyncingInvestorPositionField` | 正在同步中的投资者持仓 | 50 | `reserve1, BrokerID, InvestorID, PosiDirection, HedgeFlag, PositionDate, YdPosition, Position ...` |
| `SyncingInstrumentMarginRateField` | 正在同步中的合约保证金率 | 11 | `reserve1, InvestorRange, BrokerID, InvestorID, HedgeFlag, LongMarginRatioByMoney, LongMarginRatioByVolume, ShortMarginRatioByMoney ...` |
| `SyncingInstrumentCommissionRateField` | 正在同步中的合约手续费率 | 11 | `reserve1, InvestorRange, BrokerID, InvestorID, OpenRatioByMoney, OpenRatioByVolume, CloseRatioByMoney, CloseRatioByVolume ...` |
| `SyncingInstrumentTradingRightField` | 正在同步中的合约交易权限 | 6 | `reserve1, InvestorRange, BrokerID, InvestorID, TradingRight, InstrumentID` |
| `QryOrderField` | 查询报单 | 9 | `BrokerID, InvestorID, reserve1, ExchangeID, OrderSysID, InsertTimeStart, InsertTimeEnd, InvestUnitID ...` |
| `QryTradeField` | 查询成交 | 9 | `BrokerID, InvestorID, reserve1, ExchangeID, TradeID, TradeTimeStart, TradeTimeEnd, InvestUnitID ...` |
| `QryInvestorPositionField` | 查询投资者持仓 | 6 | `BrokerID, InvestorID, reserve1, ExchangeID, InvestUnitID, InstrumentID` |
| `QryTradingAccountField` | 查询资金账户 | 5 | `BrokerID, InvestorID, CurrencyID, BizType, AccountID` |
| `QryInvestorField` | 查询投资者 | 2 | `BrokerID, InvestorID` |
| `QryTradingCodeField` | 查询交易编码 | 6 | `BrokerID, InvestorID, ExchangeID, ClientID, ClientIDType, InvestUnitID` |
| `QryInvestorGroupField` | 查询投资者组 | 1 | `BrokerID` |
| `QryInstrumentMarginRateField` | 查询合约保证金率 | 7 | `BrokerID, InvestorID, reserve1, HedgeFlag, ExchangeID, InvestUnitID, InstrumentID` |
| `QryInstrumentCommissionRateField` | 查询手续费率 | 6 | `BrokerID, InvestorID, reserve1, ExchangeID, InvestUnitID, InstrumentID` |
| `QryInstrumentTradingRightField` | 查询合约交易权限 | 4 | `BrokerID, InvestorID, reserve1, InstrumentID` |
| `QryBrokerField` | 查询经纪公司 | 1 | `BrokerID` |
| `QryTraderField` | 查询交易员 | 3 | `ExchangeID, ParticipantID, TraderID` |
| `QrySuperUserFunctionField` | 查询管理用户功能权限 | 1 | `UserID` |
| `QryUserSessionField` | 查询用户会话 | 4 | `FrontID, SessionID, BrokerID, UserID` |
| `QryPartBrokerField` | 查询经纪公司会员代码 | 3 | `ExchangeID, BrokerID, ParticipantID` |
| `QryFrontStatusField` | 查询前置状态 | 1 | `FrontID` |
| `QryExchangeOrderField` | 查询交易所报单 | 6 | `ParticipantID, ClientID, reserve1, ExchangeID, TraderID, ExchangeInstID` |
| `QryOrderActionField` | 查询报单操作 | 3 | `BrokerID, InvestorID, ExchangeID` |
| `QryExchangeOrderActionField` | 查询交易所报单操作 | 4 | `ParticipantID, ClientID, ExchangeID, TraderID` |
| `QrySuperUserField` | 查询管理用户 | 1 | `UserID` |
| `QryExchangeField` | 查询交易所 | 1 | `ExchangeID` |
| `QryProductField` | 查询产品 | 4 | `reserve1, ProductClass, ExchangeID, ProductID` |
| `QryInstrumentField` | 查询合约 | 7 | `reserve1, ExchangeID, reserve2, reserve3, InstrumentID, ExchangeInstID, ProductID` |
| `QryDepthMarketDataField` | 查询行情 | 4 | `reserve1, ExchangeID, InstrumentID, ProductClass` |
| `QryBrokerUserField` | 查询经纪公司用户 | 2 | `BrokerID, UserID` |
| `QryBrokerUserFunctionField` | 查询经纪公司用户权限 | 2 | `BrokerID, UserID` |
| `QryTraderOfferField` | 查询交易员报盘机 | 3 | `ExchangeID, ParticipantID, TraderID` |
| `QrySyncDepositField` | 查询出入金流水 | 2 | `BrokerID, DepositSeqNo` |
| `QrySettlementInfoField` | 查询投资者结算结果 | 5 | `BrokerID, InvestorID, TradingDay, AccountID, CurrencyID` |
| `QryExchangeMarginRateField` | 查询交易所保证金率 | 5 | `BrokerID, reserve1, HedgeFlag, ExchangeID, InstrumentID` |
| `QryExchangeMarginRateAdjustField` | 查询交易所调整保证金率 | 4 | `BrokerID, reserve1, HedgeFlag, InstrumentID` |
| `QryExchangeRateField` | 查询汇率 | 3 | `BrokerID, FromCurrencyID, ToCurrencyID` |
| `QrySyncFundMortgageField` | 查询货币质押流水 | 2 | `BrokerID, MortgageSeqNo` |
| `QryHisOrderField` | 查询报单 | 10 | `BrokerID, InvestorID, reserve1, ExchangeID, OrderSysID, InsertTimeStart, InsertTimeEnd, TradingDay ...` |
| `OptionInstrMiniMarginField` | 当前期权合约最小保证金 | 8 | `reserve1, InvestorRange, BrokerID, InvestorID, MinMargin, ValueMethod, IsRelative, InstrumentID` |
| `OptionInstrMarginAdjustField` | 当前期权合约保证金调整系数 | 14 | `reserve1, InvestorRange, BrokerID, InvestorID, SShortMarginRatioByMoney, SShortMarginRatioByVolume, HShortMarginRatioByMoney, HShortMarginRatioByVolume ...` |
| `OptionInstrCommRateField` | 当前期权合约手续费的详细内容 | 15 | `reserve1, InvestorRange, BrokerID, InvestorID, OpenRatioByMoney, OpenRatioByVolume, CloseRatioByMoney, CloseRatioByVolume ...` |
| `OptionInstrTradeCostField` | 期权交易成本 | 12 | `BrokerID, InvestorID, reserve1, HedgeFlag, FixedMargin, MiniMargin, Royalty, ExchFixedMargin ...` |
| `QryOptionInstrTradeCostField` | 期权交易成本查询 | 9 | `BrokerID, InvestorID, reserve1, HedgeFlag, InputPrice, UnderlyingPrice, ExchangeID, InvestUnitID ...` |
| `QryOptionInstrCommRateField` | 期权手续费率查询 | 6 | `BrokerID, InvestorID, reserve1, ExchangeID, InvestUnitID, InstrumentID` |
| `IndexPriceField` | 股指现货指数 | 4 | `BrokerID, reserve1, ClosePrice, InstrumentID` |
| `InputExecOrderField` | 输入的执行宣告 | 23 | `BrokerID, InvestorID, reserve1, ExecOrderRef, UserID, Volume, RequestID, BusinessUnit ...` |
| `InputExecOrderActionField` | 输入执行宣告操作 | 17 | `BrokerID, InvestorID, ExecOrderActionRef, ExecOrderRef, RequestID, FrontID, SessionID, ExchangeID ...` |
| `ExecOrderField` | 执行宣告 | 47 | `BrokerID, InvestorID, reserve1, ExecOrderRef, UserID, Volume, RequestID, BusinessUnit ...` |
| `ExecOrderActionField` | 执行宣告操作 | 30 | `BrokerID, InvestorID, ExecOrderActionRef, ExecOrderRef, RequestID, FrontID, SessionID, ExchangeID ...` |
| `QryExecOrderField` | 执行宣告查询 | 8 | `BrokerID, InvestorID, reserve1, ExchangeID, ExecOrderSysID, InsertTimeStart, InsertTimeEnd, InstrumentID` |
| `ExchangeExecOrderField` | 交易所执行宣告信息 | 32 | `Volume, RequestID, BusinessUnit, OffsetFlag, HedgeFlag, ActionType, PosiDirection, ReservePositionFlag ...` |
| `QryExchangeExecOrderField` | 交易所执行宣告查询 | 6 | `ParticipantID, ClientID, reserve1, ExchangeID, TraderID, ExchangeInstID` |
| `QryExecOrderActionField` | 执行宣告操作查询 | 3 | `BrokerID, InvestorID, ExchangeID` |
| `ExchangeExecOrderActionField` | 交易所执行宣告操作 | 22 | `ExchangeID, ExecOrderSysID, ActionFlag, ActionDate, ActionTime, TraderID, InstallID, ExecOrderLocalID ...` |
| `QryExchangeExecOrderActionField` | 交易所执行宣告操作查询 | 4 | `ParticipantID, ClientID, ExchangeID, TraderID` |
| `ErrExecOrderField` | 错误执行宣告 | 25 | `BrokerID, InvestorID, reserve1, ExecOrderRef, UserID, Volume, RequestID, BusinessUnit ...` |
| `QryErrExecOrderField` | 查询错误执行宣告 | 2 | `BrokerID, InvestorID` |
| `ErrExecOrderActionField` | 错误执行宣告操作 | 19 | `BrokerID, InvestorID, ExecOrderActionRef, ExecOrderRef, RequestID, FrontID, SessionID, ExchangeID ...` |
| `QryErrExecOrderActionField` | 查询错误执行宣告操作 | 2 | `BrokerID, InvestorID` |
| `OptionInstrTradingRightField` | 投资者期权合约交易权限 | 7 | `reserve1, InvestorRange, BrokerID, InvestorID, Direction, TradingRight, InstrumentID` |
| `QryOptionInstrTradingRightField` | 查询期权合约交易权限 | 5 | `BrokerID, InvestorID, reserve1, Direction, InstrumentID` |
| `InputForQuoteField` | 输入的询价 | 11 | `BrokerID, InvestorID, reserve1, ForQuoteRef, UserID, ExchangeID, InvestUnitID, reserve2 ...` |
| `ForQuoteField` | 询价 | 26 | `BrokerID, InvestorID, reserve1, ForQuoteRef, UserID, ForQuoteLocalID, ExchangeID, ParticipantID ...` |
| `QryForQuoteField` | 询价查询 | 8 | `BrokerID, InvestorID, reserve1, ExchangeID, InsertTimeStart, InsertTimeEnd, InvestUnitID, InstrumentID` |
| `ExchangeForQuoteField` | 交易所询价信息 | 14 | `ForQuoteLocalID, ExchangeID, ParticipantID, ClientID, reserve1, TraderID, InstallID, InsertDate ...` |
| `QryExchangeForQuoteField` | 交易所询价查询 | 6 | `ParticipantID, ClientID, reserve1, ExchangeID, TraderID, ExchangeInstID` |
| `InputQuoteField` | 输入的报价 | 29 | `BrokerID, InvestorID, reserve1, QuoteRef, UserID, AskPrice, BidPrice, AskVolume ...` |
| `InputQuoteActionField` | 输入报价操作 | 20 | `BrokerID, InvestorID, QuoteActionRef, QuoteRef, RequestID, FrontID, SessionID, ExchangeID ...` |
| `QuoteField` | 报价 | 57 | `BrokerID, InvestorID, reserve1, QuoteRef, UserID, AskPrice, BidPrice, AskVolume ...` |
| `QuoteActionField` | 报价操作 | 31 | `BrokerID, InvestorID, QuoteActionRef, QuoteRef, RequestID, FrontID, SessionID, ExchangeID ...` |
| `QryQuoteField` | 报价查询 | 9 | `BrokerID, InvestorID, reserve1, ExchangeID, QuoteSysID, InsertTimeStart, InsertTimeEnd, InvestUnitID ...` |
| `ExchangeQuoteField` | 交易所报价信息 | 37 | `AskPrice, BidPrice, AskVolume, BidVolume, RequestID, BusinessUnit, AskOffsetFlag, BidOffsetFlag ...` |
| `QryExchangeQuoteField` | 交易所报价查询 | 6 | `ParticipantID, ClientID, reserve1, ExchangeID, TraderID, ExchangeInstID` |
| `QryQuoteActionField` | 报价操作查询 | 3 | `BrokerID, InvestorID, ExchangeID` |
| `ExchangeQuoteActionField` | 交易所报价操作 | 17 | `ExchangeID, QuoteSysID, ActionFlag, ActionDate, ActionTime, TraderID, InstallID, QuoteLocalID ...` |
| `QryExchangeQuoteActionField` | 交易所报价操作查询 | 4 | `ParticipantID, ClientID, ExchangeID, TraderID` |
| `OptionInstrDeltaField` | 期权合约delta值 | 6 | `reserve1, InvestorRange, BrokerID, InvestorID, Delta, InstrumentID` |
| `ForQuoteRspField` | 发给做市商的询价请求 | 7 | `TradingDay, reserve1, ForQuoteSysID, ForQuoteTime, ActionDay, ExchangeID, InstrumentID` |
| `StrikeOffsetField` | 当前期权合约执行偏移值的详细内容 | 7 | `reserve1, InvestorRange, BrokerID, InvestorID, Offset, OffsetType, InstrumentID` |
| `QryStrikeOffsetField` | 期权执行偏移值查询 | 4 | `BrokerID, InvestorID, reserve1, InstrumentID` |
| `InputBatchOrderActionField` | 输入批量报单操作 | 12 | `BrokerID, InvestorID, OrderActionRef, RequestID, FrontID, SessionID, ExchangeID, UserID ...` |
| `BatchOrderActionField` | 批量报单操作 | 22 | `BrokerID, InvestorID, OrderActionRef, RequestID, FrontID, SessionID, ExchangeID, ActionDate ...` |
| `ExchangeBatchOrderActionField` | 交易所批量报单操作 | 14 | `ExchangeID, ActionDate, ActionTime, TraderID, InstallID, ActionLocalID, ParticipantID, ClientID ...` |
| `QryBatchOrderActionField` | 查询批量报单操作 | 3 | `BrokerID, InvestorID, ExchangeID` |
| `CombInstrumentGuardField` | 组合合约安全系数 | 5 | `BrokerID, reserve1, GuarantRatio, ExchangeID, InstrumentID` |
| `QryCombInstrumentGuardField` | 组合合约安全系数查询 | 4 | `BrokerID, reserve1, ExchangeID, InstrumentID` |
| `InputCombActionField` | 输入的申请组合 | 17 | `BrokerID, InvestorID, reserve1, CombActionRef, UserID, Direction, Volume, CombDirection ...` |
| `CombActionField` | 申请组合 | 33 | `BrokerID, InvestorID, reserve1, CombActionRef, UserID, Direction, Volume, CombDirection ...` |
| `QryCombActionField` | 申请组合查询 | 6 | `BrokerID, InvestorID, reserve1, ExchangeID, InvestUnitID, InstrumentID` |
| `ExchangeCombActionField` | 交易所申请组合信息 | 22 | `Direction, Volume, CombDirection, HedgeFlag, ActionLocalID, ExchangeID, ParticipantID, ClientID ...` |
| `QryExchangeCombActionField` | 交易所申请组合查询 | 6 | `ParticipantID, ClientID, reserve1, ExchangeID, TraderID, ExchangeInstID` |
| `ProductExchRateField` | 产品报价汇率 | 5 | `reserve1, QuoteCurrencyID, ExchangeRate, ExchangeID, ProductID` |
| `QryProductExchRateField` | 产品报价汇率查询 | 3 | `reserve1, ExchangeID, ProductID` |
| `QryForQuoteParamField` | 查询询价价差参数 | 4 | `BrokerID, reserve1, ExchangeID, InstrumentID` |
| `ForQuoteParamField` | 询价价差参数 | 6 | `BrokerID, reserve1, ExchangeID, LastPrice, PriceInterval, InstrumentID` |
| `MMOptionInstrCommRateField` | 当前做市商期权合约手续费的详细内容 | 13 | `reserve1, InvestorRange, BrokerID, InvestorID, OpenRatioByMoney, OpenRatioByVolume, CloseRatioByMoney, CloseRatioByVolume ...` |
| `QryMMOptionInstrCommRateField` | 做市商期权手续费率查询 | 4 | `BrokerID, InvestorID, reserve1, InstrumentID` |
| `MMInstrumentCommissionRateField` | 做市商合约手续费率 | 11 | `reserve1, InvestorRange, BrokerID, InvestorID, OpenRatioByMoney, OpenRatioByVolume, CloseRatioByMoney, CloseRatioByVolume ...` |
| `QryMMInstrumentCommissionRateField` | 查询做市商合约手续费率 | 4 | `BrokerID, InvestorID, reserve1, InstrumentID` |
| `InstrumentOrderCommRateField` | 当前报单手续费的详细内容 | 12 | `reserve1, InvestorRange, BrokerID, InvestorID, HedgeFlag, OrderCommByVolume, OrderActionCommByVolume, ExchangeID ...` |
| `QryInstrumentOrderCommRateField` | 报单手续费率查询 | 4 | `BrokerID, InvestorID, reserve1, InstrumentID` |
| `TradeParamField` | 交易参数 | 4 | `BrokerID, TradeParamID, TradeParamValue, Memo` |
| `InstrumentMarginRateULField` | 合约保证金率调整 | 10 | `reserve1, InvestorRange, BrokerID, InvestorID, HedgeFlag, LongMarginRatioByMoney, LongMarginRatioByVolume, ShortMarginRatioByMoney ...` |
| `FutureLimitPosiParamField` | 期货持仓限制参数 | 8 | `InvestorRange, BrokerID, InvestorID, reserve1, SpecOpenVolume, ArbiOpenVolume, OpenVolume, ProductID` |
| `LoginForbiddenIPField` | 禁止登录IP | 2 | `reserve1, IPAddress` |
| `IPListField` | IP列表 | 3 | `reserve1, IsWhite, IPAddress` |
| `InputOptionSelfCloseField` | 输入的期权自对冲 | 19 | `BrokerID, InvestorID, reserve1, OptionSelfCloseRef, UserID, Volume, RequestID, BusinessUnit ...` |
| `InputOptionSelfCloseActionField` | 输入期权自对冲操作 | 17 | `BrokerID, InvestorID, OptionSelfCloseActionRef, OptionSelfCloseRef, RequestID, FrontID, SessionID, ExchangeID ...` |
| `OptionSelfCloseField` | 期权自对冲 | 43 | `BrokerID, InvestorID, reserve1, OptionSelfCloseRef, UserID, Volume, RequestID, BusinessUnit ...` |
| `OptionSelfCloseActionField` | 期权自对冲操作 | 29 | `BrokerID, InvestorID, OptionSelfCloseActionRef, OptionSelfCloseRef, RequestID, FrontID, SessionID, ExchangeID ...` |
| `QryOptionSelfCloseField` | 期权自对冲查询 | 8 | `BrokerID, InvestorID, reserve1, ExchangeID, OptionSelfCloseSysID, InsertTimeStart, InsertTimeEnd, InstrumentID` |
| `ExchangeOptionSelfCloseField` | 交易所期权自对冲信息 | 28 | `Volume, RequestID, BusinessUnit, HedgeFlag, OptSelfCloseFlag, OptionSelfCloseLocalID, ExchangeID, ParticipantID ...` |
| `QryOptionSelfCloseActionField` | 期权自对冲操作查询 | 3 | `BrokerID, InvestorID, ExchangeID` |
| `ExchangeOptionSelfCloseActionField` | 交易所期权自对冲操作 | 21 | `ExchangeID, OptionSelfCloseSysID, ActionFlag, ActionDate, ActionTime, TraderID, InstallID, OptionSelfCloseLocalID ...` |
| `SyncDelaySwapField` | 延时换汇同步 | 11 | `DelaySwapSeqNo, BrokerID, InvestorID, FromCurrencyID, FromAmount, FromFrozenSwap, FromRemainSwap, ToCurrencyID ...` |
| `QrySyncDelaySwapField` | 查询延时换汇同步 | 2 | `BrokerID, DelaySwapSeqNo` |
| `InvestUnitField` | 投资单元 | 9 | `BrokerID, InvestorID, InvestUnitID, InvestorUnitName, InvestorGroupID, CommModelID, MarginModelID, AccountID ...` |
| `QryInvestUnitField` | 查询投资单元 | 3 | `BrokerID, InvestorID, InvestUnitID` |
| `SecAgentCheckModeField` | 二级代理商资金校验模式 | 5 | `InvestorID, BrokerID, CurrencyID, BrokerSecAgentID, CheckSelfAccount` |
| `SecAgentTradeInfoField` | 二级代理商信息 | 4 | `BrokerID, BrokerSecAgentID, InvestorID, LongCustomerName` |
| `MarketDataField` | 市场行情 | 25 | `TradingDay, reserve1, ExchangeID, reserve2, LastPrice, PreSettlementPrice, PreClosePrice, PreOpenInterest ...` |
| `MarketDataBaseField` | 行情基础属性 | 5 | `TradingDay, PreSettlementPrice, PreClosePrice, PreOpenInterest, PreDelta` |
| `MarketDataStaticField` | 行情静态属性 | 8 | `OpenPrice, HighestPrice, LowestPrice, ClosePrice, UpperLimitPrice, LowerLimitPrice, SettlementPrice, CurrDelta` |
| `MarketDataLastMatchField` | 行情最新成交属性 | 4 | `LastPrice, Volume, Turnover, OpenInterest` |
| `MarketDataBestPriceField` | 行情最优价属性 | 4 | `BidPrice1, BidVolume1, AskPrice1, AskVolume1` |
| `MarketDataBid23Field` | 行情申买二、三属性 | 4 | `BidPrice2, BidVolume2, BidPrice3, BidVolume3` |
| `MarketDataAsk23Field` | 行情申卖二、三属性 | 4 | `AskPrice2, AskVolume2, AskPrice3, AskVolume3` |
| `MarketDataBid45Field` | 行情申买四、五属性 | 4 | `BidPrice4, BidVolume4, BidPrice5, BidVolume5` |
| `MarketDataAsk45Field` | 行情申卖四、五属性 | 4 | `AskPrice4, AskVolume4, AskPrice5, AskVolume5` |
| `MarketDataUpdateTimeField` | 行情更新时间属性 | 5 | `reserve1, UpdateTime, UpdateMillisec, ActionDay, InstrumentID` |
| `MarketDataBandingPriceField` | 行情上下带价 | 2 | `BandingUpperPrice, BandingLowerPrice` |
| `MarketDataExchangeField` | 行情交易所代码属性 | 1 | `ExchangeID` |
| `SpecificInstrumentField` | 指定的合约 | 2 | `reserve1, InstrumentID` |
| `InstrumentStatusField` | 合约状态 | 10 | `ExchangeID, reserve1, SettlementGroupID, reserve2, InstrumentStatus, TradingSegmentSN, EnterTime, EnterReason ...` |
| `QryInstrumentStatusField` | 查询合约状态 | 3 | `ExchangeID, reserve1, ExchangeInstID` |
| `InvestorAccountField` | 投资者账户 | 4 | `BrokerID, InvestorID, AccountID, CurrencyID` |
| `PositionProfitAlgorithmField` | 浮动盈亏算法 | 5 | `BrokerID, AccountID, Algorithm, Memo, CurrencyID` |
| `DiscountField` | 会员资金折扣 | 4 | `BrokerID, InvestorRange, InvestorID, Discount` |
| `QryTransferBankField` | 查询转帐银行 | 2 | `BankID, BankBrchID` |
| `TransferBankField` | 转帐银行 | 4 | `BankID, BankBrchID, BankName, IsActive` |
| `QryInvestorPositionDetailField` | 查询投资者持仓明细 | 6 | `BrokerID, InvestorID, reserve1, ExchangeID, InvestUnitID, InstrumentID` |
| `InvestorPositionDetailField` | 投资者持仓明细 | 31 | `reserve1, BrokerID, InvestorID, HedgeFlag, Direction, OpenDate, TradeID, Volume ...` |
| `TradingAccountPasswordField` | 资金账户口令域 | 4 | `BrokerID, AccountID, Password, CurrencyID` |
| `MDTraderOfferField` | 交易所行情报盘机 | 20 | `ExchangeID, TraderID, ParticipantID, Password, InstallID, OrderLocalID, TraderConnectStatus, ConnectRequestDate ...` |
| `QryMDTraderOfferField` | 查询行情报盘机 | 3 | `ExchangeID, ParticipantID, TraderID` |
| `QryNoticeField` | 查询客户通知 | 1 | `BrokerID` |
| `NoticeField` | 客户通知 | 3 | `BrokerID, Content, SequenceLabel` |
| `UserRightField` | 用户权限 | 4 | `BrokerID, UserID, UserRightType, IsForbidden` |
| `QrySettlementInfoConfirmField` | 查询结算信息确认域 | 4 | `BrokerID, InvestorID, AccountID, CurrencyID` |
| `LoadSettlementInfoField` | 装载结算信息 | 1 | `BrokerID` |
| `BrokerWithdrawAlgorithmField` | 经纪公司可提资金算法表 | 10 | `BrokerID, WithdrawAlgorithm, UsingRatio, IncludeCloseProfit, AllWithoutTrade, AvailIncludeCloseProfit, IsBrokerUserEvent, CurrencyID ...` |
| `TradingAccountPasswordUpdateV1Field` | 资金账户口令变更域 | 4 | `BrokerID, InvestorID, OldPassword, NewPassword` |
| `TradingAccountPasswordUpdateField` | 资金账户口令变更域 | 5 | `BrokerID, AccountID, OldPassword, NewPassword, CurrencyID` |
| `QryCombinationLegField` | 查询组合合约分腿 | 5 | `reserve1, LegID, reserve2, CombInstrumentID, LegInstrumentID` |
| `QrySyncStatusField` | 查询组合合约分腿 | 1 | `TradingDay` |
| `CombinationLegField` | 组合交易合约的单腿 | 8 | `reserve1, LegID, reserve2, Direction, LegMultiple, ImplyLevel, CombInstrumentID, LegInstrumentID` |
| `SyncStatusField` | 数据同步状态 | 2 | `TradingDay, DataSyncStatus` |
| `QryLinkManField` | 查询联系人 | 2 | `BrokerID, InvestorID` |
| `LinkManField` | 联系人 | 12 | `BrokerID, InvestorID, PersonType, IdentifiedCardType, IdentifiedCardNo, PersonName, Telephone, Address ...` |
| `QryBrokerUserEventField` | 查询经纪公司用户事件 | 3 | `BrokerID, UserID, UserEventType` |
| `BrokerUserEventField` | 查询经纪公司用户事件 | 12 | `BrokerID, UserID, UserEventType, EventSequenceNo, EventDate, EventTime, UserEventInfo, InvestorID ...` |
| `QryContractBankField` | 查询签约银行请求 | 3 | `BrokerID, BankID, BankBrchID` |
| `ContractBankField` | 查询签约银行响应 | 5 | `BrokerID, BankID, BankBrchID, BankName, csrcBankID` |
| `InvestorPositionCombineDetailField` | 投资者组合持仓明细 | 23 | `TradingDay, OpenDate, ExchangeID, SettlementID, BrokerID, InvestorID, ComTradeID, TradeID ...` |
| `ParkedOrderField` | 预埋单 | 37 | `BrokerID, InvestorID, reserve1, OrderRef, UserID, OrderPriceType, Direction, CombOffsetFlag ...` |
| `ParkedOrderActionField` | 输入预埋单操作 | 24 | `BrokerID, InvestorID, OrderActionRef, OrderRef, RequestID, FrontID, SessionID, ExchangeID ...` |
| `QryParkedOrderField` | 查询预埋单 | 6 | `BrokerID, InvestorID, reserve1, ExchangeID, InvestUnitID, InstrumentID` |
| `QryParkedOrderActionField` | 查询预埋撤单 | 6 | `BrokerID, InvestorID, reserve1, ExchangeID, InvestUnitID, InstrumentID` |
| `RemoveParkedOrderField` | 删除预埋单 | 4 | `BrokerID, InvestorID, ParkedOrderID, InvestUnitID` |
| `RemoveParkedOrderActionField` | 删除预埋撤单 | 4 | `BrokerID, InvestorID, ParkedOrderActionID, InvestUnitID` |
| `InvestorWithdrawAlgorithmField` | 经纪公司可提资金算法表 | 6 | `BrokerID, InvestorRange, InvestorID, UsingRatio, CurrencyID, FundMortgageRatio` |
| `QryInvestorPositionCombineDetailField` | 查询组合持仓明细 | 6 | `BrokerID, InvestorID, reserve1, ExchangeID, InvestUnitID, CombInstrumentID` |
| `MarketDataAveragePriceField` | 成交均价 | 1 | `AveragePrice` |
| `VerifyInvestorPasswordField` | 校验投资者密码 | 3 | `BrokerID, InvestorID, Password` |
| `UserIPField` | 用户IP | 7 | `BrokerID, UserID, reserve1, reserve2, MacAddress, IPAddress, IPMask` |
| `TradingNoticeInfoField` | 用户事件通知信息 | 7 | `BrokerID, InvestorID, SendTime, FieldContent, SequenceSeries, SequenceNo, InvestUnitID` |
| `TradingNoticeField` | 用户事件通知 | 9 | `BrokerID, InvestorRange, InvestorID, SequenceSeries, UserID, SendTime, SequenceNo, FieldContent ...` |
| `QryTradingNoticeField` | 查询交易事件通知 | 3 | `BrokerID, InvestorID, InvestUnitID` |
| `QryErrOrderField` | 查询错误报单 | 2 | `BrokerID, InvestorID` |
| `ErrOrderField` | 错误报单 | 36 | `BrokerID, InvestorID, reserve1, OrderRef, UserID, OrderPriceType, Direction, CombOffsetFlag ...` |
| `ErrorConditionalOrderField` | 查询错误报单操作 | 68 | `BrokerID, InvestorID, reserve1, OrderRef, UserID, OrderPriceType, Direction, CombOffsetFlag ...` |
| `QryErrOrderActionField` | 查询错误报单操作 | 2 | `BrokerID, InvestorID` |
| `ErrOrderActionField` | 错误报单操作 | 35 | `BrokerID, InvestorID, OrderActionRef, OrderRef, RequestID, FrontID, SessionID, ExchangeID ...` |
| `QryExchangeSequenceField` | 查询交易所状态 | 1 | `ExchangeID` |
| `ExchangeSequenceField` | 交易所状态 | 3 | `ExchangeID, SequenceNo, MarketStatus` |
| `QryMaxOrderVolumeWithPriceField` | 根据价格查询最大报单数量 | 11 | `BrokerID, InvestorID, reserve1, Direction, OffsetFlag, HedgeFlag, MaxVolume, Price ...` |
| `QryBrokerTradingParamsField` | 查询经纪公司交易参数 | 4 | `BrokerID, InvestorID, CurrencyID, AccountID` |
| `BrokerTradingParamsField` | 经纪公司交易参数 | 8 | `BrokerID, InvestorID, MarginPriceType, Algorithm, AvailIncludeCloseProfit, CurrencyID, OptionRoyaltyPriceType, AccountID` |
| `QryBrokerTradingAlgosField` | 查询经纪公司交易算法 | 4 | `BrokerID, ExchangeID, reserve1, InstrumentID` |
| `BrokerTradingAlgosField` | 经纪公司交易算法 | 7 | `BrokerID, ExchangeID, reserve1, HandlePositionAlgoID, FindMarginRateAlgoID, HandleTradingAccountAlgoID, InstrumentID` |
| `QueryBrokerDepositField` | 查询经纪公司资金 | 2 | `BrokerID, ExchangeID` |
| `BrokerDepositField` | 经纪公司资金 | 13 | `TradingDay, BrokerID, ParticipantID, ExchangeID, PreBalance, CurrMargin, CloseProfit, Balance ...` |
| `QryCFMMCBrokerKeyField` | 查询保证金监管系统经纪公司密钥 | 1 | `BrokerID` |
| `CFMMCBrokerKeyField` | 保证金监管系统经纪公司密钥 | 7 | `BrokerID, ParticipantID, CreateDate, CreateTime, KeyID, CurrentKey, KeyKind` |
| `CFMMCTradingAccountKeyField` | 保证金监管系统经纪公司资金账户密钥 | 5 | `BrokerID, ParticipantID, AccountID, KeyID, CurrentKey` |
| `QryCFMMCTradingAccountKeyField` | 请求查询保证金监管系统经纪公司资金账户密钥 | 2 | `BrokerID, InvestorID` |
| `BrokerUserOTPParamField` | 用户动态令牌参数 | 8 | `BrokerID, UserID, OTPVendorsID, SerialNumber, AuthKey, LastDrift, LastSuccess, OTPType` |
| `ManualSyncBrokerUserOTPField` | 手工同步用户动态令牌 | 5 | `BrokerID, UserID, OTPType, FirstOTP, SecondOTP` |
| `CommRateModelField` | 投资者手续费率模板 | 3 | `BrokerID, CommModelID, CommModelName` |
| `QryCommRateModelField` | 请求查询投资者手续费率模板 | 2 | `BrokerID, CommModelID` |
| `MarginModelField` | 投资者保证金率模板 | 3 | `BrokerID, MarginModelID, MarginModelName` |
| `QryMarginModelField` | 请求查询投资者保证金率模板 | 2 | `BrokerID, MarginModelID` |
| `EWarrantOffsetField` | 仓单折抵信息 | 10 | `TradingDay, BrokerID, InvestorID, ExchangeID, reserve1, Direction, HedgeFlag, Volume ...` |
| `QryEWarrantOffsetField` | 查询仓单折抵信息 | 6 | `BrokerID, InvestorID, ExchangeID, reserve1, InvestUnitID, InstrumentID` |
| `QryInvestorProductGroupMarginField` | 查询投资者品种/跨品种保证金 | 7 | `BrokerID, InvestorID, reserve1, HedgeFlag, ExchangeID, InvestUnitID, ProductGroupID` |
| `InvestorProductGroupMarginField` | 投资者品种/跨品种保证金 | 30 | `reserve1, BrokerID, InvestorID, TradingDay, SettlementID, FrozenMargin, LongFrozenMargin, ShortFrozenMargin ...` |
| `QueryCFMMCTradingAccountTokenField` | 查询监控中心用户令牌 | 3 | `BrokerID, InvestorID, InvestUnitID` |
| `CFMMCTradingAccountTokenField` | 监控中心用户令牌 | 5 | `BrokerID, ParticipantID, AccountID, KeyID, Token` |
| `QryProductGroupField` | 查询产品组 | 3 | `reserve1, ExchangeID, ProductID` |
| `ProductGroupField` | 投资者品种/跨品种保证金产品组 | 5 | `reserve1, ExchangeID, reserve2, ProductID, ProductGroupID` |
| `BulletinField` | 交易所公告 | 12 | `ExchangeID, TradingDay, BulletinID, SequenceNo, NewsType, NewsUrgency, SendTime, Abstract ...` |
| `QryBulletinField` | 查询交易所公告 | 5 | `ExchangeID, BulletinID, SequenceNo, NewsType, NewsUrgency` |
| `MulticastInstrumentField` | MulticastInstrument | 7 | `TopicID, reserve1, InstrumentNo, CodePrice, VolumeMultiple, PriceTick, InstrumentID` |
| `QryMulticastInstrumentField` | QryMulticastInstrument | 3 | `TopicID, reserve1, InstrumentID` |
| `AppIDAuthAssignField` | App客户端权限分配 | 3 | `BrokerID, AppID, DRIdentityID` |
| `ReqOpenAccountField` | 转帐开户请求 | 45 | `TradeCode, BankID, BankBranchID, BrokerID, BrokerBranchID, TradeDate, TradeTime, BankSerial ...` |
| `ReqCancelAccountField` | 转帐销户请求 | 45 | `TradeCode, BankID, BankBranchID, BrokerID, BrokerBranchID, TradeDate, TradeTime, BankSerial ...` |
| `ReqChangeAccountField` | 变更银行账户请求 | 41 | `TradeCode, BankID, BankBranchID, BrokerID, BrokerBranchID, TradeDate, TradeTime, BankSerial ...` |
| `ReqTransferField` | 转账请求 | 44 | `TradeCode, BankID, BankBranchID, BrokerID, BrokerBranchID, TradeDate, TradeTime, BankSerial ...` |
| `RspTransferField` | 银行发起银行资金转期货响应 | 46 | `TradeCode, BankID, BankBranchID, BrokerID, BrokerBranchID, TradeDate, TradeTime, BankSerial ...` |
| `ReqRepealField` | 冲正请求 | 51 | `RepealTimeInterval, RepealedTimes, BankRepealFlag, BrokerRepealFlag, PlateRepealSerial, BankRepealSerial, FutureRepealSerial, TradeCode ...` |
| `RspRepealField` | 冲正响应 | 53 | `RepealTimeInterval, RepealedTimes, BankRepealFlag, BrokerRepealFlag, PlateRepealSerial, BankRepealSerial, FutureRepealSerial, TradeCode ...` |
| `ReqQueryAccountField` | 查询账户信息请求 | 37 | `TradeCode, BankID, BankBranchID, BrokerID, BrokerBranchID, TradeDate, TradeTime, BankSerial ...` |
| `RspQueryAccountField` | 查询账户信息响应 | 39 | `TradeCode, BankID, BankBranchID, BrokerID, BrokerBranchID, TradeDate, TradeTime, BankSerial ...` |
| `FutureSignIOField` | 期商签到签退 | 21 | `TradeCode, BankID, BankBranchID, BrokerID, BrokerBranchID, TradeDate, TradeTime, BankSerial ...` |
| `RspFutureSignInField` | 期商签到响应 | 25 | `TradeCode, BankID, BankBranchID, BrokerID, BrokerBranchID, TradeDate, TradeTime, BankSerial ...` |
| `ReqFutureSignOutField` | 期商签退请求 | 21 | `TradeCode, BankID, BankBranchID, BrokerID, BrokerBranchID, TradeDate, TradeTime, BankSerial ...` |
| `RspFutureSignOutField` | 期商签退响应 | 23 | `TradeCode, BankID, BankBranchID, BrokerID, BrokerBranchID, TradeDate, TradeTime, BankSerial ...` |
| `ReqQueryTradeResultBySerialField` | 查询指定流水号的交易结果请求 | 27 | `TradeCode, BankID, BankBranchID, BrokerID, BrokerBranchID, TradeDate, TradeTime, BankSerial ...` |
| `RspQueryTradeResultBySerialField` | 查询指定流水号的交易结果响应 | 26 | `TradeCode, BankID, BankBranchID, BrokerID, BrokerBranchID, TradeDate, TradeTime, BankSerial ...` |
| `ReqDayEndFileReadyField` | 日终文件就绪请求 | 14 | `TradeCode, BankID, BankBranchID, BrokerID, BrokerBranchID, TradeDate, TradeTime, BankSerial ...` |
| `ReturnResultField` | 返回结果 | 2 | `ReturnCode, DescrInfoForReturnCode` |
| `VerifyFuturePasswordField` | 验证期货资金密码 | 19 | `TradeCode, BankID, BankBranchID, BrokerID, BrokerBranchID, TradeDate, TradeTime, BankSerial ...` |
| `VerifyCustInfoField` | 验证客户信息 | 5 | `CustomerName, IdCardType, IdentifiedCardNo, CustType, LongCustomerName` |
| `VerifyFuturePasswordAndCustInfoField` | 验证期货资金密码和客户信息 | 8 | `CustomerName, IdCardType, IdentifiedCardNo, CustType, AccountID, Password, CurrencyID, LongCustomerName` |
| `DepositResultInformField` | 验证期货资金密码和客户信息 | 7 | `DepositSeqNo, BrokerID, InvestorID, Deposit, RequestID, ReturnCode, DescrInfoForReturnCode` |
| `ReqSyncKeyField` | 交易核心向银期报盘发出密钥同步请求 | 20 | `TradeCode, BankID, BankBranchID, BrokerID, BrokerBranchID, TradeDate, TradeTime, BankSerial ...` |
| `RspSyncKeyField` | 交易核心向银期报盘发出密钥同步响应 | 22 | `TradeCode, BankID, BankBranchID, BrokerID, BrokerBranchID, TradeDate, TradeTime, BankSerial ...` |
| `NotifyQueryAccountField` | 查询账户信息通知 | 41 | `TradeCode, BankID, BankBranchID, BrokerID, BrokerBranchID, TradeDate, TradeTime, BankSerial ...` |
| `TransferSerialField` | 银期转账交易流水表 | 28 | `PlateSerial, TradeDate, TradingDay, TradeTime, TradeCode, SessionID, BankID, BankBranchID ...` |
| `QryTransferSerialField` | 请求查询转帐流水 | 4 | `BrokerID, AccountID, BankID, CurrencyID` |
| `NotifyFutureSignInField` | 期商签到通知 | 25 | `TradeCode, BankID, BankBranchID, BrokerID, BrokerBranchID, TradeDate, TradeTime, BankSerial ...` |
| `NotifyFutureSignOutField` | 期商签退通知 | 23 | `TradeCode, BankID, BankBranchID, BrokerID, BrokerBranchID, TradeDate, TradeTime, BankSerial ...` |
| `NotifySyncKeyField` | 交易核心向银期报盘发出密钥同步处理结果的通知 | 22 | `TradeCode, BankID, BankBranchID, BrokerID, BrokerBranchID, TradeDate, TradeTime, BankSerial ...` |
| `QryAccountregisterField` | 请求查询银期签约关系 | 5 | `BrokerID, AccountID, BankID, BankBranchID, CurrencyID` |
| `AccountregisterField` | 客户开销户信息表 | 18 | `TradeDay, BankID, BankBranchID, BankAccount, BrokerID, BrokerBranchID, AccountID, IdCardType ...` |
| `OpenAccountField` | 银期开户信息 | 47 | `TradeCode, BankID, BankBranchID, BrokerID, BrokerBranchID, TradeDate, TradeTime, BankSerial ...` |
| `CancelAccountField` | 银期销户信息 | 47 | `TradeCode, BankID, BankBranchID, BrokerID, BrokerBranchID, TradeDate, TradeTime, BankSerial ...` |
| `ChangeAccountField` | 银期变更银行账号信息 | 43 | `TradeCode, BankID, BankBranchID, BrokerID, BrokerBranchID, TradeDate, TradeTime, BankSerial ...` |
| `SecAgentACIDMapField` | 二级代理操作员银期权限 | 5 | `BrokerID, UserID, AccountID, CurrencyID, BrokerSecAgentID` |
| `QrySecAgentACIDMapField` | 二级代理操作员银期权限查询 | 4 | `BrokerID, UserID, AccountID, CurrencyID` |
| `UserRightsAssignField` | 灾备中心交易权限 | 3 | `BrokerID, UserID, DRIdentityID` |
| `BrokerUserRightAssignField` | 经济公司是否有在本标示的交易权限 | 3 | `BrokerID, DRIdentityID, Tradeable` |
| `DRTransferField` | 灾备交易转换报文 | 4 | `OrigDRIdentityID, DestDRIdentityID, OrigBrokerID, DestBrokerID` |
| `FensUserInfoField` | Fens用户信息 | 3 | `BrokerID, UserID, LoginMode` |
| `CurrTransferIdentityField` | 当前银期所属交易中心 | 1 | `IdentityID` |
| `LoginForbiddenUserField` | 禁止登录用户 | 4 | `BrokerID, UserID, reserve1, IPAddress` |
| `QryLoginForbiddenUserField` | 查询禁止登录用户 | 2 | `BrokerID, UserID` |
| `TradingAccountReserveField` | 资金账户基本准备金 | 4 | `BrokerID, AccountID, Reserve, CurrencyID` |
| `QryLoginForbiddenIPField` | 查询禁止登录IP | 2 | `reserve1, IPAddress` |
| `QryIPListField` | 查询IP列表 | 2 | `reserve1, IPAddress` |
| `QryUserRightsAssignField` | 查询用户下单权限分配表 | 2 | `BrokerID, UserID` |
| `ReserveOpenAccountConfirmField` | 银期预约开户确认请求 | 41 | `TradeCode, BankID, BankBranchID, BrokerID, BrokerBranchID, TradeDate, TradeTime, BankSerial ...` |
| `ReserveOpenAccountField` | 银期预约开户 | 37 | `TradeCode, BankID, BankBranchID, BrokerID, BrokerBranchID, TradeDate, TradeTime, BankSerial ...` |
| `AccountPropertyField` | 银行账户属性 | 14 | `BrokerID, AccountID, BankID, BankAccount, OpenName, OpenBank, IsActive, AccountSourceType ...` |
| `QryCurrDRIdentityField` | 查询当前交易中心 | 1 | `DRIdentityID` |
| `CurrDRIdentityField` | 当前交易中心 | 1 | `DRIdentityID` |
| `QrySecAgentCheckModeField` | 查询二级代理商资金校验模式 | 2 | `BrokerID, InvestorID` |
| `QrySecAgentTradeInfoField` | 查询二级代理商信息 | 2 | `BrokerID, BrokerSecAgentID` |
| `ReqUserAuthMethodField` | 用户发出获取安全安全登陆方法请求 | 3 | `TradingDay, BrokerID, UserID` |
| `RspUserAuthMethodField` | 用户发出获取安全安全登陆方法回复 | 1 | `UsableAuthMethod` |
| `ReqGenUserCaptchaField` | 用户发出获取安全安全登陆方法请求 | 3 | `TradingDay, BrokerID, UserID` |
| `RspGenUserCaptchaField` | 生成的图片验证码信息 | 4 | `BrokerID, UserID, CaptchaInfoLen, CaptchaInfo` |
| `ReqGenUserTextField` | 用户发出获取安全安全登陆方法请求 | 3 | `TradingDay, BrokerID, UserID` |
| `RspGenUserTextField` | 短信验证码生成的回复 | 1 | `UserTextSeq` |
| `ReqUserLoginWithCaptchaField` | 用户发出带图形验证码的登录请求请求 | 13 | `TradingDay, BrokerID, UserID, Password, UserProductInfo, InterfaceProductInfo, ProtocolInfo, MacAddress ...` |
| `ReqUserLoginWithTextField` | 用户发出带短信验证码的登录请求请求 | 13 | `TradingDay, BrokerID, UserID, Password, UserProductInfo, InterfaceProductInfo, ProtocolInfo, MacAddress ...` |
| `ReqUserLoginWithOTPField` | 用户发出带动态验证码的登录请求请求 | 13 | `TradingDay, BrokerID, UserID, Password, UserProductInfo, InterfaceProductInfo, ProtocolInfo, MacAddress ...` |
| `ReqApiHandshakeField` | api握手请求 | 1 | `CryptoKeyVersion` |
| `RspApiHandshakeField` | front发给api的握手回复 | 3 | `FrontHandshakeDataLen, FrontHandshakeData, IsApiAuthEnabled` |
| `ReqVerifyApiKeyField` | api给front的验证key的请求 | 2 | `ApiHandshakeDataLen, ApiHandshakeData` |
| `DepartmentUserField` | 操作员组织架构关系 | 4 | `BrokerID, UserID, InvestorRange, InvestorID` |
| `QueryFreqField` | 查询频率，每秒查询比数 | 2 | `QueryFreq, FTDPkgFreq` |
| `AuthForbiddenIPField` | 禁止认证IP | 1 | `IPAddress` |
| `QryAuthForbiddenIPField` | 查询禁止认证IP | 1 | `IPAddress` |
| `SyncDelaySwapFrozenField` | 换汇可提冻结 | 6 | `DelaySwapSeqNo, BrokerID, InvestorID, FromCurrencyID, FromRemainSwap, IsManualSwap` |
| `UserSystemInfoField` | 用户系统信息 | 11 | `BrokerID, UserID, ClientSystemInfoLen, ClientSystemInfo, reserve1, ClientIPPort, ClientLoginTime, ClientAppID ...` |
| `AuthUserIDField` | 终端用户绑定信息 | 4 | `BrokerID, AppID, UserID, AuthType` |
| `AuthIPField` | 用户IP绑定信息 | 3 | `BrokerID, AppID, IPAddress` |
| `QryClassifiedInstrumentField` | 查询分类合约 | 6 | `InstrumentID, ExchangeID, ExchangeInstID, ProductID, TradingType, ClassType` |
| `QryCombPromotionParamField` | 查询组合优惠比例 | 2 | `ExchangeID, InstrumentID` |
| `CombPromotionParamField` | 组合优惠比例 | 4 | `ExchangeID, InstrumentID, CombHedgeFlag, Xparameter` |
| `ReqUserLoginSMField` | 国密用户登录请求 | 18 | `TradingDay, BrokerID, UserID, Password, UserProductInfo, InterfaceProductInfo, ProtocolInfo, MacAddress ...` |
| `QryRiskSettleInvstPositionField` | 投资者风险结算持仓查询 | 3 | `BrokerID, InvestorID, InstrumentID` |
| `QryRiskSettleProductStatusField` | 风险结算产品查询 | 1 | `ProductID` |
| `RiskSettleInvstPositionField` | 投资者风险结算持仓 | 49 | `InstrumentID, BrokerID, InvestorID, PosiDirection, HedgeFlag, PositionDate, YdPosition, Position ...` |
| `RiskSettleProductStatusField` | 风险品种 | 3 | `ExchangeID, ProductID, ProductStatus` |
| `SyncDeltaInfoField` | 风险结算追平信息 | 4 | `SyncDeltaSequenceNo, SyncDeltaStatus, SyncDescription, IsOnlyTrdDelta` |
| `SyncDeltaProductStatusField` | 风险结算追平产品信息 | 4 | `SyncDeltaSequenceNo, ExchangeID, ProductID, ProductStatus` |
| `SyncDeltaInvstPosDtlField` | 风险结算追平持仓明细 | 30 | `InstrumentID, BrokerID, InvestorID, HedgeFlag, Direction, OpenDate, TradeID, Volume ...` |
| `SyncDeltaInvstPosCombDtlField` | 风险结算追平组合持仓明细 | 21 | `TradingDay, OpenDate, ExchangeID, SettlementID, BrokerID, InvestorID, ComTradeID, TradeID ...` |
| `SyncDeltaTradingAccountField` | 风险结算追平资金 | 50 | `BrokerID, AccountID, PreMortgage, PreCredit, PreDeposit, PreBalance, PreMargin, InterestBase ...` |
| `SyncDeltaInitInvstMarginField` | 投资者风险结算总保证金 | 15 | `BrokerID, InvestorID, LastRiskTotalInvstMargin, LastRiskTotalExchMargin, ThisSyncInvstMargin, ThisSyncExchMargin, RemainRiskInvstMargin, RemainRiskExchMargin ...` |
| `SyncDeltaDceCombInstrumentField` | 风险结算追平组合优先级 | 11 | `CombInstrumentID, ExchangeID, ExchangeInstID, TradeGroupID, CombHedgeFlag, CombinationType, Direction, ProductID ...` |
| `SyncDeltaInvstMarginRateField` | 风险结算追平投资者期货保证金率 | 12 | `InstrumentID, InvestorRange, BrokerID, InvestorID, HedgeFlag, LongMarginRatioByMoney, LongMarginRatioByVolume, ShortMarginRatioByMoney ...` |
| `SyncDeltaExchMarginRateField` | 风险结算追平交易所期货保证金率 | 9 | `BrokerID, InstrumentID, HedgeFlag, LongMarginRatioByMoney, LongMarginRatioByVolume, ShortMarginRatioByMoney, ShortMarginRatioByVolume, ActionDirection ...` |
| `SyncDeltaOptExchMarginField` | 风险结算追平中金现货期权交易所保证金率 | 12 | `BrokerID, InstrumentID, SShortMarginRatioByMoney, SShortMarginRatioByVolume, HShortMarginRatioByMoney, HShortMarginRatioByVolume, AShortMarginRatioByMoney, AShortMarginRatioByVolume ...` |
| `SyncDeltaOptInvstMarginField` | 风险结算追平中金现货期权投资者保证金率 | 15 | `InstrumentID, InvestorRange, BrokerID, InvestorID, SShortMarginRatioByMoney, SShortMarginRatioByVolume, HShortMarginRatioByMoney, HShortMarginRatioByVolume ...` |
| `SyncDeltaInvstMarginRateULField` | 风险结算追平期权标的调整保证金率 | 11 | `InstrumentID, InvestorRange, BrokerID, InvestorID, HedgeFlag, LongMarginRatioByMoney, LongMarginRatioByVolume, ShortMarginRatioByMoney ...` |
| `SyncDeltaOptInvstCommRateField` | 风险结算追平期权手续费率 | 14 | `InstrumentID, InvestorRange, BrokerID, InvestorID, OpenRatioByMoney, OpenRatioByVolume, CloseRatioByMoney, CloseRatioByVolume ...` |
| `SyncDeltaInvstCommRateField` | 风险结算追平期货手续费率 | 12 | `InstrumentID, InvestorRange, BrokerID, InvestorID, OpenRatioByMoney, OpenRatioByVolume, CloseRatioByMoney, CloseRatioByVolume ...` |
| `SyncDeltaProductExchRateField` | 风险结算追平交叉汇率 | 5 | `ProductID, QuoteCurrencyID, ExchangeRate, ActionDirection, SyncDeltaSequenceNo` |
| `SyncDeltaDepthMarketDataField` | 风险结算追平行情 | 48 | `TradingDay, InstrumentID, ExchangeID, ExchangeInstID, LastPrice, PreSettlementPrice, PreClosePrice, PreOpenInterest ...` |
| `SyncDeltaIndexPriceField` | 风险结算追平现货指数 | 5 | `BrokerID, InstrumentID, ClosePrice, ActionDirection, SyncDeltaSequenceNo` |
| `SyncDeltaEWarrantOffsetField` | 风险结算追平仓单折抵 | 10 | `TradingDay, BrokerID, InvestorID, ExchangeID, InstrumentID, Direction, HedgeFlag, Volume ...` |
| `SPBMFutureParameterField` | SPBM期货合约保证金参数 | 11 | `TradingDay, ExchangeID, InstrumentID, ProdFamilyCode, Cvf, TimeRange, MarginRate, LockRateX ...` |
| `SPBMOptionParameterField` | SPBM期权合约保证金参数 | 9 | `TradingDay, ExchangeID, InstrumentID, ProdFamilyCode, Cvf, DownPrice, Delta, SlimiDelta ...` |
| `SPBMIntraParameterField` | SPBM品种内对锁仓折扣参数 | 5 | `TradingDay, ExchangeID, ProdFamilyCode, IntraRateY, AddOnIntraRateY2` |
| `SPBMInterParameterField` | SPBM跨品种抵扣参数 | 6 | `TradingDay, ExchangeID, SpreadId, InterRateZ, Leg1ProdFamilyCode, Leg2ProdFamilyCode` |
| `SyncSPBMParameterEndField` | 同步SPBM参数结束 | 1 | `TradingDay` |
| `QrySPBMFutureParameterField` | SPBM期货合约保证金参数查询 | 3 | `ExchangeID, InstrumentID, ProdFamilyCode` |
| `QrySPBMOptionParameterField` | SPBM期权合约保证金参数查询 | 3 | `ExchangeID, InstrumentID, ProdFamilyCode` |
| `QrySPBMIntraParameterField` | SPBM品种内对锁仓折扣参数查询 | 2 | `ExchangeID, ProdFamilyCode` |
| `QrySPBMInterParameterField` | SPBM跨品种抵扣参数查询 | 3 | `ExchangeID, Leg1ProdFamilyCode, Leg2ProdFamilyCode` |
| `SPBMPortfDefinitionField` | 组合保证金套餐 | 4 | `ExchangeID, PortfolioDefID, ProdFamilyCode, IsSPBM` |
| `SPBMInvestorPortfDefField` | 投资者套餐选择 | 4 | `ExchangeID, BrokerID, InvestorID, PortfolioDefID` |
| `InvestorPortfMarginRatioField` | 投资者新型组合保证金系数 | 6 | `InvestorRange, BrokerID, InvestorID, ExchangeID, MarginRatio, ProductGroupID` |
| `QrySPBMPortfDefinitionField` | 组合保证金套餐查询 | 3 | `ExchangeID, PortfolioDefID, ProdFamilyCode` |
| `QrySPBMInvestorPortfDefField` | 投资者套餐选择查询 | 3 | `ExchangeID, BrokerID, InvestorID` |
| `QryInvestorPortfMarginRatioField` | 投资者新型组合保证金系数查询 | 4 | `BrokerID, InvestorID, ExchangeID, ProductGroupID` |
| `InvestorProdSPBMDetailField` | 投资者产品SPBM明细 | 21 | `ExchangeID, BrokerID, InvestorID, ProdFamilyCode, IntraInstrMargin, BCollectingMargin, SCollectingMargin, IntraProdMargin ...` |
| `QryInvestorProdSPBMDetailField` | 投资者产品SPBM明细查询 | 4 | `ExchangeID, BrokerID, InvestorID, ProdFamilyCode` |
| `PortfTradeParamSettingField` | 组保交易参数设置 | 6 | `ExchangeID, BrokerID, InvestorID, Portfolio, IsActionVerify, IsCloseVerify` |
| `InvestorTradingRightField` | 投资者交易权限设置 | 3 | `BrokerID, InvestorID, InvstTradingRight` |
| `MortgageParamField` | 质押配比参数 | 4 | `BrokerID, AccountID, MortgageBalance, CheckMortgageRatio` |
| `WithDrawParamField` | 可提控制参数 | 4 | `BrokerID, AccountID, WithDrawParamID, WithDrawParamValue` |
| `ThostUserFunctionField` | Thost终端用户功能权限 | 3 | `BrokerID, UserID, ThostFunctionCode` |
| `QryThostUserFunctionField` | Thost终端用户功能权限查询 | 2 | `BrokerID, UserID` |
| `SPBMAddOnInterParameterField` | SPBM附加跨品种抵扣参数 | 6 | `TradingDay, ExchangeID, SpreadId, AddOnInterRateZ2, Leg1ProdFamilyCode, Leg2ProdFamilyCode` |
| `QrySPBMAddOnInterParameterField` | SPBM附加跨品种抵扣参数查询 | 3 | `ExchangeID, Leg1ProdFamilyCode, Leg2ProdFamilyCode` |
| `QryInvestorCommoditySPMMMarginField` | 投资者商品组SPMM记录查询 | 3 | `BrokerID, InvestorID, CommodityID` |
| `QryInvestorCommodityGroupSPMMMarginField` | 投资者商品群SPMM记录查询 | 3 | `BrokerID, InvestorID, CommodityGroupID` |
| `QrySPMMInstParamField` | SPMM合约参数查询 | 1 | `InstrumentID` |
| `QrySPMMProductParamField` | SPMM产品参数查询 | 1 | `ProductID` |
| `InvestorCommoditySPMMMarginField` | 投资者商品组SPMM记录 | 23 | `ExchangeID, BrokerID, InvestorID, CommodityID, MarginBeforeDiscount, MarginNoDiscount, LongPosRisk, LongOpenFrozenRisk ...` |
| `InvestorCommodityGroupSPMMMarginField` | 投资者商品群SPMM记录 | 21 | `ExchangeID, BrokerID, InvestorID, CommodityGroupID, MarginBeforeDiscount, MarginNoDiscount, LongRisk, ShortRisk ...` |
| `SPMMInstParamField` | SPMM合约参数 | 5 | `ExchangeID, InstrumentID, InstMarginCalID, CommodityID, CommodityGroupID` |
| `SPMMProductParamField` | SPMM产品参数 | 4 | `ExchangeID, ProductID, CommodityID, CommodityGroupID` |
| `QryTraderAssignField` | 席位与交易中心对应关系维护查询 | 1 | `TraderID` |
| `TraderAssignField` | 席位与交易中心对应关系 | 5 | `BrokerID, ExchangeID, TraderID, ParticipantID, DRIdentityID` |
| `InvestorInfoCntSettingField` | 投资者申报费阶梯收取设置 | 7 | `ExchangeID, BrokerID, InvestorID, ProductID, IsCalInfoComm, IsLimitInfoMax, InfoMaxLimit` |
| `RCAMSCombProductInfoField` | RCAMS产品组合信息 | 5 | `TradingDay, ExchangeID, ProductID, CombProductID, ProductGroupID` |
| `RCAMSInstrParameterField` | RCAMS同合约风险对冲参数 | 4 | `TradingDay, ExchangeID, ProductID, HedgeRate` |
| `RCAMSIntraParameterField` | RCAMS品种内风险对冲参数 | 4 | `TradingDay, ExchangeID, CombProductID, HedgeRate` |
| `RCAMSInterParameterField` | RCAMS跨品种风险折抵参数 | 7 | `TradingDay, ExchangeID, ProductGroupID, Priority, CreditRate, CombProduct1, CombProduct2` |
| `RCAMSShortOptAdjustParamField` | RCAMS空头期权风险调整参数 | 5 | `TradingDay, ExchangeID, CombProductID, HedgeFlag, AdjustValue` |
| `RCAMSInvestorCombPositionField` | RCAMS策略组合持仓 | 12 | `ExchangeID, BrokerID, InvestorID, InstrumentID, HedgeFlag, PosiDirection, CombInstrumentID, LegID ...` |
| `InvestorProdRCAMSMarginField` | 投资者品种RCAMS保证金 | 27 | `ExchangeID, BrokerID, InvestorID, CombProductID, HedgeFlag, ProductGroupID, RiskBeforeDiscount, IntraInstrRisk ...` |
| `QryRCAMSCombProductInfoField` | RCAMS产品组合信息查询 | 3 | `ProductID, CombProductID, ProductGroupID` |
| `QryRCAMSInstrParameterField` | RCAMS同合约风险对冲参数查询 | 1 | `ProductID` |
| `QryRCAMSIntraParameterField` | RCAMS品种内风险对冲参数查询 | 1 | `CombProductID` |
| `QryRCAMSInterParameterField` | RCAMS跨品种风险折抵参数查询 | 3 | `ProductGroupID, CombProduct1, CombProduct2` |
| `QryRCAMSShortOptAdjustParamField` | RCAMS空头期权风险调整参数查询 | 1 | `CombProductID` |
| `QryRCAMSInvestorCombPositionField` | RCAMS策略组合持仓查询 | 4 | `BrokerID, InvestorID, InstrumentID, CombInstrumentID` |
| `QryInvestorProdRCAMSMarginField` | 投资者品种RCAMS保证金查询 | 4 | `BrokerID, InvestorID, CombProductID, ProductGroupID` |
| `RULEInstrParameterField` | RULE合约保证金参数 | 12 | `TradingDay, ExchangeID, InstrumentID, InstrumentClass, StdInstrumentID, BSpecRatio, SSpecRatio, BHedgeRatio ...` |
| `RULEIntraParameterField` | RULE品种内对锁仓折扣参数 | 7 | `TradingDay, ExchangeID, ProdFamilyCode, StdInstrumentID, StdInstrMargin, UsualIntraRate, DeliveryIntraRate` |
| `RULEInterParameterField` | RULE跨品种抵扣参数 | 10 | `TradingDay, ExchangeID, SpreadId, InterRate, Leg1ProdFamilyCode, Leg2ProdFamilyCode, Leg1PropFactor, Leg2PropFactor ...` |
| `QryRULEInstrParameterField` | RULE合约保证金参数查询 | 2 | `ExchangeID, InstrumentID` |
| `QryRULEIntraParameterField` | RULE品种内对锁仓折扣参数查询 | 2 | `ExchangeID, ProdFamilyCode` |
| `QryRULEInterParameterField` | RULE跨品种抵扣参数查询 | 4 | `ExchangeID, Leg1ProdFamilyCode, Leg2ProdFamilyCode, CommodityGroupID` |
| `InvestorProdRULEMarginField` | 投资者产品RULE保证金 | 27 | `ExchangeID, BrokerID, InvestorID, ProdFamilyCode, InstrumentClass, CommodityGroupID, BStdPosition, SStdPosition ...` |
| `QryInvestorProdRULEMarginField` | 投资者产品RULE保证金查询 | 5 | `ExchangeID, BrokerID, InvestorID, ProdFamilyCode, CommodityGroupID` |
| `SyncDeltaSPBMPortfDefinitionField` | 风险结算追平SPBM组合保证金套餐 | 6 | `ExchangeID, PortfolioDefID, ProdFamilyCode, IsSPBM, ActionDirection, SyncDeltaSequenceNo` |
| `SyncDeltaSPBMInvstPortfDefField` | 风险结算追平投资者SPBM套餐选择 | 6 | `ExchangeID, BrokerID, InvestorID, PortfolioDefID, ActionDirection, SyncDeltaSequenceNo` |
| `SyncDeltaSPBMFutureParameterField` | 风险结算追平SPBM期货合约保证金参数 | 13 | `TradingDay, ExchangeID, InstrumentID, ProdFamilyCode, Cvf, TimeRange, MarginRate, LockRateX ...` |
| `SyncDeltaSPBMOptionParameterField` | 风险结算追平SPBM期权合约保证金参数 | 11 | `TradingDay, ExchangeID, InstrumentID, ProdFamilyCode, Cvf, DownPrice, Delta, SlimiDelta ...` |
| `SyncDeltaSPBMIntraParameterField` | 风险结算追平SPBM品种内对锁仓折扣参数 | 7 | `TradingDay, ExchangeID, ProdFamilyCode, IntraRateY, AddOnIntraRateY2, ActionDirection, SyncDeltaSequenceNo` |
| `SyncDeltaSPBMInterParameterField` | 风险结算追平SPBM跨品种抵扣参数 | 8 | `TradingDay, ExchangeID, SpreadId, InterRateZ, Leg1ProdFamilyCode, Leg2ProdFamilyCode, ActionDirection, SyncDeltaSequenceNo` |
| `SyncDeltaSPBMAddOnInterParamField` | 风险结算追平SPBM附加跨品种抵扣参数 | 8 | `TradingDay, ExchangeID, SpreadId, AddOnInterRateZ2, Leg1ProdFamilyCode, Leg2ProdFamilyCode, ActionDirection, SyncDeltaSequenceNo` |
| `SyncDeltaSPMMInstParamField` | 风险结算追平SPMM合约参数 | 7 | `ExchangeID, InstrumentID, InstMarginCalID, CommodityID, CommodityGroupID, ActionDirection, SyncDeltaSequenceNo` |
| `SyncDeltaSPMMProductParamField` | 风险结算追平SPMM产品相关参数 | 6 | `ExchangeID, ProductID, CommodityID, CommodityGroupID, ActionDirection, SyncDeltaSequenceNo` |
| `SyncDeltaInvestorSPMMModelField` | 风险结算追平投资者SPMM模板选择 | 6 | `ExchangeID, BrokerID, InvestorID, SPMMModelID, ActionDirection, SyncDeltaSequenceNo` |
| `SyncDeltaSPMMModelParamField` | 风险结算追平SPMM模板参数设置 | 9 | `ExchangeID, SPMMModelID, CommodityGroupID, IntraCommodityRate, InterCommodityRate, OptionDiscountRate, MiniMarginRatio, ActionDirection ...` |
| `SyncDeltaRCAMSCombProdInfoField` | 风险结算追平RCAMS产品组合信息 | 7 | `TradingDay, ExchangeID, ProductID, CombProductID, ProductGroupID, ActionDirection, SyncDeltaSequenceNo` |
| `SyncDeltaRCAMSInstrParameterField` | 风险结算追平RCAMS同合约风险对冲参数 | 6 | `TradingDay, ExchangeID, ProductID, HedgeRate, ActionDirection, SyncDeltaSequenceNo` |
| `SyncDeltaRCAMSIntraParameterField` | 风险结算追平RCAMS品种内风险对冲参数 | 6 | `TradingDay, ExchangeID, CombProductID, HedgeRate, ActionDirection, SyncDeltaSequenceNo` |
| `SyncDeltaRCAMSInterParameterField` | 风险结算追平RCAMS跨品种风险折抵参数 | 9 | `TradingDay, ExchangeID, ProductGroupID, Priority, CreditRate, CombProduct1, CombProduct2, ActionDirection ...` |
| `SyncDeltaRCAMSSOptAdjParamField` | 风险结算追平RCAMS空头期权风险调整参数 | 7 | `TradingDay, ExchangeID, CombProductID, HedgeFlag, AdjustValue, ActionDirection, SyncDeltaSequenceNo` |
| `SyncDeltaRCAMSCombRuleDtlField` | 风险结算追平RCAMS策略组合规则明细 | 14 | `TradingDay, ExchangeID, ProdGroup, RuleId, Priority, HedgeFlag, CombMargin, ExchangeInstID ...` |
| `SyncDeltaRCAMSInvstCombPosField` | 风险结算追平RCAMS策略组合持仓 | 14 | `ExchangeID, BrokerID, InvestorID, InstrumentID, HedgeFlag, PosiDirection, CombInstrumentID, LegID ...` |
| `SyncDeltaRULEInstrParameterField` | 风险结算追平RULE合约保证金参数 | 14 | `TradingDay, ExchangeID, InstrumentID, InstrumentClass, StdInstrumentID, BSpecRatio, SSpecRatio, BHedgeRatio ...` |
| `SyncDeltaRULEIntraParameterField` | 风险结算追平RULE品种内对锁仓折扣参数 | 9 | `TradingDay, ExchangeID, ProdFamilyCode, StdInstrumentID, StdInstrMargin, UsualIntraRate, DeliveryIntraRate, ActionDirection ...` |
| `SyncDeltaRULEInterParameterField` | 风险结算追平RULE跨品种抵扣参数 | 12 | `TradingDay, ExchangeID, SpreadId, InterRate, Leg1ProdFamilyCode, Leg2ProdFamilyCode, Leg1PropFactor, Leg2PropFactor ...` |
| `IpAddrParamField` | 服务地址参数 | 14 | `BrokerID, Address, DRIdentityID, DRIdentityName, AddrSrvMode, AddrVer, AddrNo, AddrName ...` |
| `QryIpAddrParamField` | 服务地址参数查询 | 1 | `BrokerID` |
| `TGIpAddrParamField` | 服务地址参数 | 15 | `BrokerID, UserID, Address, DRIdentityID, DRIdentityName, AddrSrvMode, AddrVer, AddrNo ...` |
| `QryTGIpAddrParamField` | 服务地址参数查询 | 3 | `BrokerID, UserID, AppID` |
| `TGSessionQryStatusField` | TGate会话查询状态 | 2 | `LastQryFreq, QryStatus` |
| `LocalAddrConfigField` | 内网地址配置 | 5 | `BrokerID, PeerAddr, NetMask, DRIdentityID, LocalAddress` |
| `QryLocalAddrConfigField` | 内网地址配置查询 | 1 | `BrokerID` |
| `ReqQueryBankAccountBySecField` | 次席查询银行资金帐户信息请求 | 39 | `TradeCode, BankID, BankBranchID, BrokerID, BrokerBranchID, TradeDate, TradeTime, BankSerial ...` |
| `RspQueryBankAccountBySecField` | 次席查询银行资金帐户信息回报 | 41 | `TradeCode, BankID, BankBranchID, BrokerID, BrokerBranchID, TradeDate, TradeTime, BankSerial ...` |
| `ReqTransferBySecField` | 次中心发起的转帐交易 | 46 | `TradeCode, BankID, BankBranchID, BrokerID, BrokerBranchID, TradeDate, TradeTime, BankSerial ...` |
| `RspTransferBySecField` | 次中心发起的转帐交易回报 | 48 | `TradeCode, BankID, BankBranchID, BrokerID, BrokerBranchID, TradeDate, TradeTime, BankSerial ...` |
| `NotifyQueryFutureAccountBySecField` | 查询银行资金帐户信息通知 要发往次席 | 43 | `TradeCode, BankID, BankBranchID, BrokerID, BrokerBranchID, TradeDate, TradeTime, BankSerial ...` |
| `ExitEmergencyField` | 退出紧急状态参数 | 1 | `BrokerID` |
| `InvestorPortfMarginModelField` | 新组保保证金系数投资者模板对应关系 | 3 | `BrokerID, InvestorID, MarginModelID` |
| `InvestorPortfSettingField` | 投资者新组保设置 | 5 | `ExchangeID, BrokerID, InvestorID, HedgeFlag, UsePortf` |
| `QryInvestorPortfSettingField` | 投资者新组保设置查询 | 3 | `ExchangeID, BrokerID, InvestorID` |
| `UserPasswordUpdateFromSecField` | 来自次席的用户口令变更 | 5 | `BrokerID, UserID, OldPassword, NewPassword, FromSec` |
| `SettlementInfoConfirmFromSecField` | 来自次席的结算结果确认 | 5 | `BrokerID, InvestorID, ConfirmDate, ConfirmTime, FromSec` |
| `TradingAccountPasswordUpdateFromSecField` | 来自次席的资金账户口令变更 | 6 | `BrokerID, AccountID, OldPassword, NewPassword, CurrencyID, FromSec` |
| `RiskForbiddenRightField` | 风控禁止的合约交易权限 | 4 | `BrokerID, InvestorID, InstrumentID, UserID` |
| `InvestorInfoCommRecField` | 投资者申报费阶梯收取记录 | 11 | `ExchangeID, BrokerID, InvestorID, InstrumentID, OrderCount, OrderActionCount, ForQuoteCnt, InfoComm ...` |
| `QryInvestorInfoCommRecField` | 投资者申报费阶梯收取记录查询 | 3 | `InvestorID, InstrumentID, BrokerID` |
| `CombLegField` | 组合腿信息 | 6 | `CombInstrumentID, LegID, LegInstrumentID, Direction, LegMultiple, ImplyLevel` |
| `QryCombLegField` | 组合腿信息查询 | 1 | `LegInstrumentID` |
| `InputOffsetSettingField` | 输入的对冲设置 | 13 | `BrokerID, InvestorID, InstrumentID, UnderlyingInstrID, ProductID, OffsetType, Volume, IsOffset ...` |
| `OffsetSettingField` | 对冲设置 | 34 | `BrokerID, InvestorID, InstrumentID, UnderlyingInstrID, ProductID, OffsetType, Volume, IsOffset ...` |
| `CancelOffsetSettingField` | 撤销对冲设置 | 25 | `BrokerID, InvestorID, InstrumentID, UnderlyingInstrID, ProductID, OffsetType, Volume, IsOffset ...` |
| `QryOffsetSettingField` | 查询对冲设置 | 4 | `BrokerID, InvestorID, ProductID, OffsetType` |
| `AddrAppIDRelationField` | 服务地址和AppID的关系 | 4 | `BrokerID, Address, DRIdentityID, AppID` |
| `QryAddrAppIDRelationField` | 服务地址和AppID的关系查询 | 1 | `BrokerID` |
| `WechatUserSystemInfoField` | 微信小程序等用户系统信息 | 9 | `BrokerID, UserID, WechatCltSysInfoLen, WechatCltSysInfo, ClientIPPort, ClientLoginTime, ClientAppID, ClientPublicIP ...` |
| `InvestorReserveInfoField` | 投资者预留信息 | 3 | `BrokerID, UserID, ReserveInfo` |
| `QryInvestorDepartmentFlatField` | 查询组织架构投资者对应关系 | 1 | `BrokerID` |
| `InvestorDepartmentFlatField` | 组织架构投资者对应关系 | 3 | `BrokerID, InvestorID, DepartmentID` |
| `QryDepartmentUserField` | 查询操作员组织架构关系 | 1 | `BrokerID` |
| `AppAuthenticationCodeField` | App客户端认证码 | 5 | `BrokerID, AppID, AuthCode, PreAuthCode, AppType` |
| `UserDRIBypassField` | 客户中心权限豁免 | 3 | `BrokerID, UserID, DRIdentityID` |
| `ReqGenSMSCodeField` | 申请短信验证码请求 | 3 | `BrokerID, UserID, Mobile` |
| `RspGenSMSCodeField` | 申请短信验证码响应 | 3 | `BrokerID, UserID, GenTime` |
| `SMSVerifyInfoFromSecField` | 短信验证信息通知 | 9 | `BrokerID, BrokerAbbr, UserID, Mobile, SMSCode, CreateDate, CreateTime, IsUsed ...` |
| `SMSVerifyConfigField` | 登录验证设置 | 4 | `UserID, BrokerID, Mobile, UseSMSVerify` |
| `SMSVerifyInfoField` | 短信验证信息通知 | 3 | `CreateTime, Mobile, SMSContent` |
| `InputSpdApplyField` | 套利确认输入基本信息 | 13 | `BrokerID, UserID, InvestorID, ExchangeID, FirstLegInstrumentID, SecondLegInstrumentID, Volume, Direction ...` |
| `InputHedgeCfmField` | 套保确认输入基本信息 | 11 | `BrokerID, UserID, InvestorID, ExchangeID, InstrumentID, Volume, Direction, RequestID ...` |
| `SpdApplyField` | 套利申请回报 | 34 | `BrokerID, InvestorID, FirstLegInstrumentID, SecondLegInstrumentID, UserID, Volume, Direction, RequestID ...` |
| `HedgeCfmField` | 套保申请回报 | 34 | `BrokerID, InvestorID, InstrumentID, UserID, Volume, Direction, RequestID, FrontID ...` |
| `QrySpdApplyField` | 套利套保申请查询 | 6 | `BrokerID, InvestorID, ExchangeID, OrderSysID, FirstLegInstrumentID, SecondLegInstrumentID` |
| `QryHedgeCfmField` | 套利套保申请查询 | 5 | `BrokerID, InvestorID, ExchangeID, OrderSysID, InstrumentID` |
| `InputSpdApplyActionField` | 套利申请撤销 | 11 | `BrokerID, UserID, InvestorID, ExchangeID, OrderSysID, OrderRef, FrontID, SessionID ...` |
| `InputHedgeCfmActionField` | 套保申请撤销 | 11 | `BrokerID, UserID, InvestorID, ExchangeID, OrderSysID, OrderRef, FrontID, SessionID ...` |
| `SpdApplyActionField` | 套利申请撤销回报 | 21 | `BrokerID, InvestorID, ActionDate, ActionTime, TraderID, InstallID, OrderLocalID, ActionLocalID ...` |
| `HedgeCfmActionField` | 套保申请撤销回报 | 21 | `BrokerID, InvestorID, ActionDate, ActionTime, TraderID, InstallID, OrderLocalID, ActionLocalID ...` |
| `FrontInfoField` | 前置信息 | 3 | `FrontAddr, QryFreq, FTDPkgFreq` |

## 7. 使用注意事项

- **枚举值通常是字符串编码**：例如买卖方向、开平标志、价格类型等字段在 CTP 中多为 `char` 枚举，传入时常见形式是 `"0"`、`"1"`、`"2"`。请以 CTP 官方头文件和期货公司说明为准。
- **请求频率要限流**：查询类请求过快可能被柜台限流。建议按柜台要求控制 `ReqQry*()` 调用频率。
- **回调对象不要长期持有底层指针**：如果需要异步处理或跨线程使用，优先调用 `to_dict()` 拷贝成普通字典。
- **生产报单前要校验字段**：包括合约、交易所、方向、开平、价格、数量、资金、风控、报单引用等。
- **空字段不等于合法字段**：默认空字符串只是 Python 构造层面的便利，柜台是否接受取决于对应请求类型和 CTP 规则。
