# What is WebSocket?

WebSocket 是双向的，一种全双工、有状态协议，客户端和服务端之间可以保持长时间的连接，直到被任何一方终止为止。
使得我们的应用程序和服务器之间可以实时、双向的进行数据传送。

相比于 HTTP， WebSocket 的优点：
- 服务端可以主动向客户端推送消息
- 发送消息的速度快
- 节省服务器请求资源

相比于 HTTP， WebSocket 的缺点：
- Debug 的时间成本相对较高。
- 需要维护 WebSocket 连接状态的稳定。

# WebSocket 连接的生命周期

![](https://github.com/wuzhenquan/blog/blob/main/images/Websocket%20+%20React%20Hooks%20%E5%AE%9E%E7%8E%B0%20IM%20Web%20%E5%BA%94%E7%94%A8/uQv3gSw.png?raw=true)
## WebSocket 连接的生命周期 -- 握手状态（opening handshake）

![](https://github.com/wuzhenquan/blog/blob/main/images/Websocket%20+%20React%20Hooks%20%E5%AE%9E%E7%8E%B0%20IM%20Web%20%E5%BA%94%E7%94%A8/SCR-20221231-sv0.png?raw=true) 

## WebSocket 连接的生命周期 -- 连接打开状态(Data transfer)

![](https://github.com/wuzhenquan/blog/blob/main/images/Websocket%20+%20React%20Hooks%20%E5%AE%9E%E7%8E%B0%20IM%20Web%20%E5%BA%94%E7%94%A8/rSUiWCh.png?raw=true)

## WebSocket 连接的生命周期 -- 关闭中和已关闭状态

关闭连接: 浏览器或者服务端，发送关闭帧，带上指定状态代码和关闭连接的原因。
```javascript
const ws = new WebSocket('wss://im.***.com/?uid=007');
ws.close(3001, '关闭原因: 用户退出会话页');
```

此时浏览器或者服务端开始关闭握手（closing handshake）
```javascript
ws.readyState === WebSocket.CLOSING; // => true
```

另一个端响应这个关闭帧表示连接断开（TCP connection terminated）
```javascript
ws.readyState === WebSocket.CLOSED; // => true
```

# 如何使用 WebSocket

[WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)  
[WebSocket interface](https://i.imgur.com/fTNYUks.png)

```javascript
const ws = new WebSocket('wss://***.com/?uid=007');
ws.onopen = () => console.log('websocket 连接成功');
ws.onclose = e => console.log(`websocket 断开: ${e.code} ${e.reason}`);
ws.onerror = e => console.error("WebSocket error:", e);;

// 发送消息
ws.send(JSON.stringify({ text: 'Hello Server' }));

// 接收消息
ws.onmessage = (e) => console.log(e.data); // => '{"text":"Hello Client"}'

// 关闭连接
ws.close();

// 获取连接状态
console.log(ws.readyState); // 0: WebSocket.CONNECTING, 1: WebSocket.OPEN, 2: WebSocket.CLOSING, 3: WebSocket.CLOSED

```

# WebSocket 的心跳机制

有时，服务端和浏览器之间的连接可能会中断，使服务端和浏览器都不知道连接的中断状态（例如意外断网时）。在这种情况下，实现心跳机制可以验证对方是否还在连接中。

心跳机制👇🏻

> A 端：发送 ping 消息到 B 端，如果收到了 B 端的 pong 消息表示 WebSocket 连接正常。

> B 端：收到 A 端的 ping 消息则发送 pong 消息来应答 A 端。

但是，对于 WebSocket 来说，**ping 消息只能从服务端发送到浏览器**，浏览器会自动使用 Pong 消息进行应答。

所以为了保证 Websocket 的稳定，也需要前端实现一套心跳机制，即 ping 消息从浏览器发送到服务端，然后检查服务端是否用 pong 去响应。

```javascript
// 前端实现心跳机制
const ws= new WebSocket('wss://im.******.com/?fuid=007');
const reconnect = () => {};
let heartbeatInterval = null;
let missedHeartbeats = 0;

ws.onpen = (){
  if (heartbeatInterval) return;
  missedHeartbeats = 0;
  heartbeatInterval = setInterval(() => {
    missedHeartbeats++;
    if (missedHeartbeats >= 60){ // 心跳检查 60 次都没反应, 断开并重连 websocket
      console.error(`😱连续检查 ${missedHeartbeats} 次检查都没心跳😱`);
      clearInterval(heartbeatInterval);
      heartbeatInterval = null;
      ws.close();
      reconnect();
    } else {
      // hearbeats 在 60 次以内，没间隔 1000ms 发送一次 ping
      ws.send('ping');
    }

  }, 1000)
}

// 检查服务端是否用 pong 响应了，如果是，清空 missedHeartbeats
ws.onmessge = (e) => {
  if (e.data === 'pong') missedHeartbeats = 0;
}
````

# WebSocket + React Hooks 的 Im 应用实践

### Features
- 消息的发送和接收（目前只有文本消息）
- 消息的发送状态（「发送中」、「发送成功」、「发送失败」）
- 消息的被读状态（「已读」、「未读」）
- 重新发送消息

### hooks
- 管理消息被读状态的 hook: useMsgStatus
- 管理消息发送状态的 hook: useMsgRead
- 管理消息重新发送的 hook: useMseResend
- 管理消息的 hook: useIM

### 消息的数据类型使用 Map 类型
- 主要优点一：Map 中的键是有序的
- 主要优点二：在频繁增删键值对的场景下表现更好

# 项目地址
repo: https://github.com/wuzhenquan/React-IM