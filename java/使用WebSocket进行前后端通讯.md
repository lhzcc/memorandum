### WebSocket整合SpringMvc进行前后端通讯

------

1. 概念

   > 软件通信有七层结构，下三层结构偏向与数据通信，上三层更偏向于数据处理，中间的传输层则是连接上三层与下三层之间的桥梁，每一层都做不同的工作，上层协议依赖与下层协议。基于这个通信结构的概念。
   >
   > Socket 其实并不是一个协议，是应用层与 TCP/IP 协议族通信的中间软件抽象层，它是一组接口。当两台主机通信时，让 Socket 去组织数据，以符合指定的协议。TCP 连接则更依靠于底层的 IP 协议，IP 协议的连接则依赖于链路层等更低层次。
   >
   > WebSocket 则是一个典型的应用层协议。
   >
   > 总的来说：**Socket 是传输控制层协议，WebSocket 是应用层协议。**

   

2. 创建WebSocket服务端

   - 引入jar

     ```xml
     <!-- websocket  开始 -->
     <dependency>
         <groupId>javax.websocket</groupId>
         <artifactId>javax.websocket-api</artifactId>
         <version>1.1</version>
     </dependency>
     <dependency>
         <groupId>org.springframework</groupId>
         <artifactId>spring-websocket</artifactId>
         <version>${spring-version}</version>
     </dependency>
     <!-- websocket  结束 -->
     ```

   - 创建WebSocket服务类

     ```java
     /**
      * websocket连接URL地址和可被调用配置
      * @Date: 2019/12/6 11:44
      * @Version 1.0
      */
     @ServerEndpoint(value="/websocketserver/{userId}",configurator = SpringConfigurator.class)
     public class WebsocketServer {
     
         public WebsocketServer() {
             System.out.println("启动成功");
         }
     
         /**
          * 日志记录
          */
         private Logger logger = LoggerFactory.getLogger(WebsocketServer.class);
         /**
          * 静态变量，用来记录当前在线连接数。应该把它设计成线程安全的。
          */
         private static int onlineCount = 0;
     
         /**
          * 记录每个用户下多个终端的连接
          */
         private static Map<Long, Set<WebsocketServer>> userSocket = new HashMap<>();
     
         /**
          * 需要session来对用户发送数据, 获取连接特征userId
          */
         private Session session;
         private Long userId;
     
         /**
          * @Title: onOpen
          * @Description: websocekt连接建立时的操作
          * @param userId 用户id
          * @param session websocket连接的session属性
          */
         @OnOpen
         public void onOpen(@PathParam("userId") Long userId, Session session) {
             this.session = session;
             this.userId = userId;
             onlineCount++;
             //根据该用户当前是否已经在别的终端登录进行添加操作
             if (userSocket.containsKey(this.userId)) {
                 logger.debug("当前用户id:{}已有其他终端登录",this.userId);
                 //增加该用户set中的连接实例
                 userSocket.get(this.userId).add(this);
             } else {
                 logger.debug("当前用户id:{}第一个终端登录",this.userId);
                 Set<WebsocketServer> addUserSet = new HashSet<>();
                 addUserSet.add(this);
                 userSocket.put(this.userId, addUserSet);
             }
             logger.debug("用户{}登录的终端个数是为{}",userId,userSocket.get(this.userId).size());
             logger.debug("当前在线用户数为：{},所有终端个数为：{}",userSocket.size(),onlineCount);
         }
     
         /**
          * @Title: onClose
          * @Description: 连接关闭的操作
          */    @OnClose
         public void onClose() {
             //移除当前用户终端登录的websocket信息,如果该用户的所有终端都下线了，则删除该用户的记录
             if (userSocket.get(this.userId).size() == 0) {
                 userSocket.remove(this.userId);
             }else{
                 userSocket.get(this.userId).remove(this);
             }
             logger.debug("用户{}登录的终端个数是为{}",this.userId,userSocket.get(this.userId).size());
             logger.debug("当前在线用户数为：{},所有终端个数为：{}",userSocket.size(),onlineCount);
         }
     
         /**
          * @Title: onMessage
          * @Description: 收到消息后的操作
          * @param message 收到的消息
          * @param session 该连接的session属性
          */
         @OnMessage
         public void onMessage(String message, Session session) {
             logger.debug("收到来自用户id为：{}的消息：{}",this.userId,message);
             if(session ==null) {
                 logger.debug("session null");
             }
         }
     
         /**
          * @Title: onError
          * @Description: 连接发生错误时候的操作
          * @param session 该连接的session
          * @param error 发生的错误
          */
         @OnError
         public void onError(Session session, Throwable error) {
             logger.debug("用户id为：{}的连接发送错误",this.userId);
             error.printStackTrace();
         }
     
         /**
          * @Title: sendMessageToUser
          * @Description: 发送消息给用户下的所有终端
          * @param userId 用户id
          * @param message 发送的消息
          * @return 发送成功返回true，反则返回false
          */
         public Boolean sendMessageToUser(Long userId,String message){
             if (userSocket.containsKey(userId)) {
                 logger.debug(" 给用户id为：{}的所有终端发送消息：{}",userId,message);
                 for (WebsocketServer WS : userSocket.get(userId)) {
                     logger.debug("sessionId为:{}",WS.session.getId());
                     try {
                         WS.session.getBasicRemote().sendText(message);
                     } catch (IOException e) {
                         e.printStackTrace();
                         logger.debug(" 给用户id为：{}发送消息失败",userId);
                         return false;
                     }
                 }
                 return true;
             }
             logger.debug("发送错误：当前连接不包含id为：{}的用户",userId);
             return false;
         }
     }
     ```

   - 创建消息服务类，用于发送消息

     ```java
     /**
      * @Date: 2019/12/6 11:51
      * @Version 1.0
      */
     @Service("wSMessageService")
     public class WSMessageService {
         private Logger logger = LoggerFactory.getLogger(WSMessageService.class);
         /**
          * 声明websocket连接类
          */
         private WebsocketServer websocketServer = new WebsocketServer();
         /**
          * @Title: sendToAllTerminal
          * @Description: 调用websocket类给用户下的所有终端发送消息
          * @param userId 用户id
          * @param message 消息
          * @return 发送成功返回true，否则返回false
          */
         public Boolean sendToAllTerminal(Long userId,String message){
             logger.debug("向用户{}的消息：{}",userId,message);
             if(websocketServer.sendMessageToUser(userId, message)){
                 return true;
             } else {
                 return false;
             }
         }
     }
     ```

     

   - 创建Controller，用于接收发送消息的命令

     ```java
     /**
      * @Date: 2019/12/6 11:55
      * @Version 1.0
      */
     @Controller
     @RequestMapping("/message")
     public class MessageController {
     
         private Logger logger = LoggerFactory.getLogger(MessageController.class);
         /**
          * websocket服务层调用类
          */
         @Autowired
         private WSMessageService wSMessageService;
     
         /**
          * 请求入口
          * @param userId
          * @param message
          * @return
          */
         @RequestMapping(value="/TestWS",method= RequestMethod.GET)
         @ResponseBody
         public R TestWS(@RequestParam(value="userId") Long userId,
                         @RequestParam(value="message") String message){
             logger.debug("收到发送请求，向用户{}的消息：{}",userId,message);
     
             if(wSMessageService.sendToAllTerminal(userId, message)) {
                 return R.ok("发送成功");
             } else {
                 return R.error("发送失败");
             }
         }
     }
     ```

3. 测试，创建WebSocket客户端

   （1） 在前端使用websocket

   - 创建html

     ```html
     <!DOCTYPE HTML>
     <html>
        <head>
        <meta charset="utf-8">
        <title>websocket测试</title>
           <script type="text/javascript">
              function WebSocketTest() {
                 if ("WebSocket" in window) {
                    // 打开一个 web socket
                    var ws = new WebSocket("ws://localhost:8080/websocketserver/123");
                    ws.onopen = function() {
     				console.log("已连接成功√")
                       // Web Socket 已连接上，使用 send() 方法发送数据
                    };
                    ws.onmessage = function (e) { 
                       var received_msg = e.data;
     				  console.log(e.data)
                    };
                    ws.onclose = function() { 
                       // 关闭 websocket
                       console.log("连接已关闭..."); 
                    };
                 } else {
                    // 浏览器不支持 WebSocket
                    alert("您的浏览器不支持 WebSocket!");
                 }
              }
           </script>   
        </head>
        <body>
           <div id="sse">
              <a href="javascript:WebSocketTest()">运行 WebSocket</a>
           </div>
        </body>
     </html>
     ```

     - 点击运行websocket程序, 状态码为101表示连接成功

       ![微信图片_20200116155930](..\images\websocket\微信图片_20200116155930.png)

     - 后台控制台输出

       ![1579161822959](F:\文档记录\myproject\memorandum\images\websocket\1579161822959.png)

     - 测试发送消息

       ![1579161914814](F:\文档记录\myproject\memorandum\images\websocket\1579161914814.png)

     - 前端接收消息进行处理

       ![1579161967909](F:\文档记录\myproject\memorandum\images\websocket\1579161967909.png)

     （2）使用java创建WebSocket客户端

     - 引入jar

       ```xml
       <dependency>
           <groupId>org.java-websocket</groupId>
           <artifactId>Java-WebSocket</artifactId>
           <version>1.3.8</version>
       </dependency>
       ```

     - 创建WebSocket客户端类

       ```java
       /**
        * @Author: yaunlh
        * @Date: 2019/12/2 10:15
        * @Version 1.0
        */
       public class MyWebSocketClient extends WebSocketClient {
       
           public MyWebSocketClient(URI serverUri) {
               super(serverUri);
           }
           @Override
           public void onOpen(ServerHandshake serverHandshake) {
               System.out.println("------ MyWebSocket onOpen ------");
           }
           @Override
           public void onMessage(String s) {
               System.out.println("-------- 接收到服务端数据： " + s + "--------");
           }
           @Override
           public void onClose(int i, String s, boolean b) {
               System.out.println("------ MyWebSocket onClose ------");
           }
           @Override
           public void onError(Exception e) {
               System.out.println("------ MyWebSocket onError ------");
           }
       }
       ```

     - 测试

       ```java
       /**
        * @Author: yaunlh
        * @Date: 2019/12/2 10:20
        * @Version 1.0
        */
       public class TestWebSocket {
           public static void main(String[] args) throws URISyntaxException {
               MyWebSocketClient client 
                       = new MyWebSocketClient(
                   new URI("ws://localhost:8080/websocketserver/1234")
               );
               client.connect();
           }
       }
       ```

       客户端

       ![1579166666188](F:\文档记录\myproject\memorandum\images\websocket\1579166666188.png)

       服务端

       ![1579166566185](F:\文档记录\myproject\memorandum\images\websocket\1579166566185.png)

       给客户端发送消息

       ![1579166764545](F:\文档记录\myproject\memorandum\images\websocket\1579166764545.png)

       ![1579166784818](F:\文档记录\myproject\memorandum\images\websocket\1579166784818.png)

     4. 注意事项（坑）

        **Websocket在tomcat8以下未受到良好的支持，所以maven自带的tomcat访问会报404，只能使用外置的tomcat进行调试。**

------

end

整理by   yuanlh