### websocket推送
```
// websocket连接地址

// 测试网地址
wss://stream-testnet.bybit.com/realtime

// 主网地址
wss://stream.bybit.com/realtime

```

### 连接限制

一个api_key可同时建立20个连接,超过20个连接后再建立连接会被拒绝

### 身份验证

对于公共类topic不需要进行身份验证即可订阅,个人类topic则需要先进行身份验证

目前有两种方式进行身份验证

1. 在建立连接的请求上附加上身份验证信息进行认证
2. 建立连接后通过auth指令认证

```js
// 第一种认证方式
var api_key = "";
var secret = "";
// http请求的失效时间，防止重放攻击
// 单位:毫秒
var expires = time.now()+1000;

// 消息加签
var signature = hex(HMAC_SHA256(secret, 'GET/realtime' + expires));

// 参数列表
var param = "api_key={api_key}&expires={expires}&signature={signature}";

// 建立连接
var ws = new WebSocket("wsurl?param");

// --------------------------------------------------------------------------

// 第二种认证方式
var ws = new WebSocket("wsurl")
// signature加签方式与第一种方式一致
ws.send('{"op":"auth","args":["{api_key}","{expires}","{signature}"]}');
```

### <span id="signature-algorithm">签名算法示例</span>

* [C#](example/Encryption.cs)
* [Python](example/Encryption.py)
* [C++](example/Encryption.cpp)
* [Go](example/Encryption.go)


### 如何发送业务心跳包

**网络不稳定或者程序异常等因素会导致websocket连接异常断开，所以我们强烈建议您使用Ping来维持连接，且对锻炼的情况进行处理**

连接建立后，通过发送json格式的心跳包来进行心跳探测,具体格式如下
```js
ws.send('{"op":"ping"}');

// 返应格式如下
{
    "success":true,
    "ret_msg":"pong",
    "request":{
        "op":"ping",
        "args":null
    }
}
```

### 如何订阅topic

连接建立后，通过发送json格式的订阅指令来进行订阅topic,具体格式如下
```js
ws.send('{"op":"subscribe","args":["topic","topic.filter"]}');

// 同一个类型的filter有多个时，以'|'分割
// 如订阅BTCUSD一分钟和三分钟的kline
ws.send('{"op":"subscribe","args":["kline.BTCUSD.1m|3m"]}');

// 订阅同一个类型filter的所有数据时请使用'*'
// 如订阅所有产品的所有interval kline
ws.send('{"op":"subscribe","args":["kline.*.*"]}')


// 订阅topic结果
// 每一个订阅指令都会有一个响应,响应格式如下
{
    "success":true, // 订阅是否成功
    "ret_msg":"",   // 订阅成功时为空，失败时为具体错误信息
    "request":{     // 请求订阅的指令
        "op":"subscribe",
        "args":[
            "kline.BTCUSD.1m"
        ]
    }
}

```

## 目前支持的topic

### 公共类topic
* ~~[orderBook25](#orderBook25) `// 25档orderBook`~~ -----这是过时的，推荐使用V2版本的[orderBookL2_25](#orderBook25_v2)
* ~~[kline](#kline) `// K线`~~  -----已过期，推荐使用最新v2版本的[klineV2](#kline_v2)
* [trade](#trade) `// 实时交易`
* [insurance](#insurance) `// 每日保险基金更新`
* ~~[instrument](#instrument) `// 产品最新信息`~~ -----这是过时的，推荐使用V2版本的[instrument_info](#instrument_info)

### 新版行情topic
* [orderBookL2_25](#orderBook25_v2) `// 25档orderBook`
* [orderBookL2_200](#orderBook200_v2) `// 200档orderBook`
* [instrument_info](#instrument_info) `//合约信息`
* [klineV2](#kline_v2) `// 新K线topic`

### 个人类topic
* [position](#position) `// 仓位变化`
* [execution](#execution) `// 委托单成交信息`
* [order](#order) `// 委托单的更新`
* [stop_order](#stop-order) `// 条件委托更新`


<hr>

### <span id="orderBook25">订阅25档orderBook</span>
```js
// 发送订阅指令
ws.send('{"op": "subscribe", "args": ["orderBook25.BTCUSD"]}');

// 推送的消息格式
{
    "topic":"orderBook25",
    "action":"snapshot",
    "data":{
        "lastUpdateId":709,
        "symbol":"BTCUSD",
        "bids":[],
        "asks":[
            {"price":6400,"quantity":38},
            {"price":6300,"quantity":5},
        ]
    }
}

```

<hr>

### <span id="kline">k线</span>

* 目前支持的interval
* 1m 3m 5m 15m 30m
* 1h 2h 3h 4h 6h
* 1d 3d
* 1w 2w
* 1M
```js
ws.send('{"op":"subscribe","args":["kline.BTCUSD.1m"]}');

// 推送的消息格式
{
    "topic":"kline.BTCUSD.1m",
    "data":{
        "id":563,
        "symbol":"BTCUSD",
        "open_time":1539918000,
        "open":5900,
        "high":6501,
        "low":6501,
        "close":6501,
        "volume":9,
        "turnover":0.0013844,
        "interval":"1m"
    }
}
```

<hr>

### <span id="trade">实时交易信息</span>

```js
 ws.send('{"op":"subscribe","args":["trade"]}')

 // 推送的消息格式
{
    "topic":"trade.BTCUSD",
    "data":[
        {
            "timestamp":"2019-01-22T15:04:33.461Z",
            "symbol":"BTCUSD",
            "side":"Buy",
            "size":980,
            "price":3563.5,
            "tick_direction":"PlusTick",
            "trade_id":"9d229f26-09a8-42f8-aff3-0ba047b0449d",
            "cross_seq":163261271
        }
    ]
}
```

<hr>

### <span id="insurance">每日保险基金更新</span>

```js
ws.send('{"op":"subscribe","args":["insurance"]}')

// 推送的消息格式
 {
     "topic":"insurance.BTC",
     "action":"update",
     "data":{
        "currency":"BTC",
        "timestamp":"2018-10-24T12:00:00.000Z",
        "wallet_balance":140224705439 // 单位:聪
     }
 }
```

<hr>

### <span id="instrument">产品最新行情</span>

```js
ws.send('{"op":"subscribe","args":["instrument.BTCUSD"]}')

// 推送的消息格式
// NOTE: 这里data下的字段只在其变化时推送，无变化时推送的数据无该字段
// 比如index_price和mark_price变化了，而成交价没有变化，则推送的数据只有symbol、index_price、mark_price而没有last_price
 {
     "topic":"instrument.BTCUSD",
     "data":{
        "symbol": "BTCUSD",
        "mark_price": 5000.5, // 标记价格
        "index_price": 5000.5, // 指数价格
        "last_price": 5000.5 // 最新成交价
     }
 }
```
<hr>


### <span id="orderBook25_v2">订阅新版25档orderBook</span>
```js
// 发送订阅指令 以BTCUSD为例
// orderBookL2_25.BTCUSD 
ws.send('{"op": "subscribe", "args": ["orderBookL2_25.BTCUSD"]}');

// 推送的消息格式 
// 当在一条websocket连接上订阅成功后返回第一条消息的type是snapshot类型的
// 后续的消息的type均是delta类型的，在接收到的snapshot消息上进行计算得到最新的orderbook

//snapshot类型消息格式,data里的数据按价格排序，从buy到sell方向
{
     "topic":"orderBookL2_25.BTCUSD",
     "type":"snapshot",
     "data":[
        {
            "price":"2999.00",
            "symbol":"BTCUSD",
            "id":29990000,
            "side":"Buy",
            "size":9
        },
        {
            "price":"3001.00",
            "symbol":"BTCUSD",
            "id":30010000,
            "side":"Sell",
            "size":10
        }
     ],
     "cross_seq":11518,
     "timestamp_e6":1555647164875373
}

//delta类型的消息包含delete update insert三类数据，
//价格档根据price字段或id字段来作为唯一标示
//delta类型的消息包含delete update insert三类数据，
//delete表示某个或多个价格档挂单量变为0
//update表示某个或多个价格档的size更新
//insert表示新增某个或多个价格档的挂挡
 
//delta类型消息格式
{
     "topic":"orderBookL2_25.BTCUSD",
     "type":"delta",
     "data":{
          "delete":[
			 {
                   "price":"3001.00",
                   "symbol":"BTCUSD",
                   "id":30010000,
                   "side":"Sell"
             }
          ],
          "update":[
             {
                   "price":"2999.00",
                   "symbol":"BTCUSD",
                   "id":29990000,
                   "side":"Buy",
                   "size":8
             }
          ],
          "insert":[
             {
                   "price":"2998.00",
                   "symbol":"BTCUSD",
                   "id":29980000,
                   "side":"Buy",
                   "size":8
             }
          ],
          "transactTimeE6":0
     },
     "cross_seq":11519,
     "timestamp_e6":1555647221331673
}

```
orderbook由两个方向分别为buy和sell的列表组成，列表的键为价格，列表的值为数量。

当接收到snapshot类型包时，清空之前所维护的orderbook,并以此snapshot包为基础开始进行演算,开始构建新的orderbook。后续在连接不断开的情况下，只会收到delta类型的数据包，delta数据包包含三类数据（delete,update,insert），每类数据中都包含方向，根据此方向来指定修改的orderbook里的键值列表，先处理delete数据，再处理update与insert的数据。

delete表示在相应方向的列表中此价格的挂单档位挂单数目变为0，update表示在相应方向的列表中此价格的挂单档位数量修改至最新的size，insert表示在相应方向的列表中增加此价格的挂单档位且数量为size的值。

<hr>

### <span id="orderBook200_v2">新版200档orderBook</span>
```js
// 发送订阅指令 以BTCUSD为例
ws.send('{"op": "subscribe", "args": ["orderBook_200.100ms.BTCUSD"]}');
//snapshot类型消息格式,data里的数据按价格排序，从buy到sell方向
{
     "topic":"orderBook_200.100ms.BTCUSD",
     "type":"snapshot",
     "data":[
        {
            "price":"2999.00",
            "symbol":"BTCUSD",
            "id":29990000,
            "side":"Buy",
            "size":9
        },
        {
            "price":"3001.00",
            "symbol":"BTCUSD",
            "id":30010000,
            "side":"Sell",
            "size":10
        }
     ],
     "cross_seq":11518,
     "timestamp_e6":1555647164875373
}

//delta类型的消息包含delete update insert三类数据，
//价格档根据price字段或id字段来作为唯一标示
//delta类型的消息包含delete update insert三类数据，
//delete表示某个或多个价格档挂单量变为0
//update表示某个或多个价格档的size更新
//insert表示新增某个或多个价格档的挂挡

//delta类型消息格式
{
     "topic":"orderBook_200.100ms.BTCUSD",
     "type":"delta",
     "data":{
          "delete":[
			 {
                   "price":"3001.00",
                   "symbol":"BTCUSD",
                   "id":30010000,
                   "side":"Sell"
             }
          ],
          "update":[
             {
                   "price":"2999.00",
                   "symbol":"BTCUSD",
                   "id":29990000,
                   "side":"Buy",
                   "size":8
             }
          ],
          "insert":[
             {
                   "price":"2998.00",
                   "symbol":"BTCUSD",
                   "id":29980000,
                   "side":"Buy",
                   "size":8
             }
          ],
          "transactTimeE6":0
     },
     "cross_seq":11519,
     "timestamp_e6":1555647221331673
}
```

<hr>

### <span id="instrument_info">产品最新行情</span>

```js
ws.send('{"op":"subscribe","args":["instrument_info.100ms.BTCUSD"]}')

// 推送的消息格式 先收到snapshot包，当连接未断开时后续只收到delta包
// snapshot包格式如下 e4代表乘以10^4，e6代表乘以10^6
{
	"topic": "instrument_info.100ms.BTCUSD",
	"type": "snapshot",
	"data": {
		"id": 1,
		"symbol": "BTCUSD",                     //合约名字
		"last_price_e4": 100000000,             //最新市价
		"last_tick_direction": "ZeroPlusTick",  //价格变化方向:PlusTick,ZeroPlusTick,MinusTick,ZeroMinusTick
		"prev_price_24h_e4": 100000000,         //24小时前的整点市价
		"price_24h_pcnt_e6": 0,                 //市价相对24h变化百分比
		"high_price_24h_e4": 100000000,         //24h最高价
		"low_price_24h_e4": 58000000,           //24h最低价
		"prev_price_1h_e4": 71000000,           //1小时前的整点市价
		"price_1h_pcnt_e6": 408450,             //市价相对1小时前变化百分比
		"mark_price_e4": 96758100,              //标记价格
		"index_price_e4": 97000000,             //指数价格
		"open_interest": 158666,                //未平仓合约数量，更新频率相对较慢，最慢一分钟更新一次
		"open_value_e8": 2004325380,            //未平仓价值，更新频率相对较慢，最慢一分钟更新一次
		"total_turnover_e8": 257108049130,      //总营业额(BTC价值)
		"turnover_24h_e8": 8969373218,          //24小时营业额(BTC价值)
		"total_volume": 15462289,               //总交易量(合约数)
		"volume_24h": 541359,                   //24小时交易量(合约数)
		"funding_rate_e6": -3750,               //资金费率
		"predicted_funding_rate_e6": -3750,     //预测资金费率
		"cross_seq": 7980,                      //序列号
		"created_at": "2018-10-17T11:53:15Z",   
		"updated_at": "2019-07-30T03:12:42Z",
		"next_funding_time": "2019-07-30T08:00:00Z",//下次结算资金费用时间
		"countdown_hour": 5                     //剩余时间去结算资金费用
	},
	"cross_seq": 7980,
	"timestamp_e6": 1564456370126493            //行情推送时间
}
// delta包格式如下 只有在data字段下的update字段里有值，e4代表乘以10^4，e6代表乘以10^6，当instrument_info里的字段没有发生变化时相应的字段不进行推送
{
	"topic": "instrument_info.100ms.BTCUSD",
	"type": "delta",
	"data": {
		"delete": [],
		"update": [{
			"id": 1,
			"symbol": "BTCUSD",
			"total_turnover_e8": 257108059130,
			"turnover_24h_e8": 8969383218,
			"total_volume": 15462290,
			"volume_24h": 541360,
			"cross_seq": 7981,
			"created_at": "2018-10-17T11:53:15Z",
			"updated_at": "2019-07-30T03:12:52Z"
		}],
		"insert": []
	},
	"cross_seq": 7981,
	"timestamp_e6": 1564456372227451
}
```

<hr>

### <span id="kline_v2">kline_v2 topic</span>

* 目前支持的K线周期
* 1 3 5 15 30
* 60 120 240 360
* D
* W
* M

**如果`confirm`字段为`true`，意味着这是这根K线的最后一个tick.否则，这只是一个快照数据，即中间价格**

```js

ws.send('{"op":"subscribe","args":["klineV2.1.BTCUSD"]}')

// Response content format
{
    'topic': 'klineV2.1.BTCUSD',                //topic名
    'data': [{
        'start': 1572425640,                    //K线的起始时间
        'end': 1572425700,                      //K线的结束时间
        'open': 9200,                           //开盘价
        'close': 9202.5,                        //收盘价
        'high': 9202.5,                         //最高价
        'low': 9196,                            //最低价
        'volume': 81790,                        //成交量
        'turnover': 8.889247899999999,          //成交额
        'confirm': False,                       //是否为快照数据标志
        'cross_seq': 297503466,                 
        'timestamp': 1572425676958323           //撮合时间
    }],
    'timestamp_e6': 1572425677047994            //服务器推送时间
}
```

<hr>

### <span id="position">仓位变化消息</position>

```js
ws.send('{"op":"subscribe","args":["position"]}')

// 推送的消息格式
{
    "topic":"position",
    "action":"update",
    "data":[
        {
            "user_id": 1,                       // 用户id
            "symbol":"BTCUSD",                  // 产品
            "size":11,                          // 数量
            "side":"Sell",                      // 方向
            "position_value":'0.00159252',        // 仓位名义价值
            "entry_price":'6907.291588174717',    // 开仓价
            "liq_price":'7100.234',               // 强平价
            "bust_price":'7088.1234',             // 破产价
            "leverage":'1',                       // 杠杆
            "order_margin": '1',                  // 委托预占用保证金
            "position_margin": '1',               // 仓位保证金
            "available_balance": '2',             // 可用余额
            "take_profit":'0',                    // 止盈价格
            "tp_trigger_by": '2',                 // 由什么价格来触发止盈，如：最新价格、标记价格等; 非条件单不返回该字段
            "stop_loss":'0',                      // 止损价格
            "sl_trigger_by": '2',                 // 由什么价格来触发止损，如：最新价格、标记价格等；非条件单不返回该字段
            "realised_pnl": '0.10',               // 今日已结盈亏
            "trailing_stop":'0',                  // 追踪止损点数
            "wallet_balance": '4.12',             // 账户余额
            "risk_id": 1,                       
            "occ_closing_fee": '0.1',             // 平仓手续费
            "occ_funding_fee": '0.1',             // 预占用资金费用
            "auto_add_margin":0,                // 是否自动追加保证金(0:否 1:是)
            "cum_realised_pnl": '0.12',           // Total realized profit and loss
            "position_status":"Normal",         // 仓位状态(Normal:正常 Liq:强平中 Adl:被减仓中)
            "position_seq":14                   // 仓位版本号
        }
    ]
}
```


<hr>

### <span id="execution">成交信息</span>
```js
 ws.send('{"op":"subscribe","args":["execution"]}')

 // 推送的消息格式
{
    "topic":"execution",
    "data":[
        {
            "symbol":"BTCUSD",
            "side":"Sell",
            "order_id":"095f99d2-844c-4358-9a8f-4a973eb5c418",
            "exec_id":"ce6e3ec9-0fb1-4972-8b66-c3d2fcd352f6",
            "order_link_id":"4119012214309",
            "price":3559,
            "exec_qty":1028,
            "exec_fee":-0.00007221,
            "leaves_qty":0,
            "is_maker":true,
            "trade_time":"2019-01-22T14:49:38.000Z"
        },
    ]
}

```

<hr>

### <span id="order">委托更新</span>

```js
 ws.send('{"op":"subscribe","args":["order"]}')

 // 推送的消息格式
{
    "topic":"order",
    "data":[
        {
            "order_id":"xxxxxxxx-xxxx-xxxx-832b-1eca710bf0a6",
            "order_link_id":"xxxxxxxx",
            "symbol":"BTCUSD",
            "side":"Sell",
            "order_type":"Limit",
            "price":3559.5,
            "qty":850,
            "time_in_force":"GoodTillCancel",
            "create_type":"CreateByUser",
            "cancel_type":"CancelByUser",
            "order_status":"Cancelled",
            "leaves_qty":0,
            "cum_exec_qty":0,
            "cum_exec_value":0,
            "cum_exec_fee":0,
            "timestamp":"2019-01-22T14:49:38.000Z"
        }
    ]
}
```


<hr>

### <span id="stop-order">条件委托更新</span>

```js
ws.send('{"op":"subscribe","args":["stop_order"]}')

// 推送的消息格式
{
  "topic": "stop_order",
  "data": [{
    "order_id": "795d87a1-db49-4fd5-acd9-062cc45bfad9",
    "order_status": "Untriggered",
    "stop_order_type": "StopLoss",
    "symbol": "BTCUSD",
    "side": "Buy",
    "qty": 1000,
    "user_id": 529950,
    "price": 0,
    "order_type": "Market",
    "time_in_force": "ImmediateOrCancel",
    "trigger_price": 9270,
    "trigger_by": "LastPrice"
  }],
  "user_id": 529950
}

```
