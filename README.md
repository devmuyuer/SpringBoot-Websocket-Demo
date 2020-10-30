# socket-demo WebSocket示例

工作中有这样一个需示，我们把项目中用到代码缓存到前端浏览器IndexedDB里面，当系统管理员在后台对代码进行变动操作时我们要更新前端缓存中的代码怎么做开始用想用版本方式来处理，但这样的话每次使用代码之前都需要调用获取版本API来判断版本是否有变化来是否更新本地代码，这样的话对服务器造成很大的压力。后来考虑http慢轮讯方式，最后了解到WebSocket这简直是神器，以后还可用来扩展项目中的即时聊天功能。

### WebSocket是什么
我们知道HTTP协议都是先由浏览器向服务器发送请求，服务器响应这个请求，再把数据发送给浏览器。 如果需要服务器发送消息给浏览器怎么办 WebSocket是HTML5新增的协议，让浏览器和服务器之间可以建立无限制的全双工通信，任何一方都可以主动发消息给对方。


# SpringBoot-Websocket-Demo
#### 项目介绍

- springboot 2.1.6.RELEASE

- spring-boot-starter-websocket

- 本项目主要为了测试springboot集成websocket实现向前端浏览器发送一个对象，发送消息操作手动触发。

*代码已上传到github 传送门* [https://github.com/devmuyuer/SpringBoot-Websocket-Demo](https://github.com/devmuyuer/SpringBoot-Websocket-Demo)

#### 代码说明
- 1.新建spingboot项目

- 2.加入WebSocket依赖
```java
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

- 3.WebSocketConfig 
开启WebSocket支持
```java
package com.example.socket;

/**
 * @author muyuer 182443947@qq.com
 * @version 1.0
 * @date 2019-07-22 18:16
 */

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.server.standard.ServerEndpointExporter;

/**
 * 开启WebSocket支持
 * @author muyuer 182443947@qq.com
 * @version 1.0
 * @date 2019-07-22 18:16
 */
@Configuration
public class WebSocketConfig {

    @Bean
    public ServerEndpointExporter serverEndpointExporter() {
        return new ServerEndpointExporter();
    }

}
```

- 4.WebSocketServer
WebSocket采用ws协议，这里的WebSocketServer类似于一个ws协议的Controller
```java
package com.example.socket;

import cn.hutool.json.JSONUtil;
import org.springframework.stereotype.Component;
import javax.websocket.*;
import javax.websocket.server.PathParam;
import javax.websocket.server.ServerEndpoint;
import java.io.IOException;
import java.util.concurrent.CopyOnWriteArraySet;
import cn.hutool.log.Log;
import cn.hutool.log.LogFactory;

/**
 * @author muyuer 182443947@qq.com
 * @version 1.0
 * @date 2019-07-22 18:17
 */
@ServerEndpoint("/web/socket/{sid}")
@Component
public class WebSocketServer {

    static Log log=LogFactory.get(WebSocketServer.class);
    /**
     * 静态变量，用来记录当前在线连接数。应该把它设计成线程安全的。
     */
    private static int onlineCount = 0;
    /**
     * concurrent包的线程安全Set，用来存放每个客户端对应的MyWebSocket对象。
     */
    private static CopyOnWriteArraySet<WebSocketServer> webSocketSet = new CopyOnWriteArraySet<WebSocketServer>();

    /**
     * 与某个客户端的连接会话，需要通过它来给客户端发送数据
     */
    private Session session;

    /**
     * 接收sid
      */
    private String sid="";
    /**
     * 连接建立成功调用的方法*/
    @OnOpen
    public void onOpen(Session session,@PathParam("sid") String sid) {
        this.session = session;
        //加入set中
        webSocketSet.add(this);
        //在线数加1
        addOnlineCount();
        log.info("有新窗口开始监听:"+sid+",当前在线人数为" + getOnlineCount());
        this.sid=sid;
        try {
            sendMessage("连接成功");
        } catch (IOException e) {
            log.error("websocket IO异常");
        }
    }

    /**
     * 连接关闭调用的方法
     */
    @OnClose
    public void onClose() {
        webSocketSet.remove(this);
        subOnlineCount();
        log.info("有一连接关闭！当前在线人数为" + getOnlineCount());
    }

    /**
     * 收到客户端消息后调用的方法
     *
     * @param message 客户端发送过来的消息*/
    @OnMessage
    public void onMessage(String message, Session session) {
        log.info("收到来自窗口"+sid+"的信息:"+message);
        for (WebSocketServer item : webSocketSet) {
            try {
                item.sendMessage(message);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    /**
     *
     * @param session
     * @param error
     */
    @OnError
    public void onError(Session session, Throwable error) {
        log.error("发生错误");
        error.printStackTrace();
    }
    /**
     * 实现服务器主动推送
     */
    public void sendMessage(String message) throws IOException {
        this.session.getBasicRemote().sendText(message);
    }
    /**
     * 实现服务器主动推送
     */
    public void sendMessage(SocketMessage message) throws IOException {
        this.session.getBasicRemote().sendText(JSONUtil.toJsonStr(message));
    }


    /**
     * 群发自定义消息
     * */
    public static void sendInfo(SocketMessage message,@PathParam("sid") String sid) throws IOException {
        log.info("推送消息到窗口"+sid+"，推送内容:"+message);
        for (WebSocketServer item : webSocketSet) {
            try {
                if(sid==null) {
                    item.sendMessage(message);
                }else if(item.sid.equals(sid)){
                    item.sendMessage(message);
                }
            } catch (IOException e) {
                continue;
            }
        }
    }

    public static synchronized int getOnlineCount() {
        return onlineCount;
    }

    public static synchronized void addOnlineCount() {
        WebSocketServer.onlineCount++;
    }

    public static synchronized void subOnlineCount() {
        WebSocketServer.onlineCount--;
    }
}
```

- 5.WebSocketController
测试用api 可在项目中调用 WebSocketServer.sendInfo(newMessage,cid);推送消息
```java
package com.example.socket;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.servlet.ModelAndView;

import java.io.IOException;
import java.util.Date;

/**
 * @author muyuer 182443947@qq.com
 * @version 1.0
 * @date 2019-07-22 18:19
 */
@Controller
@RequestMapping("/web/socket")
public class WebSocketController {

    /**
     * 页面请求
     * @param cid
     * @return
     */
    @GetMapping("/{cid}")
    public ModelAndView socket(@PathVariable String cid) {
        ModelAndView mav=new ModelAndView("/socket");
        mav.addObject("cid", cid);
        return mav;
    }

    /**
     * 推送数据接口
     * @param cid
     * @param message
     * @return
     */
    @ResponseBody
    @RequestMapping("/send/")
    public String pushToWeb(String cid,String message) {
        try {
            SocketMessage newMessage = new SocketMessage(message, new Date());
            WebSocketServer.sendInfo(newMessage,cid);
        } catch (IOException e) {
            e.printStackTrace();
            return cid+"#"+e.getMessage();
        }
        return cid;
    }
}
```

-6. 前端调用代码
```html
<!DOCTYPE HTML>
<html>
<head>
	<title>My WebSocket</title>
</head>

<body>
Welcome<br/>
<input id="text" type="text" /><button onclick="send()">Send</button>    <button onclick="closeWebSocket()">Close</button>
<div id="message">
</div>
</body>

<script type="text/javascript">
    var websocket = null;

    //判断当前浏览器是否支持WebSocket
    if('WebSocket' in window){
        websocket = new WebSocket("ws://localhost:8083/web/socket/20");
    }
    else{
        alert('Not support websocket')
    }

    //连接发生错误的回调方法
    websocket.onerror = function(){
        setMessageInnerHTML("error");
    };

    //连接成功建立的回调方法
    websocket.onopen = function(event){
        setMessageInnerHTML("open");
    }

    //接收到消息的回调方法
    websocket.onmessage = function(event){
        setMessageInnerHTML(event.data);
    }

    //连接关闭的回调方法
    websocket.onclose = function(){
        setMessageInnerHTML("close");
    }

    //监听窗口关闭事件，当窗口关闭时，主动去关闭websocket连接，防止连接还没断开就关闭窗口，server端会抛异常。
    window.onbeforeunload = function(){
        websocket.close();
    }

    //将消息显示在网页上
    function setMessageInnerHTML(innerHTML){
        document.getElementById('message').innerHTML += innerHTML + '<br/>';
    }

    //关闭连接
    function closeWebSocket(){
        websocket.close();
    }

    //发送消息
    function send(){
        var message = document.getElementById('text').value;
        websocket.send(message);
    }
</script>
</html>

```

####测试
- 1.f9运行项目，项目端口是8083可以配置文件中修改
- 2.首先在浏览器中打开地址 http://localhost:8083/ 建立连接
- 3.访问地址 http://localhost:8083/web/socket/send?cid=20&message=hello 推送消息 cid是客户端建立连接id 也就是html中"ws://localhost:8083/web/socket/20"的id 20

####参考资料
* [https://www.liaoxuefeng.com/wiki/1022910821149312/1103303693824096](https://www.liaoxuefeng.com/wiki/1022910821149312/1103303693824096)
* [https://blog.csdn.net/moshowgame/article/details/80275084](https://blog.csdn.net/moshowgame/article/details/80275084)

# SpringBoot-Websocket-Demo
