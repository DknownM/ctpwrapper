# CTPWrapper：CTP 期货 API 的 Python 封装

[![build action](https://github.com/nooperpudd/ctpwrapper/actions/workflows/build.yaml/badge.svg?branch=master)](https://github.com/nooperpudd/ctpwrapper/actions/workflows/build.yaml)
[![Build status](https://ci.appveyor.com/api/projects/status/gvvtcqsjo9nsw0ct/branch/master?svg=true)](https://ci.appveyor.com/project/nooperpudd/ctpwrapper)
[![Codacy Badge](https://api.codacy.com/project/badge/Grade/9ed5d0e55ed84dfeba30a7630ab5c160)](https://www.codacy.com/app/nooperpudd/ctpwrapper?utm_source=github.com&amp;utm_medium=referral&amp;utm_content=nooperpudd/ctpwrapper&amp;utm_campaign=Badge_Grade)
[![pypi](https://img.shields.io/pypi/v/ctpwrapper.svg)](https://pypi.python.org/pypi/ctpwrapper)
[![status](https://img.shields.io/pypi/status/ctpwrapper.svg)](https://pypi.python.org/pypi/ctpwrapper)
[![pyversion](https://img.shields.io/pypi/pyversions/ctpwrapper.svg)](https://pypi.python.org/pypi/ctpwrapper)
[![implementation](https://img.shields.io/pypi/implementation/ctpwrapper.svg)](https://pypi.python.org/pypi/ctpwrapper)
[![Downloads](https://pepy.tech/badge/ctpwrapper)](https://pepy.tech/project/ctpwrapper)

CTPWrapper 是对上期技术 CTP（Comprehensive Transaction Platform）期货交易接口的 Python/Cython 封装。项目将 CTP 官方 C++ 行情接口、交易接口与数据采集接口包装为更符合 Python 使用习惯的类，方便在 Python 中完成行情订阅、交易登录、认证、报单、撤单、查询资金、查询持仓等操作。

> 官方 CTP API 下载入口：[SimNow API 下载](https://www.simnow.com.cn/static/apiDownload.action)
> 当前封装版本：`6.7.13`
> 支持平台：Linux 64 bit、Windows 64 bit
> Python 要求：Python `>=3.10`、x86-64 架构
> 实现方式：Cython + C++ 动态库封装
> 许可证：LGPL-3.0-or-later

---

## 目录

- [项目能做什么](#项目能做什么)
- [安装与构建](#安装与构建)
- [快速开始](#快速开始)
- [行情接口用法](#行情接口用法)
- [交易接口用法](#交易接口用法)
- [结构体用法](#结构体用法)
- [ApiStructure 函数字典](docs/api_structure_dictionary.md)
- [数据采集接口用法](#数据采集接口用法)
- [TraderApiPy 函数字典](doc/trader_api_py_function_dictionary.md)：交易接口函数、回调、报单撤单和查询用法备查。
- [代码结构](#代码结构)
- [代码逻辑与调用链](#代码逻辑与调用链)
- [常见问题](#常见问题)
- [测试与开发](#测试与开发)
- [捐助与联系](#捐助与联系)

---

## 项目能做什么

本项目主要提供以下能力：

1. **行情 API 封装**
   - 连接行情前置机。
   - 登录行情服务。
   - 订阅、退订合约行情。
   - 接收深度行情推送。
   - 订阅、退订询价响应。

2. **交易 API 封装**
   - 连接交易前置机。
   - 客户端认证。
   - 用户登录、登出。
   - 查询投资者信息、资金账户、持仓、成交、委托、合约等。
   - 报单、撤单、预埋单、期权自对冲、组合指令等 CTP 原生请求。
   - 处理私有流、公共流、报单回报、成交回报、错误回报等异步事件。

3. **CTP 结构体封装**
   - 使用 `ctypes.Structure` 表达 CTP 请求/响应结构体。
   - 支持通过构造函数或 `from_dict()` 创建结构体对象。
   - 支持通过 `to_dict()` 转换为 Python 字典。
   - 自动处理 CTP 常见 GBK 字节串解码。

4. **数据采集接口封装**
   - 提供 `GetSystemInfo()`、`GetDataCollectApiVersion()`，用于 CTP 终端信息采集相关场景。

---

## 安装与构建

### 1. 从 PyPI 安装

```bash
# 安装或升级 Cython
pip install --upgrade cython

# 安装或升级 ctpwrapper
pip install --upgrade ctpwrapper
```

### 2. 从源码安装

```bash
# 克隆项目
git clone https://github.com/nooperpudd/ctpwrapper.git
cd ctpwrapper

# 安装构建依赖
pip install --upgrade pip setuptools cython

# 本地编译安装
pip install .
```

### 3. 平台与编译说明

- 项目只支持 64 位 Python。如果 Python 不是 `x86-64`，`setup.py` 会直接抛出错误。
- Linux 下会链接 `ctp/linux/` 目录中的 `libthostmduserapi_se.so`、`libthosttraderapi_se.so`、`libLinuxDataCollect.so`。
- Windows 下会链接 `ctp/win/` 目录中的 `.dll` / `.lib`。
- 安装时会把 CTP 动态库复制进 `ctpwrapper` 包内，并通过扩展模块链接。
- 本项目的 Cython 扩展包括：
  - `ctpwrapper.MdApi`：行情 C++ API 封装。
  - `ctpwrapper.TraderApi`：交易 C++ API 封装。
  - `ctpwrapper.datacollect`：终端信息采集 API 封装。

---

## 快速开始

### 配置文件示例

`samples/config.json` 用于示例程序读取账号和前置地址。你可以按自己的 SimNow 或期货公司环境修改：

```json
{
  "broker_id": "9999",
  "investor_id": "你的账号",
  "password": "你的密码",
  "app_id": "你的 AppID",
  "auth_code": "你的 AuthCode",
  "md_server": "tcp://行情前置地址:端口",
  "trader_server": "tcp://交易前置地址:端口"
}
```

> 注意：真实交易环境的 `BrokerID`、前置地址、`AppID`、`AuthCode` 和认证规则由期货公司提供。不要把真实账号密码提交到代码仓库。

### 运行行情示例

```bash
cd samples
python md_main.py
```

### 运行交易示例

```bash
cd samples
python trader_main.py
```

---

## 行情接口用法

行情接口核心类是 `ctpwrapper.MdApiPy`。通常做法是继承它，并重写需要处理的回调函数。

> 需要逐个函数备查时，请阅读单独的 [MdApiPy 行情接口函数字典](docs/mdapi-function-dictionary.md)。

### 基本调用顺序

1. 创建自定义行情类，继承 `MdApiPy`。
2. 重写 `OnFrontConnected()`，连接成功后发起登录。
3. 重写 `OnRspUserLogin()`，登录成功后设置状态或订阅合约。
4. 重写 `OnRtnDepthMarketData()`，处理行情推送。
5. 调用 `Create()` 创建 API 实例。
6. 调用 `RegisterFront()` 注册行情前置地址。
7. 调用 `Init()` 启动接口线程。
8. 登录成功后调用 `SubscribeMarketData()` 订阅合约。
9. 进程结束前调用 `Release()` 或让 `Join()` 阻塞等待接口线程结束。

### 最小行情示例

```python
# encoding:utf-8
import time
from ctpwrapper import ApiStructure, MdApiPy


class MyMdApi(MdApiPy):
    """自定义行情 API：通过重写回调处理 CTP 的异步事件。"""

    def __init__(self, broker_id, user_id, password):
        # 保存登录所需参数
        self.broker_id = broker_id
        self.user_id = user_id
        self.password = password
        self.request_id = 0
        self.login = False

    def next_request_id(self):
        # CTP 请求编号通常由客户端递增维护
        self.request_id += 1
        return self.request_id

    def OnFrontConnected(self):
        # 前置连接成功后，立即发起行情登录请求
        req = ApiStructure.ReqUserLoginField(
            BrokerID=self.broker_id,
            UserID=self.user_id,
            Password=self.password,
        )
        self.ReqUserLogin(req, self.next_request_id())

    def OnRspUserLogin(self, pRspUserLogin, pRspInfo, nRequestID, bIsLast):
        # 登录响应：ErrorID 为 0 表示成功
        if pRspInfo.ErrorID == 0:
            print("行情登录成功", pRspUserLogin)
            self.login = True
            self.SubscribeMarketData(["rb2410"])
        else:
            print("行情登录失败", pRspInfo)

    def OnRspSubMarketData(self, pSpecificInstrument, pRspInfo, nRequestID, bIsLast):
        # 订阅响应：用于确认订阅请求是否被服务端接受
        print("订阅行情响应", pSpecificInstrument, pRspInfo)

    def OnRtnDepthMarketData(self, pDepthMarketData):
        # 行情推送：每收到一笔行情就会触发
        print("收到行情", pDepthMarketData.InstrumentID, pDepthMarketData.LastPrice)


md = MyMdApi("9999", "你的账号", "你的密码")
md.Create()
md.RegisterFront("tcp://行情前置地址:端口")
md.Init()

# 某些环境中 Init 后立刻订阅可能收不到流，建议等待登录回调或适当 sleep
while not md.login:
    time.sleep(0.2)

# 保持进程运行，等待回调
md.Join()
```

### 常用行情方法

| 方法 | 说明 |
| --- | --- |
| `Create(pszFlowPath="", bIsUsingUdp=False, bIsMulticast=False, bIsProductionMode=True)` | 创建行情 API 实例。 |
| `RegisterFront(pszFrontAddress)` | 注册行情前置地址，例如 `tcp://127.0.0.1:17001`。 |
| `RegisterNameServer(pszNsAddress)` | 注册名字服务器，优先级高于 `RegisterFront()`。 |
| `Init()` | 初始化并启动 API 工作线程。 |
| `Join()` | 阻塞等待 API 线程结束。 |
| `Release()` | 释放 API 资源。 |
| `ReqUserLogin(req, request_id)` | 发起行情登录请求。 |
| `ReqUserLogout(req, request_id)` | 发起行情登出请求。 |
| `SubscribeMarketData([instrument_id])` | 订阅行情。 |
| `UnSubscribeMarketData([instrument_id])` | 退订行情。 |
| `SubscribeForQuoteRsp([instrument_id])` | 订阅询价响应。 |
| `UnSubscribeForQuoteRsp([instrument_id])` | 退订询价响应。 |
| `GetTradingDay()` | 获取当前交易日，登录成功后结果才可靠。 |

### 常用行情回调

| 回调 | 触发时机 |
| --- | --- |
| `OnFrontConnected()` | 与行情前置建立连接。 |
| `OnFrontDisconnected(nReason)` | 与行情前置断开。 |
| `OnHeartBeatWarning(nTimeLapse)` | 心跳超时警告。 |
| `OnRspUserLogin(...)` | 行情登录响应。 |
| `OnRspSubMarketData(...)` | 订阅行情响应。 |
| `OnRspUnSubMarketData(...)` | 退订行情响应。 |
| `OnRtnDepthMarketData(pDepthMarketData)` | 深度行情推送。 |
| `OnRspError(...)` | 错误响应。 |

---

## 交易接口用法

交易接口核心类是 `ctpwrapper.TraderApiPy`。与行情接口一样，交易接口也采用“主动请求 + 异步回调”的模式。

### 基本调用顺序

1. 创建自定义交易类，继承 `TraderApiPy`。
2. 重写 `OnFrontConnected()`，连接成功后发起客户端认证 `ReqAuthenticate()`。
3. 重写 `OnRspAuthenticate()`，认证成功后发起登录 `ReqUserLogin()`。
4. 重写 `OnRspUserLogin()`，登录成功后进行结算单确认、查询或交易。
5. 调用 `Create()` 创建 API 实例。
6. 调用 `RegisterFront()` 注册交易前置地址。
7. 根据需要在 `Init()` 前调用 `SubscribePrivateTopic()`、`SubscribePublicTopic()`。
8. 调用 `Init()` 启动接口。
9. 通过 `ReqQry...`、`ReqOrderInsert()`、`ReqOrderAction()` 等方法发起请求。
10. 在 `OnRsp...`、`OnRtn...` 回调中处理响应和推送。

### 最小交易登录示例

```python
# encoding:utf-8
import time
from ctpwrapper import ApiStructure, TraderApiPy


class MyTraderApi(TraderApiPy):
    """自定义交易 API：负责认证、登录与交易回调处理。"""

    def __init__(self, broker_id, user_id, password, app_id, auth_code):
        # 保存交易登录和认证参数
        self.broker_id = broker_id
        self.user_id = user_id
        self.password = password
        self.app_id = app_id
        self.auth_code = auth_code
        self.request_id = 0
        self.login = False

    def next_request_id(self):
        # 每次请求使用新的请求编号，便于在回调中对应请求
        self.request_id += 1
        return self.request_id

    def OnFrontConnected(self):
        # 连接交易前置成功后，先做客户端认证
        req = ApiStructure.ReqAuthenticateField(
            BrokerID=self.broker_id,
            UserID=self.user_id,
            AppID=self.app_id,
            AuthCode=self.auth_code,
        )
        self.ReqAuthenticate(req, self.next_request_id())

    def OnRspAuthenticate(self, pRspAuthenticateField, pRspInfo, nRequestID, bIsLast):
        # 认证成功后再登录
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
        # 登录成功后才可以查询、报单等
        if pRspInfo.ErrorID == 0:
            print("交易登录成功", pRspUserLogin)
            self.login = True
        else:
            print("交易登录失败", pRspInfo)

    def OnRspQryTradingAccount(self, pTradingAccount, pRspInfo, nRequestID, bIsLast):
        # 查询资金账户响应
        print("资金账户", pTradingAccount, "是否最后一条", bIsLast)


trader = MyTraderApi("9999", "你的账号", "你的密码", "你的AppID", "你的AuthCode")
trader.Create()
trader.RegisterFront("tcp://交易前置地址:端口")

# 私有流和公共流订阅要在 Init 前调用
trader.SubscribePrivateTopic(2, 0)  # 2 表示 THOST_TERT_QUICK，只接收登录后的私有流
trader.SubscribePublicTopic(2)   # 2 表示 THOST_TERT_QUICK，只接收登录后的公共流

trader.Init()

while not trader.login:
    time.sleep(0.2)

# 登录成功后查询资金账户
req = ApiStructure.QryTradingAccountField(
    BrokerID="9999",
    InvestorID="你的账号",
    BizType="1",
)
trader.ReqQryTradingAccount(req, trader.next_request_id())

trader.Join()
```

### 查询示例

```python
# 查询投资者信息
qry_investor = ApiStructure.QryInvestorField(
    BrokerID="9999",
    InvestorID="你的账号",
)
trader.ReqQryInvestor(qry_investor, trader.next_request_id())

# 查询持仓
qry_position = ApiStructure.QryInvestorPositionField(
    BrokerID="9999",
    InvestorID="你的账号",
    InstrumentID="rb2410",  # 可为空，具体支持情况以柜台为准
)
trader.ReqQryInvestorPosition(qry_position, trader.next_request_id())

# 查询合约
qry_instrument = ApiStructure.QryInstrumentField(
    InstrumentID="rb2410",
)
trader.ReqQryInstrument(qry_instrument, trader.next_request_id())
```

### 报单示例

下面是一个简化示例，真实生产交易前请仔细确认交易所、合约、方向、开平、价格、数量、报单引用、资金和风控规则。

```python
# encoding:utf-8
from ctpwrapper import ApiStructure

# 构造限价买开报单请求
order = ApiStructure.InputOrderField(
    BrokerID="9999",
    InvestorID="你的账号",
    InstrumentID="rb2410",
    OrderRef="000000000001",
    UserID="你的账号",
    Direction="0",              # 0: 买，1: 卖
    CombOffsetFlag="0",         # 0: 开仓，1: 平仓，3: 平今等，以 CTP 枚举为准
    CombHedgeFlag="1",          # 1: 投机
    LimitPrice=3500.0,
    VolumeTotalOriginal=1,
    OrderPriceType="2",         # 2: 限价
    TimeCondition="3",          # 3: 当日有效
    VolumeCondition="1",        # 1: 任意数量
    MinVolume=1,
    ContingentCondition="1",    # 1: 立即
    ForceCloseReason="0",       # 0: 非强平
    IsAutoSuspend=0,
)

# 发起报单，响应在 OnRspOrderInsert / OnRtnOrder / OnRtnTrade 等回调中处理
trader.ReqOrderInsert(order, trader.next_request_id())
```

### 常用交易方法

| 方法 | 说明 |
| --- | --- |
| `Create(pszFlowPath="", bIsProductionMode=True)` | 创建交易 API 实例。 |
| `RegisterFront(pszFrontAddress)` | 注册交易前置地址。 |
| `RegisterNameServer(pszNsAddress)` | 注册名字服务器。 |
| `SubscribePrivateTopic(nResumeType, nSeqNo)` | 订阅私有流，需在 `Init()` 前调用。 |
| `SubscribePublicTopic(nResumeType)` | 订阅公共流，需在 `Init()` 前调用。 |
| `Init()` | 初始化并启动接口线程。 |
| `ReqAuthenticate(req, request_id)` | 客户端认证。 |
| `ReqUserLogin(req, request_id)` | 用户登录。 |
| `ReqSettlementInfoConfirm(req, request_id)` | 结算单确认。 |
| `ReqQryTradingAccount(req, request_id)` | 查询资金账户。 |
| `ReqQryInvestorPosition(req, request_id)` | 查询投资者持仓。 |
| `ReqQryOrder(req, request_id)` | 查询委托。 |
| `ReqQryTrade(req, request_id)` | 查询成交。 |
| `ReqQryInstrument(req, request_id)` | 查询合约。 |
| `ReqOrderInsert(req, request_id)` | 报单录入。 |
| `ReqOrderAction(req, request_id)` | 撤单。 |
| `ReqUserLogout(req, request_id)` | 用户登出。 |

### 常用交易回调

| 回调 | 触发时机 |
| --- | --- |
| `OnFrontConnected()` | 与交易前置建立连接。 |
| `OnFrontDisconnected(nReason)` | 与交易前置断开。 |
| `OnRspAuthenticate(...)` | 客户端认证响应。 |
| `OnRspUserLogin(...)` | 用户登录响应。 |
| `OnRspSettlementInfoConfirm(...)` | 结算单确认响应。 |
| `OnRspQryTradingAccount(...)` | 查询资金账户响应。 |
| `OnRspQryInvestorPosition(...)` | 查询持仓响应。 |
| `OnRspQryOrder(...)` | 查询委托响应。 |
| `OnRspQryTrade(...)` | 查询成交响应。 |
| `OnRspOrderInsert(...)` | 报单录入错误响应。 |
| `OnRspOrderAction(...)` | 撤单错误响应。 |
| `OnRtnOrder(pOrder)` | 报单状态推送。 |
| `OnRtnTrade(pTrade)` | 成交推送。 |
| `OnRspError(...)` | 通用错误响应。 |

---

## 结构体用法

CTP 的请求和响应都由大量结构体组成。本项目在 `ctpwrapper.ApiStructure` 中提供了对应的 Python 结构体类。更详细的逻辑说明、使用场景、示例和完整结构体总表，请参考 [ApiStructure 函数字典](docs/api_structure_dictionary.md)。

### 构造函数创建

```python
from ctpwrapper import ApiStructure

# 直接使用关键字参数创建结构体
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

### 字典创建

```python
from ctpwrapper import ApiStructure

# 使用 from_dict 创建结构体，适合从配置或数据库中恢复请求对象
data = {
    "BrokerID": "9999",
    "ToCurrencyID": "USD",
    "ExchangeRate": 7.2,
}

field = ApiStructure.ExchangeRateField.from_dict(data)

# 未传入的字符串字段通常默认为空字符串，数值字段通常默认为 0
print(field.to_dict())
```

### 编码与解码逻辑

`ctpwrapper.base.Base` 是结构体基类：

- 读取结构体字段时，如果字段是 `bytes`，会优先尝试按 `gbk` 解码。
- 如果遇到 `UnicodeDecodeError`，会返回原始 `bytes`，避免程序直接崩溃。
- `to_dict()` 会遍历 `_fields_`，把结构体字段转换为 Python 字典。
- `from_dict()` 会把字典解包为结构体构造参数。
- `__repr__()` 会输出结构体名称和字段值，便于调试。

---

## 数据采集接口用法

项目顶层导出了两个数据采集函数：

```python
from ctpwrapper import GetSystemInfo, GetDataCollectApiVersion

# 获取数据采集 API 版本
print(GetDataCollectApiVersion())

# 获取终端系统信息，具体参数和返回值以 CTP 官方 DataCollect 接口为准
print(GetSystemInfo())
```

交易柜台要求终端信息采集时，一般需要结合期货公司要求和 CTP 官方文档使用。

---

## 代码结构

```text
ctpwrapper/
├── ctpwrapper/                    # Python 包源码
│   ├── __init__.py                 # 顶层导出 MdApiPy、TraderApiPy、数据采集函数与版本号
│   ├── base.py                     # CTP 结构体基类，负责 bytes 解码、字典转换、repr
│   ├── ApiStructure.py             # 大量 CTP 结构体的 Python ctypes 定义
│   ├── Md.py                       # 行情 Python 友好封装类 MdApiPy
│   ├── Trader.py                   # 交易 Python 友好封装类 TraderApiPy
│   ├── MdApi.pyx                   # Cython 行情扩展，连接 C++ 行情 API 和 Python 回调
│   ├── TraderApi.pyx               # Cython 交易扩展，连接 C++ 交易 API 和 Python 回调
│   ├── datacollect.pyx             # Cython 数据采集扩展
│   ├── headers/                    # Cython 使用的 pxd 声明文件
│   └── cppheader/                  # C++ 桥接头文件
├── ctp/                            # CTP 官方头文件和动态库
│   ├── header/                     # 官方 .h 头文件、错误码 XML/DTD
│   ├── linux/                      # Linux 动态库
│   ├── win/                        # Windows 动态库和导入库
│   └── version.txt                 # CTP 版本信息
├── samples/                        # 示例程序
│   ├── config.json                 # 示例配置
│   ├── md_main.py                  # 行情示例
│   └── trader_main.py              # 交易示例
├── tests/                          # 单元测试
├── doc/ctp/                        # CTP 官方文档资料
├── generate.py                     # 从 CTP 官方头文件生成 pxd 类型声明
├── generate_structure.py           # 生成/辅助结构体相关代码
├── setup.py                        # setuptools + Cython 构建脚本
├── pyproject.toml                  # PEP 517 构建配置和项目元数据
├── requirements.txt                # 依赖列表
└── README.md                       # 项目说明文档
```

---

## 代码逻辑与调用链

### 1. 导入入口

用户通常从顶层包导入：

```python
from ctpwrapper import MdApiPy, TraderApiPy, ApiStructure
```

`ctpwrapper/__init__.py` 会导出：

- `MdApiPy`：行情接口 Python 封装。
- `TraderApiPy`：交易接口 Python 封装。
- `GetSystemInfo`、`GetDataCollectApiVersion`：数据采集接口。
- `__version__`：包版本。

### 2. Python 友好封装层

`Md.py` 与 `Trader.py` 主要负责把 Python 类型转换为 Cython/C++ 层需要的类型。例如：

- `RegisterFront("tcp://...")` 会把字符串编码为 `bytes` 后传给底层。
- `SubscribeMarketData(["rb2410"])` 会把字符串列表转换为 `bytes` 列表。
- `GetTradingDay()` 会把底层返回的 `bytes` 解码为 Python `str`。
- `Init()` 调用底层初始化后会 `sleep(1)`，给 C++ API 线程一点启动时间。

### 3. Cython 桥接层

`MdApi.pyx`、`TraderApi.pyx` 和 `datacollect.pyx` 是核心桥接层，负责：

- 持有并管理 C++ API 对象。
- 调用官方 CTP C++ API 方法。
- 接收 C++ SPI 回调。
- 将 C++ 结构体转换为 Python 结构体对象。
- 调用 Python 子类中重写的 `OnRsp...`、`OnRtn...`、`OnFront...` 方法。

换句话说，用户写的 Python 回调函数不是主动被 Python 调度，而是由 CTP C++ API 在线程中收到网络消息后，经 Cython 桥接再回调到 Python。

### 4. 结构体层

`ApiStructure.py` 中的结构体继承自 `Base`。结构体字段与 CTP 官方头文件中的字段保持对应关系。典型请求流程是：

```text
Python dict / Python 参数
        ↓
ApiStructure.XxxField 结构体对象
        ↓
TraderApiPy / MdApiPy 请求方法
        ↓
Cython 扩展模块
        ↓
CTP 官方 C++ API
        ↓
柜台 / 前置机
```

典型响应流程是：

```text
柜台 / 前置机
        ↓
CTP 官方 C++ API 回调
        ↓
Cython SPI 桥接
        ↓
ApiStructure.XxxField 结构体对象
        ↓
Python OnRsp... / OnRtn... 回调
        ↓
用户策略、日志、风控、数据库等业务代码
```

### 5. 异步回调模型

CTP API 是典型的异步接口：

- `Req...` 方法只表示“请求已发送/提交到底层 API”，不代表业务成功。
- 业务结果需要在对应 `OnRsp...` 回调中判断。
- 委托状态和成交信息通常通过 `OnRtnOrder()`、`OnRtnTrade()` 推送。
- 查询类接口可能返回多条记录，需要结合 `bIsLast` 判断是否为最后一条。
- 每个请求应使用递增的 `nRequestID`，便于在回调中定位请求来源。

---

## 常见问题

### 1. `Init()` 后没有收到行情怎么办？

有些环境中，行情流连接和登录后立刻订阅可能没有数据。建议：

- 在 `OnRspUserLogin()` 登录成功后再订阅。
- 或在 `Init()` 后适当 `time.sleep(1~2)`。
- 确认合约代码、交易时段、前置地址、账号权限是否正确。

### 2. 为什么字段有时是字符串，有时是 bytes？

结构体基类会自动尝试用 `gbk` 解码 `bytes`。如果底层返回的字节串不完整或不是合法 GBK，代码会保留原始 `bytes`，避免触发 `UnicodeDecodeError` 导致回调中断。

处理方式示例：

```python
value = pRspInfo.ErrorMsg

# 如果自动解码失败，value 可能仍然是 bytes，需要自行容错处理
if isinstance(value, bytes):
    value = value.decode("gbk", errors="ignore")

print(value)
```

### 3. 查询回调为什么会触发多次？

CTP 查询类请求可能返回多条记录。每条记录都会触发一次回调，最后一条的 `bIsLast` 为 `True`。如果你要汇总查询结果，应在回调中先收集数据，等 `bIsLast` 为 `True` 时再统一处理。

```python
class MyTraderApi(TraderApiPy):
    def __init__(self):
        self.positions = []

    def OnRspQryInvestorPosition(self, pInvestorPosition, pRspInfo, nRequestID, bIsLast):
        # 非空记录才加入结果列表
        if pInvestorPosition:
            self.positions.append(pInvestorPosition.to_dict())

        # 最后一条到达后再统一处理
        if bIsLast:
            print("持仓查询完成", self.positions)
```

### 4. 请求方法返回值代表什么？

大多数 `Req...` 方法返回 `int`：

- `0` 通常表示请求成功提交到底层 API。
- 非 `0` 表示本地提交失败，例如网络、流控、参数或 API 状态问题。
- 即使返回 `0`，业务仍可能失败，需要看对应响应回调中的 `pRspInfo.ErrorID`。

### 5. 生产交易需要注意什么？

- 请不要在不了解 CTP 字段含义的情况下直接实盘报单。
- 报单、撤单要做好本地风控、日志、异常恢复和幂等处理。
- 不同期货公司柜台对认证、终端信息采集、流控、查询频率可能有不同要求。
- 委托状态应以 `OnRtnOrder()`、`OnRtnTrade()` 和柜台查询结果综合判断。
- 网络断线后 CTP API 会自动重连，但你的业务状态、订阅状态、登录状态仍需要自行管理。

---

## 测试与开发

### 运行测试

```bash
pytest
```

### 只运行结构体测试

```bash
pytest tests/test_structure.py
```

### 查看 API 版本测试

```bash
pytest tests/test_api.py -s
```

### 重新生成 Cython 类型声明

当更新 CTP 官方头文件后，可使用：

```bash
python generate.py
```

`generate.py` 会读取 `ctp/header/ThostFtdcUserApiDataType.h` 和 `ctp/header/ThostFtdcUserApiStruct.h`，生成 `ctpwrapper/headers/ThostFtdcUserApiDataType.pxd` 与 `ctpwrapper/headers/ThostFtdcUserApiStruct.pxd`。

---

## 文档

CTP 官方文档资料位于：

```text
doc/ctp/
```

其中包含 CTP 接口说明文件，可配合本项目结构体和方法名查阅字段含义、枚举值和业务流程。

---



## 致谢

本项目 Inspired By lovelylain，并感谢 CTP 社区中持续提供反馈、问题定位和兼容性建议的开发者。
