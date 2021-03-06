# TradingView使用教程  

## 运行demo

```bash
# 将项目clone到本地
git clone https://github.com/PutChaiKo/How-To-Use-TradingView.git
# 进入项目目录
cd ./How-To-Use-TradingView
# 安装依赖
npm i
# 运行
npm run dev
```

## TradingView是什么  

TradingView（下称TV）是一个免费的图表工具，用于显示类似于股票行情的图表，在加密货币领域这个工具非常受欢迎，很多网站都用了这个工具。  
TV的github地址在[这里](https://github.com/tradingview/charting_library/wiki)，如果你打开看到是404页面，那是正常的，这是因为TV虽然是免费的，但是它也是一个私有github项目，你可以通过官网申请这个私有项目的权限来使用它。  
申请权限的网址在[这里](https://www.tradingview.com/HTML5-stock-forex-bitcoin-charting-library/)。  
申请的时候要签个文档，和一些对于使用在哪里简单的介绍就行  
> 我申请的时候还出了点小问题，通过邮件沟通发现他们还有中文团队  

## 注意事项  

1. 这个工具本身并不会提供数据，要使用它你需要自己准备好数据源，如果你想要一个提供市场行情的工具，你可以去它官网看看，里面貌似有付费的提供数据的服务。
2. 官方有提供很丰富的英文使用说明，也有人把它[翻译成了中文](https://b.aitrade.ga/books/tradingview/)，但是官方说明对新手不友好。不仅是新手程序员，里面还涉及到一些金融行业的一些专业名词和金融数据特有的表述方式，还有这个工具自定义的一些表述模糊的使用方法，这也是本文要理清的问题。
3. 这个文章只是随手记录为了防止自己忘记怎么使用TV，具体请参考官方文档。

## vue 与 vue-cli  

vue是什么就不多介绍了，用vue脚手架工具新建一个空的vue项目做为本次教程的前端框架。然后将charting_library文件夹放在vue项目的/static/custorm_scripts目录里。  
vue会有个问题，TV是一个普通的外部js文件，并不像vue常用的npm安装包和有export写法的js文件可以直接引用。  
这里使用在 index.html 插入 script 引用的方法引入外部js。  

```html
  <script src="./static/custorm_scripts/charting_library/charting_library.min.js"></script>
```

然后在组件里使用全局变量的形式来引用  

```javascript
const TradingView = window.TradingView
```

这个形式在单页面demo里没有问题，但是多页面的时候会导致不相关页面都引用了多余的js，还会有污染全局变量的影响，怎么解决这个问题可以自行搜索vue如何引入外部js。

## TradingView.widget() 小部件  
这个工具的一切开端是构建一个 TradingView.widget()，在你需要展示的vue组件里添加一个methods和mounted

```javascript
<script>
// datafeed.js文件内容在文章的下面
import datafeed from './datafeed'

export default {
  data() {
    return {
      options: {
        debug: false,
        symbol: "Coinbase:BTC/USD",
        datafeed: datafeed,
        interval: "15",
        container_id: "div_container_id",
        library_path: "/static/custorm_scripts/charting_library/",
        locale: "en",
        disabled_features: [],
        enabled_features: [],
        client_id: "test",
        user_id: "id",
        fullscreen: false,
        autosize: true,
        overrides: {}
      }
    };
  },
  methods: {
    constructWidget() {
      const widget = new window.TradingView.widget(
        this.options
      );
      widget.onChartReady(() => {
        console.log("TV图表加载成功");
      });
    }
  },
  mounted() {
    window.onload = () => {
      this.constructWidget();
    }
  }
};
</script>
```

详细解释一下发生了什么事情，从vue生命周期mounted()开始，首先监控window是不是已经正确加载完毕，然后传入一个对象options到全局变量window.TradingView.widget()里构建一个widget，widget根据options.container_id将构建好的iframe放到一个id为div_container_id的div里  
其中如果不监控window.onload，貌似mounted会比window.TradingView.widget加载完成得更快，将会报错。  
options对象包含了主要的选项

```javascript
    // debug模式的开关，开发模式下开启更容易确定错误原因
    debug: false,
    // 在加密货币行业叫做交易对，在传统股市是指某一股对应某种法定货币的价格
    // 数据来源需要什么格式的symbol就传什么
    symbol: "Coinbase:BTC/USD",
    // 处理数据的函数，组件成功构建后会马上调用这个函数
    // 后面会详细说明
    datafeed: datafeed,
    // 初始显示的时间精度
    interval: "15",
    // 需要在页面准备好一个id为div_container_id的空div
    container_id: "div_container_id",
    // charting_library所在的文件夹路径，里面有TV运行需要的文件
    // 如果配置不当将无法运行TV
    library_path: "/static/custorm_scripts/charting_library/",
    // 语言，官方文档有支持的语言表，包括简繁中文
    locale: "en",
    // 自定义组件的一些功能不显示，具体选项请查看官方文档
    disabled_features: [],
    // 同上，只是反过来，这里会覆盖上面的设置
    enabled_features: [],
    client_id: "test",
    user_id: "id",
    // 全屏模式
    fullscreen: false,
    autosize: true,
    overrides: {}
```

## Datafeed 数据来源  

官方提供了两种数据接入的方式，UDF和JavaScript API，这里使用JS API的方法处理数据。
直接贴上datafeed.js文件

```javascript
export default {
  onReady: cb => {
    console.log("onReady已运行");
    setTimeout(
      () => cb(),
      0
    );
  },
  resolveSymbol: (symbolName, onSymbolResolvedCallback, onResolveErrorCallback) => {
    console.log("resolveSymbol运行，symbolName为: ", symbolName)
    var split_data = symbolName.split(/[:/]/)
    onSymbolResolvedCallback({
      name: symbolName,
      description: '',
      type: 'crypto',
      session: '24x7',
      timezone: 'Etc/UTC',
      ticker: symbolName,
      exchange: split_data[0],
      minmov: 1,
      pricescale: 100000000,
      has_intraday: true,
      intraday_multipliers: ['1', '60'],
      supported_resolution: ["1", "3", "5", "15", "30", "60", "120", "240", "D"],
      volume_precision: 8,
      data_status: 'streaming',
    })
  },
  getBars: function (
    symbolInfo,
    resolution,
    from,
    to,
    onHistoryCallback,
    onErrorCallback,
    firstDataRequest
  ) {
    console.log('getBars被调用')
    onHistoryCallback([{
      "time": 1548274740000,
      "low": 3536.35,
      "high": 3540,
      "open": 3540,
      "close": 3537.22,
      "volume": 3.85
    }]);
  },
  subscribeBars: (symbolInfo, resolution, onRealtimeCallback, subscribeUID, onResetCacheNeededCallback) => {
    console.log('subscribeBars被调用')
  },
  calculateHistoryDepth: (resolution, resolutionBack, intervalBack) => {
    console.log('calculateHistoryDepth 被调用')
    return resolution < 60 ? {
      resolutionBack: 'D',
      intervalBack: '1'
    } : undefined
  },
}
```

这个文件被上面的vue文件引用之后TV就可以运行了，可以看到图标中显示了一条数据。打开控制台发现一直在刷新，这是因为里面只有一条写死的数据导致的不正常现象，我们先解释一下各个函数的功能，然后再来解决数据的问题。  

### onReady  

onReady是首先被调用的函数，这个函数有一个传入的callback函数，这个cb()函数用setTimeout强制立刻执行，执行的时候可以传入一个设置对象

```javascript
// 设置对象的示例，按需设置
cb({
    supports_search: true,
    supports_group_request: false,
    supports_marks: true,
    exchanges: [
        {value: "", name: "All Exchanges", desc: ""},
        {value: "XETRA", name: "XETRA", desc: "XETRA"},
        {value: "NSE", name: "NSE", desc: "NSE"}
    ],
    symbolsTypes: [
        {name: "All types", value: ""},
        {name: "Stock", value: "stock"},
        {name: "Index", value: "index"}
    ],
    // 数据来源支持的时间分辨率
    supportedResolutions: [ "1", "15", "30", "60", "D", "2D", "3D", "W", "3W", "M", '6M' ]
})
```

### resolveSymbol 设置交易的具体细项目  

resolveSymbol是用于解析symbol的函数，同时也是一个设置TV选项很重要的地方，这个函数传入3个参数，分别是 symbolName, onSymbolResolvedCallback, onResolveErrorCallback。  
如果把 symbolName 打印出来会发现它就是上面TV构建时候的设置的symbol字符串，将这个字符串处理好之后立刻执行cb函数onSymbolResolvedCallback(Symbology)，其中Symbology是一个写有设置的对象，将参考官方介绍进行简单的说明。  

```javascript
onSymbolResolvedCallback({
    // 这个是给用户看的，显示在图表的左上角
    // 以股票举例的话，这里写的应该是股票的名称
    name: symbolName,
    // 对于这个股票/交易对的说明或描述
    description: '',
    type: 'crypto',
    // 交易的时间，股票交易明显是工作日才开门，而虚拟货币就需要设置成24x7小时的交易模式
    session: '24x7',
    // 交易所的时区，具体需要填的字符串需要去文档查
    // 小知识，因为它所使用的时区格式在制定的过程中的历史原因，这个表格里面找不到北京时间是正常的
    timezone: 'Etc/UTC',
    // 这个是查询时候使用的，与name可以不同
    // 以股票来说的话，这里应该设置的是股票代码
    ticker: symbolName,
    // 显示在图表的左上角，一般是表示某个交易所的名字
    exchange: split_data[0],
    // 最小价格波动
    minmov: 1,
    // 价格精度
    pricescale: 100000000,
    // 是否有一日之内的历史数据
    has_intraday: true,
    // 日内数据的精度
    intraday_multipliers: ['1', '60'],
    // 支持的精度
    supported_resolution: ["1", "3", "5", "15", "30", "60", "120", "240", "D"],
    // 成交量显示的数字的小数位，如果是0则是整数，1则是0.1
    volume_precision: 8,
    // 显示在页面的右上角的状态，有固定几个可选，请参考官方文档
    data_status: 'streaming',
})
```

pricescale价格精度是一个很重要的参数，假设某个股票0.01块钱一股，那就要写成100，如果是加密电子货币，有些货币相互兑换的价格可以达到小数点后8位，那就要设置为100000000。  

## getBars 获取历史数据  

getBars带有很多个参数，其中：

### symbolInfo

打印在控制台就可以看到，它里面大部分信息都是在resolveSymbol函数处填入的数据，自己挑一些合适的参数出来用就行。  

### onHistoryCallback(bars, meta)

这是一个重要的cb函数，它接受两个参数，第一个是包含数据信息的数组，第二个是是关于数据的一些设置。  
bars的格式是这样子的：

```javascript
[
    {
        // 时间戳，精确到毫秒
        "time": 1548274740000,
        // 这里是股票交易概念里的高开低收
        "low": 3536.35,
        "high": 3540,
        "open": 3540,
        "close": 3537.22,
        // 成交量
        "volume": 3.85
    }，
    // 很多个类似的对象...
]
```

这里就需要一点点的金融知识，TV使用的是股票图表常用的”[美国线](https://zh.wikipedia.org/wiki/%E7%BE%8E%E5%9C%8B%E7%B7%9A)“，用一条竖着的线条来表示一个时间周期的开盘价、最高价、最低价、收盘价以及成交量，所以传输数据的时候也需要提供相应的参数，如果不想显示成交量可以将它设置为0,或者TV有其它隐藏成交量的方式我没发现。  
> 炒股的同学别取笑我，我是在使用TV的时候才懂得看K线图的

## 获取真正的数据  

用假数据总不是办法，前面不是说过要自备数据么，有不少免费提供数据的API，例如[CryptoCompare](https://min-api.cryptocompare.com/)，前端只需要自行接入就行，获取数据的思路是：

1. 先从api上面把数据取下来，整理好格式，放到getBars里呈现在TV上
2. 根据TV传来的不同的精度去调取相应精度的API获取相应精度的数据

vue使用里面引入[request-promise](https://github.com/request/request-promise)模块来获取数据，当然你也可以用自己喜欢的请求方式  

```bash
# request 是 request-promise 的依赖包
npm install --save request
npm install --save request-promise
```

新建一个getApiData.js文件专门获取和整理数据  

```javascript
const rp = require('request-promise');
export default (symbolInfo, resolution) => {
  console.log('getdata异步函数被调用', symbolInfo, resolution);
  const mainUrl = 'https://min-api.cryptocompare.com/data';
  // 处理时间精度
  let resolutionUrl;
  if (resolution === '1') {
    resolutionUrl = '/histominute?';
  } else if (resolution === '60') {
    resolutionUrl = '/histohour?';
  } else if (resolution === 'D' || resolution === '1D' ) {
    resolutionUrl = '/histoday?';
  } else {
    console.warn('supported_resolution设置不正确');
  }
  // 处理交易对
  const symbolArr = symbolInfo.name.split('/');
  const url = mainUrl + resolutionUrl + `fsym=${symbolArr[0]}&tsym=${symbolArr[1]}&limit=2000`;
  console.log(`调用的链接cryptocompare是：${url}`);
  return rp(url)
    .then((data) => {
      const formatData = JSON.parse(data).Data.map((item) => {
        const { close, high, low, open, time, volumefrom, volumeto } = item;
        return {
          time: time * 1000,
          low,
          high,
          open,
          close,
          volume: volumeto,
        };
      });
      return formatData;
    })
    .catch((err) => {
      console.warn('api调取错误', err);
    });
};
```


因为 cryptocompare 提供了三种精度的数据，分别为1分钟、1小时、1天，所以需要判断resolution参数来组成不同的请求链接。  
但是在TV的界面上可以发现它提供给用户的不止三个精度，从间隔15分钟到1月的数据都有，而且不做改动的话用户还可以随便选精度。这是因为TV将获取的数据进行了整合，如果用户想要5分钟间隔的图表，TV看到支持的精度 supported_resolution 数组里面没有的话就会去取低于目的精度的1分钟间隔的数据，然后拼成5分钟的间隔数据。   

其中关键的代码是  
```javascript
return rp(url).then((data) => {
  // 把data处理好格式后
  return data;
});
```
rq是一个Promise对象包装好的请求异步函数，将rp函数return回去也会获得一个新的Promise函数，再通过async await的方法异步请求这个函数。

## 获取历史数据的一些坑和补充的东西  

### 无限请求接口的问题

TV 的 getBars 函数里面有一个 from to 参数，还有一个 firstDataRequest 参数，前者的意思是TV想要这个时间段的数据，如果它获取的数据的时间戳没有覆盖from和to，就会导致TV不断地去调取接口，正如demo里面的cryptocompare，它不是按照时间范围，而是只按照条数来提供数据，最多2000条，如果是调取了分钟的接口，再被TV转换为45分钟间隔的数据，2000分钟也只有45条左右的有效数据，用户稍微缩小一下窗口就会引发无限更新。  
解决方法有很多，可以要求后台按照时间范围去吐数据，但是有时候后台也无能为力，例如加密货币在2014年才出现，如果以一个月为间隔一个图表轻松就能显示完成。
可以根据firstDataRequest参数来限制请求次数。  
可以在前台限制用户点击的时间精度，这个可以通过配置TV来实现，也可以通过禁用TV自带的选择框自己在前端写选择框来配置interval的数值，这样子比较合适不熟悉TV的人，但是每修改一个参数就要重新构建一遍TV，如果是更换语言等参数的话就只能这样子重修构建。  

### 搜索

TV还提供了很多工具，例如可以自己输入参数获取相应的数据，还可以添加标记，要一一接入还是很麻烦的，但是基本原理和getBar()差不多，如果要一一禁用，用外部参数重新构建TV的方法，要一个个禁用也是挺麻烦的，需要在官方文档一个个找设计。  

### 样式  

TV可自定义的样式的地方特别丰富，定义的时候一般是这样子的  

```javascript
overrides: {
  "paneProperties.background": "#131722",
  "paneProperties.vertGridProperties.color": "#363c4e",
  "paneProperties.horzGridProperties.color": "#363c4e",
  "symbolWatermarkProperties.transparency": 90,
  "scalesProperties.textColor" : "#AAA",
  "mainSeriesProperties.candleStyle.wickUpColor": '#336854',
  "mainSeriesProperties.candleStyle.wickDownColor": '#7f323f',
}
```

需要去官方文档翻各个地方的key，以及多试试。  

## 获取实时数据  

上面的配置已经已经可以做到获取数据了，是通过get请求的方式，如果愿意，写一个轮询的函数隔一段时间去请求一下历史数据也可以做到实时更新。  
不过轮询非常浪费资源，浪费用户的资源也浪费服务器的资源，最好的解决办法是通过WebSocket推送数据。  
CryptoCompare也支持WebSocket的方式获取实时数据，但它的[文档](https://min-api.cryptocompare.com/documentation?key=Streaming&cat=subsByPairEndpoint)写得并不是很清楚，不过好在它提供了[官方的demo](https://github.com/cryptoqween/cryptoqween.github.io) 

## WebSocket  

[WebSocket](https://zh.wikipedia.org/wiki/WebSocket)（下称ws）的一大特点是支持双向通信，从发布至今大部分浏览器都对它进行了兼容。  
要写一个简单的ws连接可以这样子写：  

```javascript
const socket = require('socket.io-client')('wss://streamer.cryptocompare.com');
socket.on('m', (e) => {console.log('ws', e)})
socket.emit('SubAdd', {subs: ["0~Coinbase~BTC~USD"]})
```

这是在nodejs环境下的写法，用到的库是socket.io-client（传统html写法可以参考上面的官方demo）。一步步来说，先用require的方式把库加载到当前环境里，这个库其实暴露出来的是一个函数，这个函数接收一个参数就是ws或wss打头的链接字符串，再把这个函数运行的结果赋值给变量socket。  
socket.on()接收两个参数，第一个参数是一个字符串，其中m代表的是move，也就是获得一条新数据的时候，第二个参数是一个回调函数，cb函数传进来的参数就是ws服务器发来的消息。  
写好了怎么处理拿到的数据后，就可以通过socket.emit()的方法向服务器请求连接并且发送参数，在这个代码里，SubAdd是用于激活ws服务端某些函数的代码，由双方约定好的，后面的对象是传给SubAdd对应后端函数的对象，在这个ws连接里可以在数组里面传多个字符串，字符串由~分割，第一个是ws的连接代码，第二个是交易所的代号，第三个是需要兑换的货币，第四个是基准货币。  