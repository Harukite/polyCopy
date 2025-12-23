 # Polymarket 跟单系统架构设计文档

**版本:** 1.0  
**日期:** 2025年12月  
**状态:** 详细设计

---

## 目录

1. [文档概述](#1-文档概述)
2. [系统架构](#2-系统架构)
3. [技术栈选型](#3-技术栈选型)
4. [核心模块设计](#4-核心模块设计)
5. [数据流设计](#5-数据流设计)
6. [数据模型设计](#6-数据模型设计)
7. [API集成方案](#7-api集成方案)
8. [风险控制系统](#8-风险控制系统)
9. [性能优化方案](#9-性能优化方案)
10. [安全设计](#10-安全设计)
11. [部署架构](#11-部署架构)
12. [监控与告警](#12-监控与告警)
13. [容灾与备份](#13-容灾与备份)
14. [开发计划](#14-开发计划)

---

## 1. 文档概述

### 1.1 项目背景

Polymarket 是全球最大的去中心化预测市场平台,本系统旨在实现对目标交易者的自动化跟单功能,通过实时监控目标用户的交易行为并自动复制,帮助用户跟随优秀交易者的策略。

### 1.2 设计目标

- **低延迟**: 交易复制延迟 < 3秒
- **高可靠**: 系统可用性 > 99.5%
- **可扩展**: 支持同时跟踪 50+ 目标用户
- **安全性**: 资金安全和私钥管理
- **可配置**: 灵活的风控参数和跟单策略

### 1.3 技术约束

- Polymarket 基于 Polygon (Chain ID: 137) 网络
- CLOB API 为混合去中心化订单簿系统
- 订单需要 EIP-712 签名
- 存在地理位置访问限制

---

## 2. 系统架构

### 2.1 总体架构

根据实际开源实现,系统架构保持简单:

```
┌─────────────────────────────────────────────────────────────────┐
│                       核心服务层                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│  │ Trade Monitor│  │  Order Engine│  │  Position    │        │
│  │   Service    │  │   Service    │  │   Manager    │        │
│  └──────┬───────┘  └──────┬───────┘  └──────────────┘        │
│         │                  │                                    │
│         └──────────┬───────┘                                    │
│                    │                                            │
└────────────────────┼────────────────────────────────────────────┘
                     │
┌────────────────────┴────────────────────────────────────────────┐
│                       数据存储层 (可选)                           │
│  ┌──────────────┐  ┌──────────────┐                          │
│  │   MongoDB    │  │  JSON Files  │                          │
│  │   (可选)      │  │  (默认)       │                          │
│  └──────────────┘  └──────────────┘                          │
└────────────────────┬────────────────────────────────────────────┘
                     │
┌────────────────────┴────────────────────────────────────────────┐
│                       外部接口层                                  │
│  ┌──────────────┐  ┌──────────────┐                          │
│  │   Polymarket │  │    Polygon   │                          │
│  │   CLOB API   │  │   RPC Node   │                          │
│  └──────────────┘  └──────────────┘                          │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 数据流架构

```
目标用户交易 → Polymarket CLOB
                    ↓
        [WebSocket 实时监听] ← REST API 轮询备用
                    ↓
        Trade Monitor (解析&过滤&去重)
                    ↓
          Order Engine (构建&提交订单)
                    ↓
        本地 JSON / MongoDB (记录)
```

---

## 3. 技术栈选型

### 3.1 后端技术栈

| 组件 | 技术选型 | 版本 | 理由 |
|------|---------|------|------|
| 运行时 | Node.js | 20.x LTS | 与官方 SDK 兼容,异步 I/O 性能优秀 |
| 编程语言 | TypeScript | 5.x | 类型安全,开发效率高 |
| Web框架 | NestJS | 10.x | 模块化设计,企业级架构支持 |
| CLOB客户端 | @polymarket/clob-client | Latest | 官方 SDK,完整支持 |
| WebSocket | ws + RealTimeDataClient | Latest | 原生性能 + 官方实时数据客户端 |
| 区块链交互 | ethers.js | 5.8.0 | 成熟稳定,与 CLOB 客户端兼容 |

### 3.2 数据存储

| 组件 | 技术选型 | 版本 | 用途 |
|------|---------|------|------|
| 主数据库 | MongoDB | 7.x | 交易记录、持仓数据 (可选) |
| 缓存 | 内存对象 | - | 实时数据缓存、去重 |

**注意**: 根据实际开源项目,大部分实现只使用本地 JSON 文件存储数据,MongoDB 是可选的。

### 3.3 基础设施

| 组件 | 技术选型 | 版本 | 用途 |
|------|---------|------|------|
| 进程管理 | PM2 | Latest | 生产环境进程守护 |
| 日志 | console.log | - | 简单日志输出 |

---

## 4. 核心模块设计

### 4.1 Trade Monitor Service (交易监控服务)

**职责**: 实时监控目标用户的交易活动

**技术方案**:

#### 方案A: WebSocket 实时监控 (推荐)

```typescript
// 基于官方 RealTimeDataClient
import { RealTimeDataClient } from '@polymarket/real-time-data-client';

class TradeMonitorService {
  private rtdsClient: RealTimeDataClient;
  private clobWSClient: WebSocket;
  
  async initialize() {
    // 1. RTDS 订阅公开交易活动
    this.rtdsClient = new RealTimeDataClient({
      onMessage: this.handleRTDSMessage.bind(this),
      onConnect: this.subscribeToActivity.bind(this)
    });
    
    // 2. CLOB WebSocket 订阅用户私有数据
    this.clobWSClient = new WebSocket(
      'wss://ws-subscriptions-clob.polymarket.com/ws/user'
    );
  }
  
  private subscribeToActivity(client: RealTimeDataClient) {
    client.subscribe({
      subscriptions: [
        {
          topic: 'activity',
          type: 'trades',
          // 过滤特定用户
          filters: JSON.stringify({
            makerAddress: targetUserAddress
          })
        }
      ]
    });
  }
  
  private async handleRTDSMessage(message: Message) {
    if (message.type === 'trades') {
      await this.processTrade(message.payload);
    }
  }
}
```

#### 方案B: REST API 轮询备用

```typescript
class PollingMonitor {
  private intervalId: NodeJS.Timeout;
  private lastTradeTimestamp: number;
  
  start(intervalMs: number = 1000) {
    this.intervalId = setInterval(
      () => this.pollTrades(),
      intervalMs
    );
  }
  
  private async pollTrades() {
    const trades = await this.clobClient.getTrades({
      maker: TARGET_USER_ADDRESS,
      start_time: this.lastTradeTimestamp
    });
    
    for (const trade of trades) {
      if (trade.timestamp > this.lastTradeTimestamp) {
        await this.processTrade(trade);
        this.lastTradeTimestamp = trade.timestamp;
      }
    }
  }
}
```

**核心功能**:

1. **多源监控**: WebSocket (主) + REST 轮询 (备)
2. **事件过滤**: 只处理目标用户的交易
3. **去重机制**: 基于 trade_id 去重
4. **断线重连**: 指数退避重连策略
5. **监控指标**: 延迟、消息量、错误率

### 4.2 Order Engine Service (订单引擎)

**职责**: 构建和提交复制订单

```typescript
interface CopyOrderConfig {
  sizeMultiplier: number;      // 跟单倍数
  maxOrderSize: number;        // 最大单笔金额
  minOrderSize: number;        // 最小单笔金额
  allowedMarkets: string[];    // 允许的市场
  blacklistedTokens: string[]; // 黑名单 token
}

class OrderEngine {
  private clobClient: ClobClient;
  
  async copyTrade(
    originalTrade: Trade,
    config: CopyOrderConfig
  ): Promise<OrderResponse> {
    // 1. 提取原始交易参数
    const { token_id, side, price, size } = originalTrade;
    
    // 2. 应用跟单倍数
    const copySize = size * config.sizeMultiplier;
    
    // 3. 验证约束条件
    this.validateOrder(copySize, config);
    
    // 4. 检查余额
    await this.checkBalance(copySize);
    
    // 5. 构建市价单
    const order = await this.buildMarketOrder({
      token_id,
      side,
      amount: copySize
    });
    
    // 6. 签名并提交
    const result = await this.submitOrder(order);
    
    // 7. 记录订单
    await this.recordOrder(result);
    
    return result;
  }
  
  private async buildMarketOrder(params: OrderParams) {
    const marketOrder: MarketOrderArgs = {
      token_id: params.token_id,
      amount: params.amount,
      side: params.side,
      order_type: OrderType.FOK // Fill-or-Kill
    };
    
    return await this.clobClient.create_market_order(marketOrder);
  }
  
  private async submitOrder(order: SignedOrder) {
    try {
      const response = await this.clobClient.post_order(
        order,
        OrderType.FOK
      );
      
      return response;
    } catch (error) {
      await this.handleOrderError(error);
      throw error;
    }
  }
}
```

**订单类型支持**:

- **FOK (Fill-or-Kill)**: 立即全部成交或取消
- **FAK (Fill-and-Kill)**: 立即部分成交,未成交取消
- **GTC (Good-til-Cancelled)**: 限价单,持续有效
- **GTD (Good-til-Date)**: 限价单,有效期至指定时间

### 4.3 Risk Manager Service (风控管理)

**职责**: 多层风险控制

```typescript
interface RiskLimits {
  maxDailyLoss: number;           // 每日最大亏损
  maxPositionSize: number;        // 单个持仓最大值
  maxDailyTrades: number;         // 每日最大交易次数
  maxDrawdown: number;            // 最大回撤
  allowNegativeRisk: boolean;     // 是否允许负风险市场
  positionLimitPerMarket: number; // 单市场仓位限制
}

class RiskManager {
  async validateTrade(trade: Trade, config: RiskLimits): Promise<boolean> {
    // 1. 余额检查
    if (!await this.checkSufficientBalance(trade)) {
      throw new InsufficientBalanceError();
    }
    
    // 2. 单笔订单限额
    if (trade.size > config.maxPositionSize) {
      throw new OrderSizeExceededError();
    }
    
    // 3. 日交易次数限制
    if (!await this.checkDailyTradeCount(config.maxDailyTrades)) {
      throw new DailyLimitExceededError();
    }
    
    // 4. 日亏损限制
    const dailyPnL = await this.getDailyPnL();
    if (dailyPnL < -config.maxDailyLoss) {
      throw new DailyLossExceededError();
    }
    
    // 5. 回撤检查
    const drawdown = await this.calculateDrawdown();
    if (drawdown > config.maxDrawdown) {
      throw new DrawdownExceededError();
    }
    
    // 6. 市场黑名单
    if (this.isBlacklisted(trade.token_id)) {
      throw new BlacklistedMarketError();
    }
    
    // 7. 负风险市场检查
    if (!config.allowNegativeRisk && await this.isNegativeRisk(trade.market_id)) {
      throw new NegativeRiskError();
    }
    
    return true;
  }
  
  private async checkSufficientBalance(trade: Trade): Promise<boolean> {
    const balance = await this.getUSDCBalance();
    const requiredAmount = trade.size * trade.price;
    
    // 预留 5% 的 gas 费用
    return balance >= requiredAmount * 1.05;
  }
}
```

**风控层级**:

1. **订单前检查**: 余额、限额、黑名单
2. **仓位级风控**: 单市场仓位、总仓位
3. **账户级风控**: 日亏损、回撤、交易频率
4. **紧急熔断**: 异常检测自动停止

### 4.4 Position Manager Service (持仓管理)

**职责**: 持仓同步与自动赎回

```typescript
class PositionManager {
  async syncPositions() {
    // 每 4 秒扫描一次
    setInterval(async () => {
      await this.reconcilePositions();
    }, 4000);
  }
  
  private async reconcilePositions() {
    // 1. 获取目标用户持仓
    const targetPositions = await this.getTargetUserPositions();
    
    // 2. 获取自己的持仓
    const myPositions = await this.getMyPositions();
    
    // 3. 计算差异
    const diff = this.calculatePositionDiff(
      targetPositions,
      myPositions
    );
    
    // 4. 调整持仓
    for (const [tokenId, sizeDiff] of diff.entries()) {
      if (Math.abs(sizeDiff) > this.config.threshold) {
        await this.adjustPosition(tokenId, sizeDiff);
      }
    }
  }
  
  private async adjustPosition(tokenId: string, sizeDiff: number) {
    const side = sizeDiff > 0 ? Side.BUY : Side.SELL;
    const size = Math.abs(sizeDiff);
    
    await this.orderEngine.placeOrder({
      token_id: tokenId,
      side,
      size
    });
  }
  
  async autoRedeem() {
    // 每 2 小时检查一次
    setInterval(async () => {
      const positions = await this.getResolvedPositions();
      
      for (const pos of positions) {
        if (pos.winning_outcome) {
          await this.redeemPosition(pos);
        }
      }
    }, 2 * 3600 * 1000);
  }
}
```

### 4.5 Strategy Engine (策略引擎)

**职责**: 多种跟单策略

```typescript
enum CopyStrategy {
  MIRROR = 'mirror',           // 完全镜像
  PROPORTIONAL = 'proportional', // 比例跟单
  SELECTIVE = 'selective',     // 选择性跟单
  ANTI = 'anti'                // 反向跟单
}

interface StrategyConfig {
  type: CopyStrategy;
  multiplier: number;
  filters: {
    minTradeSize?: number;
    maxTradeSize?: number;
    allowedMarkets?: string[];
    minConfidence?: number;  // 最小概率阈值
  };
}

class StrategyEngine {
  async processTradeByStrategy(
    trade: Trade,
    strategy: StrategyConfig
  ): Promise<Order | null> {
    switch (strategy.type) {
      case CopyStrategy.MIRROR:
        return this.mirrorStrategy(trade, strategy);
        
      case CopyStrategy.PROPORTIONAL:
        return this.proportionalStrategy(trade, strategy);
        
      case CopyStrategy.SELECTIVE:
        return this.selectiveStrategy(trade, strategy);
        
      case CopyStrategy.ANTI:
        return this.antiStrategy(trade, strategy);
        
      default:
        throw new Error('Unknown strategy');
    }
  }
  
  private async selectiveStrategy(
    trade: Trade,
    config: StrategyConfig
  ): Promise<Order | null> {
    // 仅复制符合条件的交易
    const market = await this.getMarketInfo(trade.market_id);
    
    // 过滤条件
    if (trade.size < config.filters.minTradeSize) return null;
    if (trade.size > config.filters.maxTradeSize) return null;
    if (!config.filters.allowedMarkets.includes(trade.market_id)) return null;
    if (market.probability < config.filters.minConfidence) return null;
    
    return this.buildOrder(trade, config.multiplier);
  }
}
```

---

## 5. 数据流设计

### 5.1 实时交易流

```
1. WebSocket 接收消息
   ↓
2. 消息解析 & 验证
   ↓
3. 内存去重检查
   ↓ (新交易)
4. Risk Manager 验证
   ↓ (通过)
5. Order Engine 构建订单
   ↓
6. CLOB API 提交订单
   ↓
7. 本地 JSON / MongoDB 记录结果
```

### 5.2 持仓同步流

```
定时触发 (4秒/次)
   ↓
1. 并行获取目标用户 & 自己的持仓
   ↓
2. 计算持仓差异
   ↓
3. 生成调整订单列表
   ↓
4. 批量提交订单
   ↓
5. 更新本地持仓记录
```

---

## 6. 数据模型设计

### 6.1 本地 JSON 存储 (推荐,简单)

根据实际开源实现,使用本地 JSON 文件存储即可:

```typescript
// src/data/token-holding.json
interface TokenHolding {
  [tokenId: string]: {
    size: number;
    avg_price: number;
    last_updated: number;
  };
}

// src/data/credential.json (自动生成)
interface APICredential {
  key: string;
  secret: string;
  passphrase: string;
}

// src/data/trade-history.json
interface TradeHistory {
  trades: Array<{
    trade_id: string;
    timestamp: number;
    token_id: string;
    side: 'BUY' | 'SELL';
    size: number;
    price: number;
    status: string;
  }>;
}
```

### 6.2 MongoDB Collections (可选,复杂场景)

如果需要更强大的查询和统计功能,可使用 MongoDB:

#### 6.2.1 trades (交易记录)

```typescript
interface TradeDocument {
  _id: ObjectId;
  trade_id: string;
  original_trader: string;
  token_id: string;
  market_id: string;
  side: 'BUY' | 'SELL';
  original_size: number;
  original_price: number;
  copied_size: number;
  copied_price: number;
  status: 'pending' | 'confirmed' | 'failed';
  tx_hash?: string;
  created_at: Date;
  latency_ms: number;
}

// 索引
db.trades.createIndex({ trade_id: 1 }, { unique: true });
db.trades.createIndex({ created_at: -1 });
```

#### 6.2.2 positions (持仓表)

```typescript
interface PositionDocument {
  _id: ObjectId;
  token_id: string;
  market_id: string;
  size: number;
  avg_entry_price: number;
  last_updated: Date;
}

db.positions.createIndex({ token_id: 1 }, { unique: true });
```

---

## 7. API集成方案

### 7.1 Polymarket CLOB API

**基础配置**:

```typescript
interface ClobConfig {
  host: 'https://clob.polymarket.com';
  chain_id: 137;                // Polygon
  ws_url: 'wss://ws-subscriptions-clob.polymarket.com';
  private_key: string;
  funder_address: string;       // Polymarket 代理地址
  signature_type: 0 | 1 | 2;    // 0: EOA, 1: Email, 2: Browser
}
```

**认证流程**:

```typescript
class ClobAuthService {
  async initialize() {
    // 1. 创建 signer
    const signer = new Wallet(this.config.private_key);
    
    // 2. 创建临时客户端
    const tempClient = new ClobClient(
      this.config.host,
      this.config.chain_id,
      signer
    );
    
    // 3. 生成 API 凭证 (L1 认证)
    const apiCreds = await tempClient.create_api_key();
    
    // 4. 使用凭证创建正式客户端
    const client = new ClobClient(
      this.config.host,
      this.config.chain_id,
      signer,
      apiCreds,
      this.config.signature_type,
      this.config.funder_address
    );
    
    return client;
  }
}
```

**核心 API 方法**:

```typescript
class ClobAPIWrapper {
  // 获取用户交易历史
  async getUserTrades(address: string, params?: {
    start_time?: number;
    end_time?: number;
    limit?: number;
  }): Promise<Trade[]> {
    return await this.client.getTrades({
      maker: address,
      ...params
    });
  }
  
  // 获取用户持仓
  async getUserPositions(address: string): Promise<Position[]> {
    return await this.client.get_positions({
      user: address
    });
  }
  
  // 获取市场信息
  async getMarket(condition_id: string): Promise<Market> {
    return await this.client.get_market(condition_id);
  }
  
  // 获取订单簿
  async getOrderBook(token_id: string): Promise<OrderBook> {
    return await this.client.get_order_book(token_id);
  }
  
  // 创建市价单
  async createMarketOrder(params: {
    token_id: string;
    side: Side;
    amount: number;
  }): Promise<SignedOrder> {
    const order = await this.client.create_market_order({
      token_id: params.token_id,
      amount: params.amount,
      side: params.side
    });
    
    return order;
  }
  
  // 提交订单
  async postOrder(
    order: SignedOrder,
    orderType: OrderType = OrderType.FOK
  ): Promise<OrderResponse> {
    return await this.client.post_order(order, orderType);
  }
  
  // 取消订单
  async cancelOrder(order_id: string): Promise<void> {
    return await this.client.cancel_order(order_id);
  }
}
```

### 7.2 WebSocket 订阅方案

**CLOB WebSocket**:

```typescript
class ClobWebSocketClient {
  private ws: WebSocket;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 10;
  
  connect() {
    this.ws = new WebSocket(
      'wss://ws-subscriptions-clob.polymarket.com/ws/user',
      {
        headers: {
          'Authorization': `Bearer ${this.apiKey}`
        }
      }
    );
    
    this.ws.on('open', () => {
      console.log('WebSocket connected');
      this.reconnectAttempts = 0;
      this.subscribe();
    });
    
    this.ws.on('message', (data) => {
      this.handleMessage(JSON.parse(data));
    });
    
    this.ws.on('error', (error) => {
      console.error('WebSocket error:', error);
    });
    
    this.ws.on('close', () => {
      this.handleReconnect();
    });
  }
  
  private subscribe() {
    // 订阅用户交易
    this.ws.send(JSON.stringify({
      type: 'subscribe',
      channel: 'user',
      auth: {
        address: this.targetAddress
      },
      markets: ['*']  // 订阅所有市场
    }));
  }
  
  private handleReconnect() {
    if (this.reconnectAttempts < this.maxReconnectAttempts) {
      const delay = Math.min(1000 * Math.pow(2, this.reconnectAttempts), 30000);
      this.reconnectAttempts++;
      
      setTimeout(() => {
        console.log(`Reconnecting... Attempt ${this.reconnectAttempts}`);
        this.connect();
      }, delay);
    }
  }
}
```

**RTDS (Real-Time Data Stream)**:

```typescript
import { RealTimeDataClient, Message } from '@polymarket/real-time-data-client';

class RTDSClient {
  private client: RealTimeDataClient;
  
  initialize() {
    this.client = new RealTimeDataClient({
      onMessage: this.handleMessage.bind(this),
      onConnect: this.handleConnect.bind(this),
      onDisconnect: this.handleDisconnect.bind(this),
      onError: this.handleError.bind(this)
    });
  }
  
  private handleConnect(client: RealTimeDataClient) {
    // 订阅活动流
    client.subscribe({
      subscriptions: [
        {
          topic: 'activity',
          type: 'trades',
          filters: JSON.stringify({
            makerAddress: this.targetAddress
          })
        }
      ]
    });
  }
  
  private handleMessage(message: Message) {
    if (message.type === 'trades') {
      const trades = message.payload as Trade[];
      trades.forEach(trade => {
        this.processTrade(trade);
      });
    }
  }
}
```

### 7.3 Polygon RPC 交互

```typescript
class PolygonRPCClient {
  private provider: JsonRpcProvider;
  
  constructor(rpcUrl: string) {
    this.provider = new JsonRpcProvider(rpcUrl);
  }
  
  // 获取 USDC 余额
  async getUSDCBalance(address: string): Promise<BigNumber> {
    const usdcContract = new Contract(
      USDC_CONTRACT_ADDRESS,
      ERC20_ABI,
      this.provider
    );
    
    return await usdcContract.balanceOf(address);
  }
  
  // 批准 USDC 支出
  async approveUSDC(spender: string, amount: BigNumber) {
    const usdcContract = new Contract(
      USDC_CONTRACT_ADDRESS,
      ERC20_ABI,
      this.signer
    );
    
    const tx = await usdcContract.approve(spender, amount);
    return await tx.wait();
  }
  
  // 获取交易状态
  async getTransactionStatus(txHash: string) {
    const receipt = await this.provider.getTransactionReceipt(txHash);
    return receipt?.status === 1 ? 'success' : 'failed';
  }
}
```

---

## 8. 风险控制系统

### 8.1 多层风控架构

```
订单前验证 → 仓位检查 → 账户级风控 → 市场风险 → 异常检测
```

### 8.2 风控规则配置

```typescript
interface RiskControlConfig {
  // 订单级别
  order: {
    minSize: number;              // 最小订单 0.1 USDC
    maxSize: number;              // 最大订单 1000 USDC
    maxSlippage: number;          // 最大滑点 5%
  };
  
  // 仓位级别
  position: {
    maxPositionPerMarket: number; // 单市场最大 500 USDC
    maxTotalPosition: number;     // 总仓位最大 5000 USDC
    maxMarketsCount: number;      // 最大市场数 50
  };
  
  // 账户级别
  account: {
    maxDailyLoss: number;         // 日最大亏损 1000 USDC
    maxDailyTrades: number;       // 日最大交易 200 笔
    maxDrawdown: number;          // 最大回撤 30%
    minBalance: number;           // 最低余额 50 USDC
  };
  
  // 市场级别
  market: {
    allowNegativeRisk: boolean;   // 禁止负风险市场
    minLiquidity: number;         // 最小流动性 1000 USDC
    maxSpread: number;            // 最大买卖价差 10%
    blacklist: string[];          // 市场黑名单
  };
}
```

### 8.3 实时风控监控

```typescript
class RiskMonitor {
  private metrics = {
    dailyTrades: 0,
    dailyVolume: 0,
    dailyPnL: 0,
    currentDrawdown: 0,
    peakBalance: 0,
    failedOrders: 0
  };
  
  async checkAndAlert() {
    // 每分钟检查一次
    setInterval(async () => {
      await this.updateMetrics();
      await this.checkThresholds();
    }, 60000);
  }
  
  private async checkThresholds() {
    const alerts = [];
    
    // 检查日亏损
    if (this.metrics.dailyPnL < -this.config.maxDailyLoss) {
      alerts.push({
        level: 'critical',
        type: 'DAILY_LOSS_EXCEEDED',
        message: `日亏损超限: ${this.metrics.dailyPnL} USDC`,
        action: 'STOP_TRADING'
      });
    }
    
    // 检查回撤
    if (this.metrics.currentDrawdown > this.config.maxDrawdown) {
      alerts.push({
        level: 'warning',
        type: 'DRAWDOWN_HIGH',
        message: `回撤过大: ${this.metrics.currentDrawdown}%`
      });
    }
    
    // 检查失败率
    const failureRate = this.metrics.failedOrders / this.metrics.dailyTrades;
    if (failureRate > 0.2) {
      alerts.push({
        level: 'warning',
        type: 'HIGH_FAILURE_RATE',
        message: `订单失败率: ${(failureRate * 100).toFixed(2)}%`
      });
    }
    
    // 发送告警
    for (const alert of alerts) {
      await this.sendAlert(alert);
      
      if (alert.action === 'STOP_TRADING') {
        await this.emergencyStop();
      }
    }
  }
  
  private async emergencyStop() {
    // 1. 停止所有交易监控
    await this.tradeMonitor.stop();
    
    // 2. 取消所有挂单
    await this.orderEngine.cancelAllOrders();
    
    // 3. 通知管理员
    await this.notifyAdmin('EMERGENCY_STOP');
    
    // 4. 记录日志
    logger.critical('Emergency stop triggered', this.metrics);
  }
}
```

### 8.4 黑名单管理

```typescript
class BlacklistManager {
  private marketBlacklist: Set<string> = new Set();
  private tokenBlacklist: Set<string> = new Set();
  
  // 添加市场到黑名单
  async addMarket(marketId: string, reason: string) {
    this.marketBlacklist.add(marketId);
    
    await this.db.collection('blacklist').insertOne({
      type: 'market',
      id: marketId,
      reason,
      added_at: new Date()
    });
  }
  
  // 检查是否在黑名单
  isBlacklisted(marketId: string, tokenId: string): boolean {
    return this.marketBlacklist.has(marketId) || 
           this.tokenBlacklist.has(tokenId);
  }
  
  // 自动黑名单 (低流动性市场)
  async autoBlacklist() {
    const markets = await this.getAllMarkets();
    
    for (const market of markets) {
      if (market.liquidity < this.config.minLiquidity) {
        await this.addMarket(
          market.id,
          `Low liquidity: ${market.liquidity}`
        );
      }
    }
  }
}
```

---

## 9. 性能优化方案

### 9.1 缓存策略

```typescript
class CacheManager {
  private redis: Redis;
  
  // 多层缓存
  async getMarketInfo(marketId: string): Promise<Market> {
    // L1: 内存缓存
    if (this.memoryCache.has(marketId)) {
      return this.memoryCache.get(marketId);
    }
    
    // L2: Redis 缓存
    const cached = await this.redis.get(`market:${marketId}`);
    if (cached) {
      const market = JSON.parse(cached);
      this.memoryCache.set(marketId, market);
      return market;
    }
    
    // L3: API 调用
    const market = await this.clobClient.get_market(marketId);
    
    // 写入缓存
    await this.redis.setex(
      `market:${marketId}`,
      600,  // 10分钟
      JSON.stringify(market)
    );
    this.memoryCache.set(marketId, market);
    
    return market;
  }
}
```

### 9.2 批量处理

```typescript
class BatchProcessor {
  private orderQueue: Order[] = [];
  private batchSize = 10;
  private batchInterval = 1000; // 1秒
  
  constructor() {
    this.startBatchProcessing();
  }
  
  async addOrder(order: Order) {
    this.orderQueue.push(order);
    
    // 达到批量大小立即处理
    if (this.orderQueue.length >= this.batchSize) {
      await this.processBatch();
    }
  }
  
  private startBatchProcessing() {
    setInterval(async () => {
      if (this.orderQueue.length > 0) {
        await this.processBatch();
      }
    }, this.batchInterval);
  }
  
  private async processBatch() {
    const batch = this.orderQueue.splice(0, this.batchSize);
    
    // 并行提交订单
    const results = await Promise.allSettled(
      batch.map(order => this.submitOrder(order))
    );
    
    // 处理结果
    results.forEach((result, index) => {
      if (result.status === 'fulfilled') {
        this.onSuccess(batch[index], result.value);
      } else {
        this.onError(batch[index], result.reason);
      }
    });
  }
}
```

### 9.3 连接池管理

```typescript
class ConnectionPoolManager {
  private wsPool: WebSocket[] = [];
  private poolSize = 5;
  
  async initialize() {
    for (let i = 0; i < this.poolSize; i++) {
      const ws = await this.createConnection();
      this.wsPool.push(ws);
    }
  }
  
  getConnection(): WebSocket {
    // 轮询策略
    const index = this.currentIndex % this.poolSize;
    this.currentIndex++;
    return this.wsPool[index];
  }
  
  async handleConnectionFailure(ws: WebSocket) {
    const index = this.wsPool.indexOf(ws);
    if (index !== -1) {
      // 重建连接
      const newWs = await this.createConnection();
      this.wsPool[index] = newWs;
    }
  }
}
```

### 9.4 数据库优化

```typescript
// 使用聚合管道优化查询
async getDailyStats(date: Date) {
  return await this.db.collection('trades').aggregate([
    {
      $match: {
        created_at: {
          $gte: startOfDay(date),
          $lt: endOfDay(date)
        }
      }
    },
    {
      $group: {
        _id: null,
        total_trades: { $sum: 1 },
        total_volume: { $sum: '$copied_size' },
        avg_latency: { $avg: '$latency_ms' },
        success_rate: {
          $avg: {
            $cond: [{ $eq: ['$status', 'confirmed'] }, 1, 0]
          }
        }
      }
    }
  ]).toArray();
}

// 使用投影减少数据传输
async getRecentTrades(limit: number = 100) {
  return await this.db.collection('trades')
    .find(
      {},
      {
        projection: {
          trade_id: 1,
          copied_size: 1,
          status: 1,
          created_at: 1
        }
      }
    )
    .sort({ created_at: -1 })
    .limit(limit)
    .toArray();
}
```

---

## 10. 安全设计

### 10.1 私钥管理

```typescript
class KeyManager {
  private kms: AWS.KMS;  // 使用 AWS KMS 或类似服务
  
  async getPrivateKey(): Promise<string> {
    // 从 KMS 解密私钥
    const encrypted = process.env.ENCRYPTED_PRIVATE_KEY;
    const params = {
      CiphertextBlob: Buffer.from(encrypted, 'base64'),
      KeyId: process.env.KMS_KEY_ID
    };
    
    const result = await this.kms.decrypt(params).promise();
    return result.Plaintext.toString('utf8');
  }
  
  // 或使用环境变量 + Vault
  async getPrivateKeyFromVault(): Promise<string> {
    const vault = new Vault({
      endpoint: process.env.VAULT_ADDR,
      token: process.env.VAULT_TOKEN
    });
    
    const secret = await vault.read('secret/polymarket/private-key');
    return secret.data.key;
  }
}
```

### 10.2 API 认证

```typescript
class APIAuth {
  private jwtSecret: string;
  
  // 生成 JWT token
  generateToken(userId: string): string {
    return jwt.sign(
      { userId, role: 'user' },
      this.jwtSecret,
      { expiresIn: '24h' }
    );
  }
  
  // 验证中间件
  authMiddleware(req, res, next) {
    const token = req.headers.authorization?.split(' ')[1];
    
    if (!token) {
      return res.status(401).json({ error: 'No token provided' });
    }
    
    try {
      const decoded = jwt.verify(token, this.jwtSecret);
      req.user = decoded;
      next();
    } catch (error) {
      return res.status(401).json({ error: 'Invalid token' });
    }
  }
}
```

### 10.3 速率限制

```typescript
class RateLimiter {
  private redis: Redis;
  
  async checkLimit(
    key: string,
    limit: number,
    windowMs: number
  ): Promise<boolean> {
    const current = await this.redis.incr(key);
    
    if (current === 1) {
      await this.redis.expire(key, Math.ceil(windowMs / 1000));
    }
    
    return current <= limit;
  }
  
  // Express 中间件
  rateLimitMiddleware(limit: number, windowMs: number) {
    return async (req, res, next) => {
      const key = `ratelimit:${req.ip}:${Date.now() / windowMs}`;
      
      if (await this.checkLimit(key, limit, windowMs)) {
        next();
      } else {
        res.status(429).json({ error: 'Too many requests' });
      }
    };
  }
}
```

### 10.4 输入验证

```typescript
import { z } from 'zod';

// 订单参数验证
const orderSchema = z.object({
  token_id: z.string().regex(/^0x[a-fA-F0-9]{64}$/),
  side: z.enum(['BUY', 'SELL']),
  size: z.number().positive().max(10000),
  market_id: z.string().min(1)
});

class InputValidator {
  validateOrder(data: unknown) {
    try {
      return orderSchema.parse(data);
    } catch (error) {
      throw new ValidationError('Invalid order parameters', error);
    }
  }
}
```

---

## 11. 部署架构

### 11.1 服务器配置

**最小配置**:
- **CPU**: 2 核心
- **内存**: 4GB RAM
- **存储**: 20GB SSD
- **网络**: 稳定网络连接
- **位置**: 荷兰 VPS (避免地理限制,推荐 tradingvps.io)

### 11.2 快速部署

**安装依赖**:

```bash
# 1. 安装 Node.js 18+
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# 2. 安装 PM2 (可选,用于后台运行)
npm install -g pm2
```

**项目部署**:

```bash
# 1. 克隆项目
git clone <your-repo>
cd polymarket-copy-trading

# 2. 安装依赖
npm install

# 3. 配置环境变量
cp .env.example .env
nano .env  # 编辑配置

# 4. 构建项目
npm run build

# 5. 运行 (开发模式)
npm start

# 6. 或使用 PM2 (生产模式)
pm2 start dist/index.js --name "polymarket-bot"
pm2 startup
pm2 save
```

### 11.3 PM2 管理命令 (可选)

```bash
# 查看状态
pm2 status

# 查看日志
pm2 logs polymarket-bot

# 重启
pm2 restart polymarket-bot

# 停止
pm2 stop polymarket-bot

# 监控
pm2 monit
```

---

## 12. 监控与日志 (可选)

### 12.1 基础日志

```typescript
// 简单的控制台日志即可
console.log('[TRADE]', {
  timestamp: new Date().toISOString(),
  trade_id: trade.id,
  action: 'copy',
  status: 'success'
});

console.error('[ERROR]', {
  timestamp: new Date().toISOString(),
  error: error.message,
  stack: error.stack
});
```

### 12.2 关键指标跟踪

```typescript
// 内存中跟踪基本指标
const metrics = {
  total_trades: 0,
  successful_trades: 0,
  failed_trades: 0,
  total_volume: 0
};

// 定期输出统计
setInterval(() => {
  console.log('[STATS]', {
    ...metrics,
    success_rate: (metrics.successful_trades / metrics.total_trades * 100).toFixed(2) + '%',
    uptime: process.uptime()
  });
}, 60000); // 每分钟
```

---

## 13. 容灾与备份 (可选)

### 13.1 数据备份

```bash
# 如果使用 JSON 文件
# 定期备份数据文件夹
tar -czf backup-$(date +%Y%m%d).tar.gz src/data/

# 如果使用 MongoDB
mongodump --uri="mongodb://localhost:27017/polymarket" --out=./backup
```

### 13.2 故障恢复

```typescript
// 添加自动重启逻辑
process.on('uncaughtException', (error) => {
  console.error('[FATAL]', error);
  process.exit(1); // PM2 会自动重启
});

process.on('unhandledRejection', (reason) => {
  console.error('[UNHANDLED REJECTION]', reason);
});
```

---

## 14. 开发计划

### 14.1 第一阶段:核心功能 (1-2周)

**Week 1: 基础实现**
- [ ] 项目初始化,环境配置
- [ ] CLOB Client 集成与认证
- [ ] WebSocket 实时监控
- [ ] 交易事件解析与去重

**Week 2: 订单执行**
- [ ] Order Engine 订单构建
- [ ] 市价单提交逻辑
- [ ] 本地 JSON 存储
- [ ] 基础风控检查

### 14.2 第二阶段:优化与测试 (1周)

**Week 3: 完善功能**
- [ ] 持仓同步服务
- [ ] 自动赎回机制
- [ ] 错误处理与重试
- [ ] 测试网测试

### 14.3 第三阶段:生产部署 (1周)

**Week 4: 上线准备**
- [ ] VPS 服务器配置
- [ ] PM2 进程管理
- [ ] 小额主网测试
- [ ] 监控日志完善

---

## 附录

### A. 环境变量配置

```bash
# .env.example

# Blockchain
PRIVATE_KEY=0x...
PROXY_WALLET=0x...
FUNDER_ADDRESS=0x...
RPC_URL=https://polygon-mainnet.infura.io/v3/YOUR_KEY

# Polymarket
CLOB_HTTP_URL=https://clob.polymarket.com
CLOB_WS_URL=wss://ws-subscriptions-clob.polymarket.com
SIGNATURE_TYPE=0  # 0: EOA, 1: Email, 2: Browser

# Target Users (comma separated)
TARGET_USERS=0x...,0x...

# Risk Management
SIZE_MULTIPLIER=0.1
MAX_ORDER_SIZE=1000
MIN_ORDER_SIZE=0.1
MAX_DAILY_LOSS=1000
MAX_DAILY_TRADES=200
MAX_DRAWDOWN=0.3

# Database
MONGO_URI=mongodb://admin:password@localhost:27017/polymarket
REDIS_URL=redis://localhost:6379
RABBITMQ_URL=amqp://admin:password@localhost:5672

# Monitoring
ENABLE_PROMETHEUS=true
PROMETHEUS_PORT=9090
GRAFANA_PORT=3001

# Alerts
ALERT_WEBHOOK_URL=https://hooks.slack.com/services/...
ADMIN_EMAIL=admin@example.com

# Logging
LOG_LEVEL=info
LOG_DIR=/app/logs
```

### B. API 端点列表

```typescript
// RESTful API
GET    /api/v1/health              // 健康检查
GET    /api/v1/metrics             // Prometheus 指标

GET    /api/v1/targets             // 目标用户列表
POST   /api/v1/targets             // 添加目标用户
PUT    /api/v1/targets/:id         // 更新配置
DELETE /api/v1/targets/:id         // 删除目标用户

GET    /api/v1/trades              // 交易历史
GET    /api/v1/trades/:id          // 交易详情

GET    /api/v1/positions           // 持仓列表
GET    /api/v1/positions/:tokenId  // 持仓详情

GET    /api/v1/stats/daily         // 日统计
GET    /api/v1/stats/summary       // 汇总统计

POST   /api/v1/controls/pause      // 暂停交易
POST   /api/v1/controls/resume     // 恢复交易
POST   /api/v1/controls/emergency  // 紧急停止

// WebSocket API
WS     /ws/trades                  // 实时交易流
WS     /ws/positions               // 实时持仓更新
WS     /ws/alerts                  // 实时告警
```

### C. 错误代码表

```typescript
enum ErrorCode {
  // 系统错误 (1xxx)
  SYSTEM_ERROR = 1000,
  
  // 网络错误 (2xxx)
  WEBSOCKET_DISCONNECTED = 2000,
  API_TIMEOUT = 2001,
  RPC_ERROR = 2002,
  
  // 认证错误 (3xxx)
  INVALID_SIGNATURE = 3000,
  INSUFFICIENT_BALANCE = 3001,
  ALLOWANCE_NOT_SET = 3002,
  
  // 订单错误 (4xxx)
  ORDER_REJECTED = 4000,
  INVALID_ORDER_SIZE = 4001,
  MARKET_CLOSED = 4002,
  INSUFFICIENT_LIQUIDITY = 4003,
  
  // 风控错误 (5xxx)
  BLACKLISTED_MARKET = 5003
}
```

### D. 项目文件结构

```
polymarket-copy-trading/
├── src/
│   ├── index.ts              # 主入口
│   ├── data/                 # 本地数据存储
│   │   ├── credential.json   # API 凭证(自动生成)
│   │   ├── token-holding.json # 持仓数据
│   │   └── trade-history.json # 交易历史
│   ├── services/
│   │   ├── monitor.ts        # 交易监控服务
│   │   ├── executor.ts       # 订单执行服务
│   │   └── position-sync.ts  # 持仓同步服务
│   ├── providers/
│   │   ├── clobClient.ts     # CLOB API 客户端
│   │   ├── wssProvider.ts    # WebSocket 提供者
│   │   └── rpcProvider.ts    # RPC 提供者
│   ├── order-builder/
│   │   ├── builder.ts        # 订单构建器
│   │   └── types.ts          # 类型定义
│   ├── security/
│   │   └── allowance.ts      # Token 授权管理
│   └── utils/
│       ├── logger.ts         # 日志工具
│       └── config.ts         # 配置管理
├── package.json
├── tsconfig.json
├── .env.example
└── README.md
```

### E. 性能基准 (简化版)

| 指标 | 目标值 | 说明 |
|------|--------|------|
| 交易延迟 (平均) | < 2s | WebSocket 模式 |
| 订单成功率 | > 90% | 正常市场条件 |
| 系统稳定性 | 连续运行 | 自动重连 |

### E. 故障排查清单

```markdown
## WebSocket 连接失败
- [ ] 检查网络连接
- [ ] 验证 API 凭证
- [ ] 检查地理位置限制 (使用荷兰 VPS)
- [ ] 查看连接日志

## 订单提交失败
- [ ] 检查 USDC 余额
- [ ] 验证 Token Allowance (运行 approve-usdc.js)
- [ ] 检查订单参数
- [ ] 查看 API 返回错误

## 持仓不同步
- [ ] 检查同步服务是否运行
- [ ] 手动对比目标用户持仓
- [ ] 检查网络连接
- [ ] 重启服务

## 系统性能问题
- [ ] 检查 CPU/内存使用 (top 命令)
- [ ] 查看日志文件大小
- [ ] 检查网络延迟
- [ ] PM2 重启服务
```

---

## 总结

本架构设计文档基于实际的 Polymarket 跟单系统开源实现,采用**极简架构**:

**核心特点**:
1. **简单直接**: 单一进程,本地 JSON 存储
2. **快速部署**: 5 分钟即可运行
3. **低成本**: 2核4G 服务器即可
4. **高可靠**: WebSocket + 自动重连

**基础配置**:
- Node.js + TypeScript
- 本地 JSON 文件存储
- WebSocket 实时监控
- PM2 进程守护

**可选扩展**:
- MongoDB (需要复杂查询时)
- Nginx (需要多实例负载均衡时)
- 监控系统 (需要详细指标时)

**开发时间**: 3-4 周即可完成基础版本

**注意事项**:
- 使用荷兰 VPS (tradingvps.io)
- 先在测试网充分测试
- 小额资金开始
- 做好私钥安全管理

---

**文档维护**:
- 版本: 1.0 (简化版)
- 最后更新: 2025年12月
- 基于实际开源实现
- 所有设计均可验证