### 集成步骤

#### 1、导入 SDK 
   引入 ```spider.jar```
#### 2、初始化
```
final SPIMConfig config = new SPIMConfig();//你可以在 config 里设置您自己部署好的 Spider Server 地址 
SPIMManager.sharedIMClient().configure(this.getApplicationContext());
SPIMManager.sharedIMClient().initWithAppkey(Appkey, config);

```
#### 3、设置消息和连接状态回调的代理
您需要设置并实现代理中的方法，来接收消息和连接状态变化的通知

```
SPIMManager.sharedIMClient().addMessageListener(new SPMessageListener() {
    @Override
    public void onReceived(SPMessage message, int left) {
        
    }
});

SPIMManager.sharedIMClient().addConnectionStatusListener(new SPConnectionStatusListener() {
    @Override
    public void onConnecting() {
  
    }

    @Override
    public void onConnected() {
 
    }

    @Override
    public void onDisconnect(int code, String describe) {
   
    }

    @Override
    public void onKickedOffline(String describe) {
        
    }
});
```

#### 4、连接
传入用户的 UserId 和他的签名（如何获取签名），你可以在成功或者失败的回调中处理相关的跳转逻辑。注：重连失败之后，如果 uid 和 sign 都争取，SDK 会帮您不断重试。

```
SPIMManager.sharedIMClient().connect(userId,token, new SPConnectionCompletion() {
    @Override
    public void success() {
    }
    @Override
    public void error(int code) {
    }
});
```
### 会话体系
目前支持一对一的单聊和，一对多的群聊模式

```
typedef NS_ENUM(NSUInteger, SPSessionType) {

    SPSessionType_Single = 1,

    SPSessionType_Group = 2,
};
``` 
### 自定义消息
#### 1、消息体系
* SPMessage 消息的基类 ，包含了消息的 Id 、内容、发送者、状态等等信息。
* SPMessageContent 消息内容的基类，所有的消息都继承这个基类比如 SPTextMessage等，子类需要继承该类并实现 SPMessageAnnotation 接口。
* SPMessageAnnotation 消息内容协议，所有消息必须实现该协议。
* messageProfileType 消息属性，定义了消息是否存储计数等信息。

```
/**
 消息存储类型
 /**
 * 空值，不表示任何意义。
 */
int NOT_SAVE_NOT_COUNT = 0x0;
/**
 * 消息需要被存储到消息历史记录。
 */
int SAVE_NOT_COUNT = 0x1;
/**
 * 消息需要被记入未读消息数。
 */
int SAVE_AND_COUNT = 0x02;
/**
 * 偷传消息, 不存储不计数。
 */
int TRANSPARENT = 0x03;
```
 SDK 内置了文本消息 SPTextMessage
 
```
@SPMessageAnnotation(messageType = "SP:TxtMsg", messageProfileType = SPMessageAnnotation.SAVE_AND_COUNT)
public class SPTextMessage extends SPMessageContent {
    public String text;
    public SPTextMessage(String text){
        this.text = text;
    }
    public SPTextMessage(){
        this.text = "";
    }

    @Override
    public String messageType() {
        return "SP:TxtMsg";
    }

    @Override
    public byte[] encode() throws JSONException {
        JSONObject object = new JSONObject();
        if (null != this.extra) {
            object.put("extra", this.extra);
        }
        if (null != this.digest) {
            object.put("digest", this.digest);
        }
        this.messageName = "SP:TxtMsg";
        this.text = this.text == null ? "" : this.text;
        object.put("text", this.text);
        return object.toString().getBytes();
    }

    @Override
    public void decode(byte[] payload) throws JSONException {
        try {
            String p = new String(payload);
            JSONObject object = new JSONObject(p);
            String text = object.getString("text");
            if(object.has("extra")){
                String extra = object.getString("extra");
                this.extra = extra;
            }
            if (object.has("digest")){
                String digest = object.getString("digest");
                this.digest = digest;
            }
            this.text = text;
        }
        catch (Exception exception) {
            XLog.i("SPTextMessage decode:" + exception);
        }
    }

    @Override
    public String digest() {
        return this.digest != null ? this.digest : this.text;
    }
}

```
#### 2、消息注册

所有的消息都有在初始化之后调用 SPIMManager 中的 ```registerMessageType``` 方法注册

```
/**
 * 注册自定义消息类型
 * @param messageClass 消息类
 */
public void registerMessageType(Class<? extends SPMessageContent> messageClass)
``` 
#### 3、消息发送

```
/**
 * 发送消息
 * @param message 发送的 message ,message 对象的 session 和 content 以及 profile
 * 不能为空，session 确定消息发送的目标，content 为消息的内容，profile 是消息属性
 * @param toSession 要发送的会话
 * @param callback 发送成功或失败回掉
 * @return SPMessage 更新 messageId
 */
public SPMessage sendMessage(SPMessage message,SPSession toSession, SPSendMessageResult callback)
```
#### 4、接收消息
当收到消息的时候会触发您设置的消息接收的监听的代理方法
```
 void onReceived(SPMessage message, int left);
```

### 插件机制
Spider-lib 支持插件的方式开发其他功能模块，方便您做功能模块的拆分，只需要在一个地方维护连接 Spider 的逻辑，其他功能模块不用关心和维护连接逻辑，模块只需要实现 SpPlugin 协议，当 Spider 连接或者收到消息的时候都会回调给扩展模块。

```
public interface SPPlugin {

    void onLogin(String appkey ,String userId);

    void onLogout();

    boolean onReceivedMessage(SPMessage message);

    void onConnectionStatusChanged(int status);

}

```

SPIMManager 里有注册和卸载插件的方法

```
/*
注册插件，需在 connect 之前注册

 @param SPPlugin
*/
public void registerPlugin(SPPlugin plugin);

/*
卸载插件

 @param SPPlugin
*/
 public void unRegisterPlugin(SPPlugin plugin);
```

#### 消息相关接口

```
/**
 * 获取某个会话下面的消息
 * @param session 会话
 * @param messageTypes 获取的消息类型
 * @param messageId 起始消息 ID
 * @param isForward 获取消息的方向
 * @param count 获取消息数量
 * @return 消息列表
 */
public List<SPMessage> getMessages(SPSession session, String[] messageTypes, long messageId, boolean isForward, int count)

public SPMessage getMessage(long messageId)

/**
 * 删除本地消息
 * @param msgIds 消息 ID 列表
 * @return true 成功， false 失败
 */
public boolean deleteMessages(long[] msgIds)

/**
 * 删除本地消息
 * @param session  会话
 * @return true 成功， false 失败
 */
public boolean deleteSessionMessages(SPSession session)

/**
 * 获取本地的会话列表
 * @param sessionTypes 会话类型的集合
 * @return 集合
 */
public List<SPSession> getSessionList(int[] sessionTypes)

/**
 * 获取会话
 * @param sessionId
 * @param sessionType
 * @return 会话 model
 */
public SPSession getSession(String sessionId, SPSessionType sessionType)

/*
 * 删除会话
 * @param session
 */
public boolean deleteSession(SPSession session)

/**
 * 清除某个 session 下面的未读状态
 * @param session 会话
 * @return true 成功，false 失败
 */
public boolean clearSessionUnreadStatus(SPSession session)

/**
 * 更新消息发送状态
 * @param messageId 消息 ID
 * @param status 发送状态
 * @return true 更新成功；false 更新失败
 */
public boolean updateMessageSentStatus(long messageId, SPMessageSentStatus status)

/**
 * 更新消息发送状态
 * @param session 会话
 * @param isTop 是否置顶
 * @return true 更新成功；false 更新失败
 */
public boolean updateSessionTopStatus(SPSession session,boolean isTop)

/**
 * 更新消息发送状态
 * @param sessionTypes 会话类型数组
 * @return int 未读数
 */
public int getUnreadCount (int[] sessionTypes)

/**
 * 更新消息发送状态
 * @param sessionTypes 会话类型数组
 * @return int 未读数
 */
public int getUnreadCount (SPSession session)
```
