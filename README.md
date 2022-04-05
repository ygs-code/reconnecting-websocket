# Reconnecting WebSocket 源码分析

Reconnecting WebSocket 功能有

WebSocket，如果连接关闭，将自动重新连接。  

重连之后会根据为啥送消息，重新发送给服务器等功能。

Reconnecting WebSocket 也是基于WebSocket 二次封装的，里面用到了订阅发版，队列等思想封装。

## Features

-兼容WebSocket API(相同接口，Level0和Level2事件模型)  

——完全可配置  

-多平台(Web, ServiceWorkers, Node.js, React Native)  

-无依赖(不依赖Window, DOM或任何EventEmitter库)  

—处理连接超时  

-允许更改服务器URL之间的重新连接  

\- - -缓冲。 将发送累积的信息打开  

-多个版本可用(见dist文件夹)  

——调试模式  

## Install

```bash
npm install --save reconnecting-websocket
```

## 使用

- 兼容WebSocket浏览器API

- 所以这个文档应该是有效的:

[MDN WebSocket API](https://developer.mozilla.org/zh-CN/docs/Web/API/WebSocket).

 

### Simple usage

```javascript
import ReconnectingWebSocket from 'reconnecting-websocket';

const rws = new ReconnectingWebSocket('ws://my.site.com');

rws.addEventListener('open', () => {
    rws.send('hello!');
});
```

### Update URL

The `url` parameter will be resolved before connecting, possible types:

-   `string`
-   `() => string`
-   `() => Promise<string>`

```javascript
import ReconnectingWebSocket from 'reconnecting-websocket';

const urls = ['ws://my.site.com', 'ws://your.site.com', 'ws://their.site.com'];
let urlIndex = 0;

// round robin url provider
const urlProvider = () => urls[urlIndex++ % urls.length];

const rws = new ReconnectingWebSocket(urlProvider);
```

```javascript
import ReconnectingWebSocket from 'reconnecting-websocket';

// async url provider
const urlProvider = async () => {
    const token = await getSessionToken();
    return `wss://my.site.com/${token}`;
};

const rws = new ReconnectingWebSocket(urlProvider);
```

### Options

#### Sample with custom options

```javascript
import ReconnectingWebSocket from 'reconnecting-websocket';
import WS from 'ws';

const options = {
    WebSocket: WS, // custom WebSocket constructor
    connectionTimeout: 1000,
    maxRetries: 10,
};
const rws = new ReconnectingWebSocket('ws://my.site.com', [], options);
```

#### Available options 参数选项

```typescript
type Options = {
    WebSocket?: any; // WebSocket构造函数，如果没有提供，默认为全局WebSocket  
    maxReconnectionDelay?: number; // 重连之间的最大延迟(ms)
    minReconnectionDelay?: number; // 重连之间的最小延迟(ms) 连接上5000毫秒以上输入一个日志
    reconnectionDelayGrowFactor?: number; // 重连接延迟增长的速度有多快 意思是每次都会增加1.3倍数
    minUptime?: number;  // 连接上5000毫秒以上输入一个日志 稳定连接上5000毫秒 
    connectionTimeout?: number; // 连接超时间
    maxRetries?: number; // maximum number of retries  // 最大重复连接次数
    maxEnqueuedMessages?: number; // 在重新连接之前要缓冲区的最大消息数   最大的发送消息队列
    startClosed?: boolean; //在关闭状态下启动websocket，调用' .reconnect() '连接  
    debug?: boolean; // enables debug output 使调试输出
};
```

#### Default values 默认参数

```javascript
WebSocket: undefined,
maxReconnectionDelay: 10000,
minReconnectionDelay: 1000 + Math.random() * 4000,
reconnectionDelayGrowFactor: 1.3,
minUptime: 5000,
connectionTimeout: 4000,
maxRetries: Infinity,
maxEnqueuedMessages: Infinity,
startClosed: false,
debug: false,
```

## API

### Methods 方法

```typescript
constructor(url: UrlProvider, protocols?: string | string[], options?: Options)

close(code?: number, reason?: string)
reconnect(code?: number, reason?: string)

send(data: string | ArrayBuffer | Blob | ArrayBufferView)

addEventListener(type: 'open' | 'close' | 'message' | 'error', listener: EventListener)
removeEventListener(type:  'open' | 'close' | 'message' | 'error', listener: EventListener)
```

### Attributes 属性

[More info](https://developer.mozilla.org/zh-CN/docs/Web/API/WebSocket)

```typescript
binaryType: string;
bufferedAmount: number;
extensions: string;
onclose: EventListener;
onerror: EventListener;
onmessage: EventListener;
onopen: EventListener;
protocol: string;
readyState: number;
url: string;
retryCount: number;
```

### Constants 连接状态

```text
CONNECTING 0 The connection is not yet open.
OPEN       1 The connection is open and ready to communicate.
CLOSING    2 The connection is in the process of closing.
CLOSED     3 The connection is closed or couldn't be opened.
```

## 逐行源码分析

```
'use strict';

/*! *****************************************************************************
Copyright (c) Microsoft Corporation. All rights reserved.
Licensed under the Apache License, Version 2.0 (the "License"); you may not use
this file except in compliance with the License. You may obtain a copy of the
License at http://www.apache.org/licenses/LICENSE-2.0

THIS CODE IS PROVIDED ON AN *AS IS* BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, EITHER EXPRESS OR IMPLIED, INCLUDING WITHOUT LIMITATION ANY IMPLIED
WARRANTIES OR CONDITIONS OF TITLE, FITNESS FOR A PARTICULAR PURPOSE,
MERCHANTABLITY OR NON-INFRINGEMENT.

See the Apache Version 2.0 License for specific language governing permissions
and limitations under the License.
***************************************************************************** */
/* global Reflect, Promise */

var extendStatics = function (d, b) {
    extendStatics = Object.setPrototypeOf ||
        ({ __proto__: [] } instanceof Array && function (d, b) { d.__proto__ = b; }) ||
        function (d, b) { for (var p in b) if (b.hasOwnProperty(p)) d[p] = b[p]; };
    return extendStatics(d, b);
};

function __extends (d, b) {
    extendStatics(d, b);
    function __ () { this.constructor = d; }
    d.prototype = b === null ? Object.create(b) : (__.prototype = b.prototype, new __());
}

function __values (o) {
    var m = typeof Symbol === "function" && o[Symbol.iterator], i = 0;
    if (m) return m.call(o);
    return {
        next: function () {
            if (o && i >= o.length) o = void 0;
            return { value: o && o[i++], done: !o };
        }
    };
}

function __read (o, n) {
    var m = typeof Symbol === "function" && o[Symbol.iterator];
    if (!m) return o;
    var i = m.call(o), r, ar = [], e;
    try {
        while ((n === void 0 || n-- > 0) && !(r = i.next()).done) ar.push(r.value);
    }
    catch (error) { e = { error: error }; }
    finally {
        try {
            if (r && !r.done && (m = i["return"])) m.call(i);
        }
        finally { if (e) throw e.error; }
    }
    return ar;
}

function __spread () {
    for (var ar = [], i = 0; i < arguments.length; i++)
        ar = ar.concat(__read(arguments[i]));
    return ar;
}

var Event = /** @class */ (function () {
    function Event (type, target) {
        this.target = target;
        this.type = type;
    }
    return Event;
}());
var ErrorEvent = /** @class */ (function (_super) {
    __extends(ErrorEvent, _super);
    function ErrorEvent (error, target) {
        var _this = _super.call(this, 'error', target) || this;
        _this.message = error.message;
        _this.error = error;
        return _this;
    }
    return ErrorEvent;
}(Event));
var CloseEvent = /** @class */ (function (_super) {
    __extends(CloseEvent, _super);
    function CloseEvent (code, reason, target) {
        if (code === void 0) { code = 1000; }
        if (reason === void 0) { reason = ''; }
        var _this = _super.call(this, 'close', target) || this;
        _this.wasClean = true;
        _this.code = code;
        _this.reason = reason;
        return _this;
    }
    return CloseEvent;
}(Event));

/*!
 * Reconnecting WebSocket
 * by Pedro Ladaria <pedro.ladaria@gmail.com>
 * https://github.com/pladaria/reconnecting-websocket
 * License MIT
 */
var getGlobalWebSocket = function () {
    if (typeof WebSocket !== 'undefined') {
        // @ts-ignore
        return WebSocket;
    }
};
/**
 * Returns true if given argument looks like a WebSocket class
 */
var isWebSocket = function (w) { return typeof w !== 'undefined' && !!w && w.CLOSING === 2; };
var DEFAULT = {
    maxReconnectionDelay: 10000, // 最大的重连时间
    minReconnectionDelay: 1000 + Math.random() * 4000, // 重连 时间间隔
    minUptime: 5000, // 连接上5000毫秒以上输入一个日志
    reconnectionDelayGrowFactor: 1.3,  // 重连接延迟增长的速度有多快 意思是每次都会增加1.3倍数
    connectionTimeout: 4000, // 连接超时
    maxRetries: Infinity,  // 最大重复连接次数
    maxEnqueuedMessages: Infinity, // 最大的发送消息队列
    startClosed: false, //  在关闭状态下启动websocket，调用' .reconnect() '连接  
    debug: false, // 是否输出日志
};
var ReconnectingWebSocket = /** @class */ (function () {
    function ReconnectingWebSocket (
        url,  // 地址
        protocols,
        options // 拓展参数
    ) {
        var _this = this;
        if (options === void 0) { options = {}; }
        this._listeners = {
            error: [],
            message: [],
            open: [],
            close: [],
        };
        this._retryCount = -1;
        this._shouldReconnect = true;
        this._connectLock = false;
        this._binaryType = 'blob';
        this._closeCalled = false;
        this._messageQueue = [];
        /**
         * An event listener to be called when the WebSocket connection's readyState changes to CLOSED
         */
        this.onclose = null; // 关闭回调函数
        /**
         * An event listener to be called when an error occurs
         */
        this.onerror = null; // 错误回调函数
        /**
         * An event listener to be called when a message is received from the server
         */
        this.onmessage = null;  // 发送信息回调函数
        /**
         * An event listener to be called when the WebSocket connection's readyState changes to OPEN;
         * this indicates that the connection is ready to send and receive data
         */
        this.onopen = null;
        //WebSocket 连接 open 方法
        this._handleOpen = function (
            event //WebSocket 事件
        ) {
            _this._debug('open event');
            var _a = _this._options.minUptime,
                minUptime = _a === void 0 ? DEFAULT.minUptime : _a;
            // 清除 连接超时定时器
            clearTimeout(_this._connectTimeout);
            _this._uptimeTimeout = setTimeout(function () {
                return _this._acceptOpen();
            }, minUptime);
            _this._ws.binaryType = _this._binaryType;
            // send enqueued messages (messages sent before websocket open event)
            /*发送排队的消息(在websocket打开事件之前发送的消息)  
             这种情况就是在发生网络突然断开之后，用发送信息未能发送个服务器，然后先存到一个队列中
             等有网的时候连接上了再从队列中获取消息发送给服务器
             */
            _this._messageQueue.forEach(function (message) {
                // 给服务器发送信息
                return _this._ws.send(message);
            });
            // 清空队列
            _this._messageQueue = [];
            // 连接上回调方法
            if (_this.onopen) {
                _this.onopen(event);
            }
            //实例化对象连接注册事件
            _this._listeners.open.forEach(function (listener) {
                return _this._callEventListener(event, listener);
            });
        };
        // WebSocket发送Message 方法
        this._handleMessage = function (event) {
            _this._debug('message event');
            if (_this.onmessage) {
                _this.onmessage(event);
            }
            //
            _this._listeners.message.forEach(function (listener) {

                return _this._callEventListener(event, listener);
            });
        };
        // WebSocket Error 方法
        this._handleError = function (event) {
            _this._debug('error event', event.message);
            _this._disconnect(undefined, event.message === 'TIMEOUT' ? 'timeout' : undefined);
            if (_this.onerror) {
                _this.onerror(event);
            }
            _this._debug('exec error listeners');
            _this._listeners.error.forEach(function (listener) {
                return _this._callEventListener(event, listener);
            });
            _this._connect();
        };
        // WebSocket Close 方法
        this._handleClose = function (event) {
            _this._debug('close event');
            _this._clearTimeouts();
            if (_this._shouldReconnect) {
                _this._connect();
            }
            if (_this.onclose) {
                _this.onclose(event);
            }
            _this._listeners.close.forEach(function (listener) { return _this._callEventListener(event, listener); });
        };
        // 连接地址
        this._url = url;
        this._protocols = protocols;
        this._options = options;
        if (this._options.startClosed) {
            this._shouldReconnect = false;
        }
        // 连接WebSocket
        this._connect();
    }

    // 0
    Object.defineProperty(ReconnectingWebSocket, "CONNECTING", {
        get: function () {
            return 0;
        },
        enumerable: true,
        configurable: true
    });

    // 1
    Object.defineProperty(ReconnectingWebSocket, "OPEN", {
        get: function () {
            return 1;
        },
        enumerable: true,
        configurable: true
    });

    //2
    Object.defineProperty(ReconnectingWebSocket, "CLOSING", {
        get: function () {
            return 2;
        },
        enumerable: true,
        configurable: true
    });

    //3
    Object.defineProperty(ReconnectingWebSocket, "CLOSED", {
        get: function () {
            return 3;
        },
        enumerable: true,
        configurable: true
    });
    // 0
    Object.defineProperty(ReconnectingWebSocket.prototype, "CONNECTING", {
        get: function () {
            return ReconnectingWebSocket.CONNECTING;
        },
        enumerable: true,
        configurable: true
    });

    //1 
    Object.defineProperty(ReconnectingWebSocket.prototype, "OPEN", {
        get: function () {
            return ReconnectingWebSocket.OPEN;
        },
        enumerable: true,
        configurable: true
    });

    //2
    Object.defineProperty(ReconnectingWebSocket.prototype, "CLOSING", {
        get: function () {
            return ReconnectingWebSocket.CLOSING;
        },
        enumerable: true,
        configurable: true
    });

    //3
    Object.defineProperty(ReconnectingWebSocket.prototype, "CLOSED", {
        get: function () {
            return ReconnectingWebSocket.CLOSED;
        },
        enumerable: true,
        configurable: true
    });
    // 使用二进制连接
    Object.defineProperty(ReconnectingWebSocket.prototype, "binaryType", {
        get: function () {
            return this._ws ? this._ws.binaryType : this._binaryType;
        },
        set: function (value) {
            this._binaryType = value;
            if (this._ws) {
                this._ws.binaryType = value;
            }
        },
        enumerable: true,
        configurable: true
    });

    // 重复连接次数
    Object.defineProperty(ReconnectingWebSocket.prototype, "retryCount", {
        /**
         * Returns the number or connection retries
         */
        get: function () {
            return Math.max(this._retryCount, 0);
        },
        enumerable: true,
        configurable: true
    });

    // 未发送至服务器的字节数。
    Object.defineProperty(ReconnectingWebSocket.prototype, "bufferedAmount", {
        /**
         * The number of bytes of data that have been queued using calls to send() but not yet
         * transmitted to the network. This value resets to zero once all queued data has been sent.
         * This value does not reset to zero when the connection is closed; if you keep calling send(),
         * this will continue to climb. Read only
         */
        get: function () {
            var bytes = this._messageQueue.reduce(function (acc, message) {
                if (typeof message === 'string') {
                    acc += message.length; // not byte size
                }
                else if (message instanceof Blob) {
                    acc += message.size;
                }
                else {
                    acc += message.byteLength;
                }
                return acc;
            }, 0);
            return bytes + (this._ws ? this._ws.bufferedAmount : 0);
        },
        enumerable: true,
        configurable: true
    });

    // 服务器选择的扩展。
    Object.defineProperty(ReconnectingWebSocket.prototype, "extensions", {
        /**
         * The extensions selected by the server. This is currently only the empty string or a list of
         * extensions as negotiated by the connection
         */
        get: function () {
            return this._ws ? this._ws.extensions : '';
        },
        enumerable: true,
        configurable: true
    });

    // 服务器选择的下属协议。
    Object.defineProperty(ReconnectingWebSocket.prototype, "protocol", {
        /**
         * A string indicating the name of the sub-protocol the server selected;
         * this will be one of the strings specified in the protocols parameter when creating the
         * WebSocket object
         */
        get: function () {
            return this._ws ? this._ws.protocol : '';
        },
        enumerable: true,
        configurable: true
    });
    // 连接状态
    Object.defineProperty(ReconnectingWebSocket.prototype, "readyState", {
        /**
         * The current state of the connection; this is one of the Ready state constants
         */
        get: function () {
            if (this._ws) {
                return this._ws.readyState;
            }
            return this._options.startClosed
                ? ReconnectingWebSocket.CLOSED
                : ReconnectingWebSocket.CONNECTING;
        },
        enumerable: true,
        configurable: true
    });

    // 连接地址
    Object.defineProperty(ReconnectingWebSocket.prototype, "url", {
        /**
         * The URL as resolved by the constructor
         */
        get: function () {
            return this._ws ? this._ws.url : '';
        },
        enumerable: true,
        configurable: true
    });
    /**
     * Closes the WebSocket connection or connection attempt, if any. If the connection is already
     * CLOSED, this method does nothing
     */
    // WebSocket关闭回调函数
    ReconnectingWebSocket.prototype.close = function (code, reason) {
        if (code === void 0) { code = 1000; }
        this._closeCalled = true;
        this._shouldReconnect = false;
        this._clearTimeouts();
        if (!this._ws) {
            this._debug('close enqueued: no ws instance');
            return;
        }
        if (this._ws.readyState === this.CLOSED) {
            this._debug('close: already closed');
            return;
        }
        this._ws.close(code, reason);
    };
    /**
     * Closes the WebSocket connection or connection attempt and connects again.
     * Resets retry counter;
     * *关闭WebSocket连接或连接尝试，然后重新连接。  
     *重置重试计数器;  
     * 
     */
    //
    ReconnectingWebSocket.prototype.reconnect = function (code, reason) {
        this._shouldReconnect = true;
        this._closeCalled = false;
        this._retryCount = -1;
        if (!this._ws || this._ws.readyState === this.CLOSED) {
            this._connect();
        }
        else {
            this._disconnect(code, reason);
            this._connect();
        }
    };
    /**
     * Enqueue specified data to be transmitted to the server over the WebSocket connection
     *  将要通过WebSocket连接传输到服务器的指定数据放入队列  
     */
    ReconnectingWebSocket.prototype.send = function (data) {
        if (this._ws && this._ws.readyState === this.OPEN) {
            this._debug('send', data);
            this._ws.send(data);
        } else {
            var _a = this._options.maxEnqueuedMessages,
                maxEnqueuedMessages = _a === void 0 ? DEFAULT.maxEnqueuedMessages : _a;
            if (this._messageQueue.length < maxEnqueuedMessages) {
                this._debug('enqueue', data);
                this._messageQueue.push(data);
            }
        }
    };
    /**
     * Register an event handler of a specific event type
     * 注册一个特定事件类型的事件处理器  
     * 实例化对象注册 事件
        socket.addEventListener("open", function () {
                    statusSpan.innerHTML = "Connected";
                    element.style.backgroundColor = "white";
        });

        socket.addEventListener("open",{
                    handleEvent:function () {
                                statusSpan.innerHTML = "Connected";
                                element.style.backgroundColor = "white";
                    }
        });
     */
    ReconnectingWebSocket.prototype.addEventListener = function (type, listener) {
        //
        if (this._listeners[type]) {
            // @ts-ignore
            // 添加到一个队列中
            this._listeners[type].push(listener);
        }
    };
    //调度 事件
    ReconnectingWebSocket.prototype.dispatchEvent = function (event) {
        var e_1, _a;
        var listeners = this._listeners[event.type];
        if (listeners) {
            try {
                for (
                    var listeners_1 = __values(listeners),
                    listeners_1_1 = listeners_1.next();
                    !listeners_1_1.done;
                    listeners_1_1 = listeners_1.next()
                ) {
                    var listener = listeners_1_1.value;
                    this._callEventListener(event, listener);
                }
            } catch (e_1_1) {
                e_1 = { error: e_1_1 };
            } finally {
                try {
                    if (listeners_1_1 && !listeners_1_1.done && (_a = listeners_1.return)) {
                        _a.call(listeners_1);
                    }
                } finally {
                    if (e_1) {
                        throw e_1.error;

                    }
                }
            }
        }
        return true;
    };
    /**
     * Removes an event listener 删除事件监听器
     */
    ReconnectingWebSocket.prototype.removeEventListener = function (type, listener) {
        if (this._listeners[type]) {
            // @ts-ignore
            this._listeners[type] = this._listeners[type].filter(function (l) {
                return l !== listener;
            });
        }
    };
    // 日志输出方法
    ReconnectingWebSocket.prototype._debug = function () {
        var args = [];
        for (var _i = 0; _i < arguments.length; _i++) {
            args[_i] = arguments[_i];
        }
        // 如果参数中需要打印日志
        if (this._options.debug) {
            // not using spread because compiled version uses Symbols
            // tslint:disable-next-line
            console.log.apply(console, __spread(['RWS>'], args));
        }
    };
    // 根据重连次数而增加重连时长间隔
    ReconnectingWebSocket.prototype._getNextDelay = function () {
        var _a = this._options,
            _b = _a.reconnectionDelayGrowFactor,
            reconnectionDelayGrowFactor = _b === void 0 ? DEFAULT.reconnectionDelayGrowFactor : _b,
            _c = _a.minReconnectionDelay,
            minReconnectionDelay = _c === void 0 ? DEFAULT.minReconnectionDelay : _c, _d = _a.maxReconnectionDelay,
            maxReconnectionDelay = _d === void 0 ? DEFAULT.maxReconnectionDelay : _d;
        var delay = 0;

        if (this._retryCount > 0) {
            // 每次重连都会根据重连次数而增加重连时间 n 次方
            delay = minReconnectionDelay * Math.pow(reconnectionDelayGrowFactor, this._retryCount - 1);
            if (delay > maxReconnectionDelay) {
                delay = maxReconnectionDelay;
            }
        }
        this._debug('next delay', delay);
        console.log('delay=======', delay)
        return delay;
    };
    // 重连时间间隔设置
    ReconnectingWebSocket.prototype._wait = function () {
        var _this = this;
        return new Promise(function (resolve) {
            // 根据重连次数而增加重连时长间隔
            setTimeout(resolve, _this._getNextDelay());
        });
    };
    // 获取地址 可以是函数或者字符串
    ReconnectingWebSocket.prototype._getNextUrl = function (urlProvider) {
        if (typeof urlProvider === 'string') {
            return Promise.resolve(urlProvider);
        }
        if (typeof urlProvider === 'function') {
            var url = urlProvider();
            if (typeof url === 'string') {
                return Promise.resolve(url);
            }
            if (!!url.then) {
                return url;
            }
        }
        throw Error('Invalid URL');
    };
    //WebSocket 连接 方法
    ReconnectingWebSocket.prototype._connect = function () {
        var _this = this;
        // 如果有连接则不在连接
        if (this._connectLock || !this._shouldReconnect) {
            return;
        }
        this._connectLock = true;
        var _a = this._options,
            _b = _a.maxRetries,
            maxRetries = _b === void 0 ? DEFAULT.maxRetries : _b, _c = _a.connectionTimeout,
            connectionTimeout = _c === void 0 ? DEFAULT.connectionTimeout : _c,
            // options中的 WebSocket
            _d = _a.WebSocket,

            // 获取  WebSocket
            WebSocket = _d === void 0 ? getGlobalWebSocket() : _d;
        // 重复最大连接数
        if (this._retryCount >= maxRetries) {
            this._debug('max retries reached', this._retryCount, '>=', maxRetries);
            return;
        }
        this._retryCount++;
        this._debug('connect', this._retryCount);
        // 删除WebSocket事件
        this._removeListeners();

        if (!isWebSocket(WebSocket)) {
            throw Error('No valid WebSocket class provided');
        }
        // 重连时间间隔设置
        this._wait()
            .then(function () {
                // 获取连接地址可以是字符串或者函数
                return _this._getNextUrl(_this._url);
            })
            .then(function (
                url  // 获取连接地址
            ) {
                // close could be called before creating the ws
                // 可以在创建ws之前调用Close
                if (_this._closeCalled) {
                    return;
                }
                // 日志
                _this._debug('connect', { url: url, protocols: _this._protocols });
                // WebSocket 实例
                _this._ws = _this._protocols
                    ? new WebSocket(url, _this._protocols)
                    : new WebSocket(url);
                //使用二进制的数据类型连接。
                _this._ws.binaryType = _this._binaryType;
                _this._connectLock = false;
                // 添加回调方法
                _this._addListeners();
                console.log('connectionTimeout=', connectionTimeout)

                // 连接时长
                _this._connectTimeout = setTimeout(
                    function () {
                        // 如果在连接设置超时时间没有连上则会报错
                        return _this._handleTimeout();
                    }, connectionTimeout
                );
            });
    };
    // 如果在连接设置超时时间没有连上则会报错
    ReconnectingWebSocket.prototype._handleTimeout = function () {
        this._debug('timeout event');
        this._handleError(new ErrorEvent(Error('TIMEOUT'), this));
    };
    // 断开连接
    ReconnectingWebSocket.prototype._disconnect = function (code, reason) {
        if (code === void 0) { code = 1000; }
        this._clearTimeouts();
        if (!this._ws) {
            return;
        }
        this._removeListeners();
        try {
            this._ws.close(code, reason);
            this._handleClose(new CloseEvent(code, reason, this));
        }
        catch (error) {
            // ignore
        }
    };
    // 已连接了多久
    ReconnectingWebSocket.prototype._acceptOpen = function () {
        this._debug('accept open');
        this._retryCount = 0;
    };

    //触发 ReconnectingWebSocket 实例化 回调函数
    ReconnectingWebSocket.prototype._callEventListener = function (event, listener) {
        console.log('event=', event)
        // 如果是listener.handleEvent  执行回调方法 把 websocket 事件回调传进去
        if ('handleEvent' in listener) {
            // @ts-ignore
            listener.handleEvent(event);
        } else {
            // @ts-ignore
            //执行回调方法 把 websocket 事件回调传进去
            listener(event);
        }
    };

    // 删除WebSocket连接事件
    ReconnectingWebSocket.prototype._removeListeners = function () {
        if (!this._ws) {
            return;
        }
        this._debug('removeListeners');
        // 删除WebSocket连接事件
        this._ws.removeEventListener('open', this._handleOpen);
        this._ws.removeEventListener('close', this._handleClose);
        this._ws.removeEventListener('message', this._handleMessage);
        // @ts-ignore
        this._ws.removeEventListener('error', this._handleError);
    };

    // 添加WebSocket响应方法
    ReconnectingWebSocket.prototype._addListeners = function () {
        if (!this._ws) {
            return;
        }
        this._debug('addListeners');
        // console.log('this._ws.addEventListener=',this._ws.addEventListener)
        // websocket打开连接事件回调
        this._ws.addEventListener('open', this._handleOpen);
        // websocket关闭连接事件回调
        this._ws.addEventListener('close', this._handleClose);
        // websocket发送消息事件回调
        this._ws.addEventListener('message', this._handleMessage);
        //websocket连接错误事件回调 @ts-ignore 错误
        this._ws.addEventListener('error', this._handleError);
    };
    // 清除定时器
    ReconnectingWebSocket.prototype._clearTimeouts = function () {
        clearTimeout(this._connectTimeout);
        clearTimeout(this._uptimeTimeout);
    };
    return ReconnectingWebSocket;
}());

module.exports = ReconnectingWebSocket;

```

### ReconnectingWebSocket类

通过ReconnectingWebSocket类 实现_handleOpen，_handleMessage，_handleError，_handleClose 方法。

绑定到WebSocket实例化对象中，通过_listeners 队列收集外部绑定的Open，Message，Error，Close方法，通过订阅发布思想把WebSocket方法中的回调函数调用Open，Message，Error，Close等方法把WebSocket响应传给他们。

### 断开消息队列重发机制

该ReconnectingWebSocket有断开以后可以记录用户需要重发的消息。我们看看他是如何做的。

先看看send方法，send方法是 判断WebSocket是否已经连接上，如果连接上直接发送， 如果没连接上先插入到一个消息队列中

```
    /**
     * Enqueue specified data to be transmitted to the server over the WebSocket connection
     *  将要通过WebSocket连接传输到服务器的指定数据放入队列  
     */
    ReconnectingWebSocket.prototype.send = function (data) {
      // 判断WebSocket是否已经连接上，如果连接上直接发送，
        if (this._ws && this._ws.readyState === this.OPEN) {
            this._debug('send', data);
            this._ws.send(data);
        } else {
        // 如果没连接上先插入到一个消息队列中
            var _a = this._options.maxEnqueuedMessages,
                maxEnqueuedMessages = _a === void 0 ? DEFAULT.maxEnqueuedMessages : _a;
            if (this._messageQueue.length < maxEnqueuedMessages) {
                this._debug('enqueue', data);
                this._messageQueue.push(data);
            }
        }
    };
```



我们在来看看websocket连接上之后如何发送队列中的信息

```
      //WebSocket 连接 open 方法
        this._handleOpen = function (
            event //WebSocket 事件
        ) {
            _this._debug('open event');
            var _a = _this._options.minUptime,
                minUptime = _a === void 0 ? DEFAULT.minUptime : _a;
            // 清除 连接超时定时器
            clearTimeout(_this._connectTimeout);
            _this._uptimeTimeout = setTimeout(function () {
                return _this._acceptOpen();
            }, minUptime);
            _this._ws.binaryType = _this._binaryType;
            // send enqueued messages (messages sent before websocket open event)
            /*
             发送排队的消息(在websocket打开事件之前发送的消息)  
             这种情况就是在发生网络突然断开之后，用发送信息未能发送个服务器，然后先存到一个队列中
             等有网的时候连接上了再从队列中获取消息发送给服务器
             */
            _this._messageQueue.forEach(function (message) {
                // 给服务器发送信息
                return _this._ws.send(message);
            });
            // 清空队列
            _this._messageQueue = [];
            // 连接上回调方法
            if (_this.onopen) {
                _this.onopen(event);
            }
            //实例化对象连接注册事件
            _this._listeners.open.forEach(function (listener) {
                return _this._callEventListener(event, listener);
            });
        };
```

我们看到 这里就就是连接上之后发送刚才websocket断开连接时候的消息循环messageQueue一条条的发送出去。

```
 _this._messageQueue.forEach(function (message) {
                // 给服务器发送信息
                return _this._ws.send(message);
            });
```



