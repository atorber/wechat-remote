# WeChatRemote

## 概述

WeChatRemote是一个微信远程调用工具，使用MQTT连接本地或云端微信客户端实现远程操作

- 支持所有的Wechaty Puppet
- 解耦Wechaty客户端与业务应用服务端，使用熟悉语言实现MQTT Client即可调用
- 本地运行bot，远程接收消息和调用bot发消息

![原理图](https://cdn.nlark.com/yuque/0/2024/jpeg/250308/1705459538734-81f76086-02e8-4ac2-8fc5-868ce63afc13.jpeg)

## 快速入门

### 安装wechat-remote插件

```shell
npm i wechat-remote
```

### 启动机器人

首先启动一个wechaty客户端（目前仅支持nodejs），并使用mqtt-wechaty插件
机器人可以运行在本地或云服务器中，只需要可以访问外网，不需要配置外网IP

```typescript
import { WechatyBuilder } from 'wechaty'
import {
  QRCodeTerminal,
  MqttGateway,
  MqttGatewayConfig,
} from 'wechat-remote'

const bot = WechatyBuilder.build({
  name : 'ding-dong-bot',
})

const config: MqttGatewayConfig = {
  events: [
    'login',
    'logout',
    'reset',
    'ready',
    'dirty',
    'dong',
    'error',
    // 'heartbeat',
    'friendship',
    'message', 'post',
    'room-invite', 'room-join',
    'room-leave', 'room-topic',
    'scan',
  ],
  mqtt: {
    clientId: 'ding-dong-test01', // 替换成自己的clientId，建议不少于16个字符串
    host: 'broker.emqx.io',
    password: '',
    port: 1883,
    username: '',
  },
  options:{
    secrectKey: '',
    simple: false,
  },
  token: '',
}

bot.use(
  MqttGateway(config),
  QRCodeTerminal(),
)

bot.start()
  .catch(console.error)

```

### MQTTX调用

使用MQTT可视化工具调用接口，可以快速理解学习WeChatRemote使用

1. 访问 [http://www.emqx.io/online-mqtt-client](http://www.emqx.io/online-mqtt-client) ，配置并连接MQTT
2. 订阅如下主题，其中的“chatbot/”之后的“+”可以换成具体的clientId
   - 接收事件 thing/chatbot/+/event/post
   - 应用端下发指令 thing/chatbot/+/command/invoke
   - 接收响应 thing/chatbot/+/response/d2c/+
3. 查询机器人状态，向 thing/chatbot/ding-dong-test01/command/invoke 主题发布如下payload：

```json
{
    "reqId":"2a42cff1-bfb5-4d5d-b185-353c8c71a00b",
    "method":"command",
    "version":"1.0",
    "timestamp":1705384610041,
    "name":"wechatyLogonoff",
    "params":{}
}
```

![image.png](https://cdn.nlark.com/yuque/0/2024/png/250308/1705456827915-8e1ace3a-3fa2-414f-9104-e861468e3f48.png#averageHue=%23dff5e0&clientId=uadef19d5-ced0-4&from=paste&height=1040&id=u2b7b76d0&originHeight=1040&originWidth=1918&originalType=binary&ratio=1&rotation=0&showTitle=false&size=308523&status=done&style=none&taskId=uad897fa2-96d1-470e-bb8e-dc2655cd408&title=&width=1918)
4. 向指定好友发送消息，向 thing/chatbot/ding-dong-test01/command/invoke 主题发布如下payload：

```json
{
    "reqId":"2a42cff1-bfb5-4d5d-b185-353c8c71a00b",
    "method":"command",
    "version":"1.0",
    "timestamp":1705384610041,
    "name":"contactSay",
    "params":{
        "contacts":["atorber"],
        "messageType":"Text",
        "messagePayload":"hello"
    }
}
```

![image.png](https://cdn.nlark.com/yuque/0/2024/png/250308/1705457339087-f8a1d427-2596-45a7-b6ea-4b11f3880ebd.png#averageHue=%23fefefd&clientId=uadef19d5-ced0-4&from=paste&height=1039&id=ubd817504&originHeight=1039&originWidth=1917&originalType=binary&ratio=1&rotation=0&showTitle=false&size=304967&status=done&style=none&taskId=uea142729-c52c-42bb-a3d0-9fe0c0c9944&title=&width=1917)

> 下载安装MQTTX [https://mqttx.app/zh/downloads](https://mqttx.app/zh/downloads) 可以使用客户端更加稳定

### 程序调用

以下是几种常用语言的SimpleCode，后续考虑提供SDK，期待大家贡献各种语言的sdk

#### JavaScript

- 安装依赖

```shell
npm install mqtt
```

- 接收消息

```javascript
const mqtt = require('mqtt');

// MQTT 服务器设置
const MQTT_BROKER_URL = 'mqtt://your-mqtt-broker-url';
const TOPIC = 'thing/chatbot/xxxx/event/post';
const MQTT_PORT = 1883; // 替换为你的 MQTT 服务器端口
const USERNAME = 'your-username'; // 替换为你的用户名
const PASSWORD = 'your-password'; // 替换为你的密码

// 连接选项
const options = {
  port: MQTT_PORT,
  username: USERNAME,
  password: PASSWORD,
};

// 创建 MQTT 客户端
const client = mqtt.connect(MQTT_BROKER_URL, options);

client.on('connect', () => {
  console.log('Connected to MQTT Broker:', MQTT_BROKER_URL);
  // 订阅指定主题
  client.subscribe(TOPIC, (err) => {
    if (!err) {
      console.log(`Subscribed to topic: ${TOPIC}`);
    } else {
      console.error(`Failed to subscribe: ${err}`);
    }
  });
});

client.on('message', (topic, message) => {
  if (topic === TOPIC) {
    try {
      // 尝试解析消息为 JSON
      const jsonMessage = JSON.parse(message.toString());
      console.log('Received JSON Message:', jsonMessage);
    } catch (error) {
      console.error('Received non-JSON message:', message.toString());
    }
  }
});

client.on('error', (error) => {
  console.error('MQTT Client Error:', error);
});
```

- 调用API

```javascript
const mqtt = require('mqtt');

const MQTT_BROKER_URL = 'mqtt://your-mqtt-broker-url';
const COMMAND_TOPIC = 'thing/chatbot/xxxx/command/invoke';
const RESPONSE_TOPIC = 'thing/chatbot/xxxx/response/d2c/+';
const MQTT_PORT = 1883; // 或者你的 MQTT 服务器端口
const USERNAME = 'your-username'; // 或者你的用户名
const PASSWORD = 'your-password'; // 或者你的密码

const message = {
  reqId: "2a42cff1-bfb5-4d5d-b185-353c8c71a00b",
  method: "command",
  version: "1.0",
  timestamp: 1705384610041,
  name: "contactSay",
  params: {
    contacts: ["ledongmao", "choogoo"],
    messageType: "Text",
    messagePayload: "hello"
  }
};

const options = {
  port: MQTT_PORT,
  username: USERNAME,
  password: PASSWORD,
};

const client = mqtt.connect(MQTT_BROKER_URL, options);

client.on('connect', () => {
  console.log('Connected to MQTT Broker:', MQTT_BROKER_URL);

  // 订阅响应主题
  client.subscribe(RESPONSE_TOPIC, (err) => {
    if (!err) {
      console.log(`Subscribed to response topic: ${RESPONSE_TOPIC}`);
    } else {
      console.error(`Failed to subscribe: ${err}`);
    }
  });

  // 发布命令消息
  client.publish(COMMAND_TOPIC, JSON.stringify(message));
  console.log(`Message published to command topic: ${COMMAND_TOPIC}`);
});

client.on('message', (topic, message) => {
  console.log(`Received message from ${topic}: ${message.toString()}`);
});

client.on('error', (error) => {
  console.error('MQTT Client Error:', error);
});
```

#### TypeScript

- 安装依赖

```shell
npm install mqtt
npm install @types/node --save-dev
```

- 接收消息

```typescript
import * as mqtt from 'mqtt';

// MQTT 服务器设置
const MQTT_BROKER_URL = 'mqtt://your-mqtt-broker-url';
const TOPIC = 'thing/chatbot/xxxx/event/post';
const MQTT_PORT = 1883; // 替换为你的 MQTT 服务器端口
const USERNAME = 'your-username'; // 替换为你的用户名
const PASSWORD = 'your-password'; // 替换为你的密码

// 连接选项
const options: mqtt.IClientOptions = {
  port: MQTT_PORT,
  username: USERNAME,
  password: PASSWORD,
};

// 创建 MQTT 客户端
const client = mqtt.connect(MQTT_BROKER_URL, options);

client.on('connect', () => {
  console.log('Connected to MQTT Broker:', MQTT_BROKER_URL);
  // 订阅指定主题
  client.subscribe(TOPIC, (err) => {
    if (!err) {
      console.log(`Subscribed to topic: ${TOPIC}`);
    } else {
      console.error(`Failed to subscribe: ${err}`);
    }
  });
  // 订阅指定主题
  client.subscribe(TOPIC, (err) => {
    if (!err) {
      console.log(`Subscribed to topic: ${TOPIC}`);
    } else {
      console.error(`Failed to subscribe: ${err}`);
    }
  });
});

client.on('message', (topic, message) => {
  if (topic === TOPIC) {
    try {
      // 尝试解析消息为 JSON
      const jsonMessage = JSON.parse(message.toString());
      console.log('Received JSON Message:', jsonMessage);
    } catch (error) {
      console.error('Received non-JSON message:', message.toString());
    }
  }
});

client.on('error', (error) => {
  console.error('MQTT Client Error:', error);
});
```

- 调用API

```typescript
import * as mqtt from 'mqtt';
import type { IClientOptions, MqttClient } from 'mqtt';

const MQTT_BROKER_URL: string = 'mqtt://your-mqtt-broker-url';
const COMMAND_TOPIC: string = 'thing/chatbot/xxxx/command/invoke';
const RESPONSE_TOPIC: string = 'thing/chatbot/xxxx/response/d2c/+';
const MQTT_PORT: number = 1883; // 或者你的 MQTT 服务器端口
const USERNAME: string = 'your-username'; // 或者你的用户名
const PASSWORD: string = 'your-password'; // 或者你的密码

interface Message {
  reqId: string;
  method: string;
  version: string;
  timestamp: number;
  name: string;
  params: {
    contacts: string[];
    messageType: string;
    messagePayload: string;
  };
}

const message: Message = {
  reqId: "2a42cff1-bfb5-4d5d-b185-353c8c71a00b",
  method: "command",
  version: "1.0",
  timestamp: 1705384610041,
  name: "contactSay",
  params: {
    contacts: ["ledongmao", "choogoo"],
    messageType: "Text",
    messagePayload: "hello"
  }
};

const options: IClientOptions = {
  port: MQTT_PORT,
  username: USERNAME,
  password: PASSWORD,
};

const client: MqttClient = mqtt.connect(MQTT_BROKER_URL, options);

client.on('connect', () => {
  console.log('Connected to MQTT Broker:', MQTT_BROKER_URL);

  // 订阅响应主题
  client.subscribe(RESPONSE_TOPIC, (err) => {
    if (!err) {
      console.log(`Subscribed to response topic: ${RESPONSE_TOPIC}`);
    } else {
      console.error(`Failed to subscribe: ${err}`);
    }
  });

  // 发布命令消息
  client.publish(COMMAND_TOPIC, JSON.stringify(message));
  console.log(`Message published to command topic: ${COMMAND_TOPIC}`);
});

client.on('message', (topic: string, message: Buffer) => {
  console.log(`Received message from ${topic}: ${message.toString()}`);
});

client.on('error', (error: Error) => {
  console.error('MQTT Client Error:', error);
});
```

#### Python

- 安装依赖

```shell
pip install paho-mqtt
```

- 接收消息

```python
import json
import paho.mqtt.client as mqtt

# MQTT 服务器设置
MQTT_BROKER_URL = 'mqtt://your-mqtt-broker-url'
TOPIC = 'thing/chatbot/xxxx/event/post'
MQTT_PORT = 1883  # 替换为你的 MQTT 服务器端口
USERNAME = 'your-username'  # 替换为你的用户名
PASSWORD = 'your-password'  # 替换为你的密码

# 连接成功回调函数
def on_connect(client, userdata, flags, rc):
    print(f'Connected to MQTT Broker: {MQTT_BROKER_URL}')
    client.subscribe(TOPIC)
    print(f'Subscribed to topic: {TOPIC}')

# 接收消息回调函数
def on_message(client, userdata, msg):
    if msg.topic == TOPIC:
        try:
            # 尝试解析消息为 JSON
            json_message = json.loads(msg.payload.decode())
            print('Received JSON Message:', json_message)
        except ValueError:
            print('Received non-JSON message:', msg.payload.decode())

# MQTT 客户端设置
client = mqtt.Client()
client.username_pw_set(USERNAME, PASSWORD)
client.on_connect = on_connect
client.on_message = on_message

# 连接 MQTT Broker
client.connect(MQTT_BROKER_URL, MQTT_PORT, 60)

# 网络循环，以处理回调函数并保持脚本运行
client.loop_forever()
```

- 调用API

```python
import json
import paho.mqtt.client as mqtt

MQTT_BROKER_URL = 'mqtt://your-mqtt-broker-url'
COMMAND_TOPIC = 'thing/chatbot/xxxx/command/invoke'
RESPONSE_TOPIC = 'thing/chatbot/xxxx/response/d2c/+'
MQTT_PORT = 1883  # 或者你的 MQTT 服务器端口
USERNAME = 'your-username'  # 或者你的用户名
PASSWORD = 'your-password'  # 或者你的密码

message = {
    "reqId": "2a42cff1-bfb5-4d5d-b185-353c8c71a00b",
    "method": "command",
    "version": "1.0",
    "timestamp": 1705384610041,
    "name": "contactSay",
    "params": {
        "contacts": ["ledongmao", "choogoo"],
        "messageType": "Text",
        "messagePayload": "hello"
    }
}

def on_connect(client, userdata, flags, rc):
    print(f"Connected to MQTT Broker: {MQTT_BROKER_URL}")

    # 订阅响应主题
    client.subscribe(RESPONSE_TOPIC)
    print(f"Subscribed to response topic: {RESPONSE_TOPIC}")

    # 发布命令消息
    client.publish(COMMAND_TOPIC, json.dumps(message))
    print(f"Message published to command topic: {COMMAND_TOPIC}")

def on_message(client, userdata, msg):
    print(f"Received message from {msg.topic}: {msg.payload.decode()}")

def on_error(client, userdata, rc):
    print(f"MQTT Client Error: {rc}")

client = mqtt.Client()

client.username_pw_set(USERNAME, PASSWORD)
client.on_connect = on_connect
client.on_message = on_message
client.on_error = on_error

client.connect(MQTT_BROKER_URL.replace('mqtt://', ''), MQTT_PORT, 60)

client.loop_forever()
```

#### Go

- 安装依赖

```shell
go get github.com/eclipse/paho.mqtt.golang
```

- 接收消息

```go
package main

import (
    "encoding/json"
    "fmt"
    "os"
    "time"

    mqtt "github.com/eclipse/paho.mqtt.golang"
)

const (
    MQTT_BROKER_URL = "tcp://your-mqtt-broker-url:1883" // 注意更换为你的 MQTT 服务器地址和端口
    TOPIC           = "thing/chatbot/xxxx/event/post"   // 更换为你的主题
    USERNAME        = "your-username"                   // 更换为你的用户名
    PASSWORD        = "your-password"                   // 更换为你的密码
)

func onConnectHandler(client mqtt.Client) {
    if token := client.Subscribe(TOPIC, 0, onMessageReceivedHandler); token.Wait() && token.Error() != nil {
        fmt.Println("Subscribe error:", token.Error())
        os.Exit(1)
    }
    fmt.Println("Connected to MQTT Broker:", MQTT_BROKER_URL)
    fmt.Println("Subscribed to topic:", TOPIC)
}

func onMessageReceivedHandler(client mqtt.Client, msg mqtt.Message) {
    fmt.Printf("Received message on topic %s: %s\n", msg.Topic(), string(msg.Payload()))

    var jsonMessage map[string]interface{}
    if err := json.Unmarshal(msg.Payload(), &jsonMessage); err != nil {
        fmt.Println("Error parsing JSON message:", err)
    } else {
        fmt.Println("Received JSON Message:", jsonMessage)
    }
}

func main() {
    opts := mqtt.NewClientOptions().AddBroker(MQTT_BROKER_URL).SetUsername(USERNAME).SetPassword(PASSWORD)
    opts.SetDefaultPublishHandler(onMessageReceivedHandler)
    opts.OnConnect = onConnectHandler

    client := mqtt.NewClient(opts)
    if token := client.Connect(); token.Wait() && token.Error() != nil {
        fmt.Println("Connection error:", token.Error())
        os.Exit(1)
    }

    // Keep the main function running until interrupted
    select {}
}
```

- API调用

#### Java

- 安装依赖

```shell
<dependencies>
    <dependency>
        <groupId>org.eclipse.paho</groupId>
        <artifactId>org.eclipse.paho.client.mqttv3</artifactId>
        <version>1.2.5</version>
    </dependency>
</dependencies>
```

- 接收消息

```shell
import org.eclipse.paho.client.mqttv3.IMqttDeliveryToken;
import org.eclipse.paho.client.mqttv3.MqttCallback;
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttConnectOptions;
import org.eclipse.paho.client.mqttv3.MqttMessage;
import org.json.JSONObject;

public class MqttSubscribeSample {

    public static void main(String[] args) {
final String brokerUrl = "tcp://your-mqtt-broker-url:1883"; // 替换为你的MQTT Broker的URL
final String topic = "thing/chatbot/xxxx/event/post"; // 替换为你的主题
final String clientId = "JavaClient";
final String username = "your-username"; // 替换为你的用户名
final String password = "your-password"; // 替换为你的密码
    try {
        MqttClient sampleClient = new MqttClient(brokerUrl, clientId);
        MqttConnectOptions connOpts = new MqttConnectOptions();
        connOpts.setCleanSession(true);
        connOpts.setUserName(username);
        connOpts.setPassword(password.toCharArray());
        
        System.out.println("Connecting to broker: " + brokerUrl);
        sampleClient.connect(connOpts);
        System.out.println("Connected");
        
        sampleClient.setCallback(new MqttCallback() {
            @Override
            public void connectionLost(Throwable cause) {
                System.out.println("Connection lost!");
            }
            
            @Override
            public void messageArrived(String topic, MqttMessage message) throws Exception {
                System.out.println("Message arrived: " + message.toString());
                try {
                    JSONObject jsonMessage = new JSONObject(message.toString());
                    System.out.println("Received JSON Message: " + jsonMessage.toString(2)); // Indent with 2 spaces
                } catch (Exception e) {
                    System.out.println("Received non-JSON message: " + message.toString());
                }
            }
            
            @Override
            public void deliveryComplete(IMqttDeliveryToken token) {
                // Not used in this example
            }
        });
        
        System.out.println("Subscribing to topic: " + topic);
        sampleClient.subscribe(topic);
    } catch(Exception e) {
        System.out.println("Exception: " + e.getMessage());
    }
}
```

- API调用

```go
import org.eclipse.paho.client.mqttv3.IMqttDeliveryToken;
import org.eclipse.paho.client.mqttv3.MqttCallback;
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttConnectOptions;
import org.eclipse.paho.client.mqttv3.MqttException;
import org.eclipse.paho.client.mqttv3.MqttMessage;
import org.json.JSONObject;

public class MqttClientDemo {
    private static final String MQTT_BROKER_URL = "tcp://your-mqtt-broker-url:1883";
    private static final String COMMAND_TOPIC = "thing/chatbot/xxxx/command/invoke";
    private static final String RESPONSE_TOPIC = "thing/chatbot/xxxx/response/d2c/+";
    private static final String USERNAME = "your-username";
    private static final String PASSWORD = "your-password";

    public static void main(String[] args) {
        try {
            MqttClient client = new MqttClient(MQTT_BROKER_URL, MqttClient.generateClientId());
            MqttConnectOptions options = new MqttConnectOptions();
            options.setUserName(USERNAME);
            options.setPassword(PASSWORD.toCharArray());
            
            client.setCallback(new MqttCallback() {
                @Override
                public void connectionLost(Throwable cause) {
                    System.out.println("MQTT Client Error: " + cause.getMessage());
                }

                @Override
                public void messageArrived(String topic, MqttMessage message) {
                    System.out.println("Received message from " + topic + ": " + new String(message.getPayload()));
                }

                @Override
                public void deliveryComplete(IMqttDeliveryToken token) {
                }
            });

            client.connect(options);
            System.out.println("Connected to MQTT Broker: " + MQTT_BROKER_URL);
            
            client.subscribe(RESPONSE_TOPIC);
            System.out.println("Subscribed to response topic: " + RESPONSE_TOPIC);
            
            JSONObject message = new JSONObject();
            message.put("reqId", "2a42cff1-bfb5-4d5d-b185-353c8c71a00b");
            message.put("method", "command");
            message.put("version", "1.0");
            message.put("timestamp", 1705384610041L);
            message.put("name", "contactSay");
            JSONObject params = new JSONObject();
            params.put("contacts", new String[]{"ledongmao", "choogoo"});
            params.put("messageType", "Text");
            params.put("messagePayload", "hello");
            message.put("params", params);

            client.publish(COMMAND_TOPIC, new MqttMessage(message.toString().getBytes()));
            System.out.println("Message published to command topic: " + COMMAND_TOPIC);

        } catch (MqttException e) {
            e.printStackTrace();
        }
    }
}
```

> 其他语言示例欢迎在评论里粘贴代码或提需求，也可以交给GPTs [多语言代码转换大师](https://chat.openai.com/g/g-TgFzzIJjo-duo-yu-yan-dai-ma-zhuan-huan-da-shi)

## API说明

:::warning
目前仅实现了常用API的支持，其他API需求可以在评论中留言，当然API的范围不能超出Wechaty支持范围
:::
远程调用调用端向 thing/chatbot/${clinetId}/command/invoke发送调用指令（其中${clinetId}表示具体的clientId）
Wechaty客户端接收到指令后向thing/chatbot/${clinetId}/response/d2c/${reqId}主题推送响应消息
应用端订阅响应消息：

- 订阅指定请求消息：thing/chatbot/xxxx/response/d2c/1234
- 订阅所有请求消息：thing/chatbot/xxxx/response/d2c/+

### 查询机器人状态

- 请求

```json
{
    "reqId":"2a42cff1-bfb5-4d5d-b185-353c8c71a00b",
    "method":"command",
    "version":"1.0",
    "timestamp":1705384610041,
    "name":"wechatyLogonoff",
    "params":{}
}
```

- 响应

```json
{
  "reqId": "2a42cff1-bfb5-4d5d-b185-353c8c71a00b",
  "method": "response",
  "version": "1.0",
  "timestamp": 1705408082094,
  "name": "wechatyLogonoff",
  "code": 200,
  "message": "success",
  "params": {
    "logonoff": true
  }
}
```

### 获取机器人信息

- 请求

```json
{
    "reqId":"2a42cff1-bfb5-4d5d-b185-353c8c71a00b",
    "method":"command",
    "version":"1.0",
    "timestamp":1705384610041,
    "name":"wechatyUserSelf",
    "params":{}
}
```

- 响应

```json
{
  "reqId": "2a42cff1-bfb5-4d5d-b185-353c8c71a00b",
  "method": "response",
  "version": "1.0",
  "timestamp": 1705408574486,
  "name": "wechatyUserSelf",
  "code": 200,
  "message": "success",
  "params": {
    "_events": {},
    "_eventsCount": 0,
    "id": "@a4c7b5b71eb79b5f790f5d4c2270b0f3b787952a3a8ca08b228efe4832021299",
    "payload": {
      "alias": "",
      "avatar": "/cgi-bin/mmwebwx-bin/webwxgeticon?seq=1851844649&username=@a4c7b5b71eb79b5f790f5d4c2270b0f3b787952a3a8ca08b228efe4832021299&skey=@crypt_3f47a99d_1d59eeac1c79b47948fa86522b106b12",
      "friend": false,
      "gender": 2,
      "id": "@a4c7b5b71eb79b5f790f5d4c2270b0f3b787952a3a8ca08b228efe4832021299",
      "name": "瓦力",
      "phone": [],
      "star": false,
      "type": 1
    }
  }
}
```

### 获取好友列表

- 请求

```json
{
    "reqId":"2a42cff1-bfb5-4d5d-b185-353c8c71a00b",
    "method":"command",
    "version":"1.0",
    "timestamp":1705384610041,
    "name":"contactFindAll",
    "params":{
      "size":100
    }
}
```

- 响应

```json
{
    "reqId":"2a42cff1-bfb5-4d5d-b185-353c8c71a00b",
    "method":"response",
    "version":"1.0",
    "timestamp":1705384610041,
    "code":200,
    "message":"success",
    "name":"contactFindAll",
    "params":{
        "page":1,
        "size":100,
        "total":3500,
        "items":[
            {
                "_events":{

                },
                "_eventsCount":0,
                "id":"@a4c7b5b71eb79b5f790f5d4c2270b0f3b787952a3a8ca08b228efe4832021299",
                "payload":{
                    "alias":"",
                    "avatar":"/cgi-bin/mmwebwx-bin/webwxgeticon?seq=11550719&username=@a4c7b5b71eb79b5f790f5d4c2270b0f3b787952a3a8ca08b228efe4832021299&skey=@crypt_3f47a99d_1d59eeac1c79b47948fa86522b106b12",
                    "friend":false,
                    "gender":2,
                    "id":"@a4c7b5b71eb79b5f790f5d4c2270b0f3b787952a3a8ca08b228efe4832021299",
                    "name":"瓦力",
                    "phone":[

                    ],
                    "star":false,
                    "type":1
                }
            },
            {
                "_events":{

                },
                "_eventsCount":0,
                "id":"@a4c7b5b71eb79b5f790f5d4c2270b0f3b787952a3a8ca08b228efe4832021299",
                "payload":{
                    "alias":"",
                    "avatar":"/cgi-bin/mmwebwx-bin/webwxgeticon?seq=11550719&username=@a4c7b5b71eb79b5f790f5d4c2270b0f3b787952a3a8ca08b228efe4832021299&skey=@crypt_3f47a99d_1d59eeac1c79b47948fa86522b106b12",
                    "friend":false,
                    "gender":2,
                    "id":"@a4c7b5b71eb79b5f790f5d4c2270b0f3b787952a3a8ca08b228efe4832021299",
                    "name":"瓦力",
                    "phone":[

                    ],
                    "star":false,
                    "type":1
                }
            }
        ]
    }
}
```

### 查询好友详情

- 请求

alias、id、name至少填写一个

```json
{
    "reqId":"2a42cff1-bfb5-4d5d-b185-353c8c71a00b",
    "method":"command",
    "version":"1.0",
    "timestamp":1705384610041,
    "name":"contactFind",
    "params":{
        "alias":"",
        "id":"@a4c7b5b71eb79b5f790f5d4c2270b0f3b787952a3a8ca08b228efe4832021299",
        "name":"瓦力"
    }
}
```

- 响应

```json
{
    "reqId":"2a42cff1-bfb5-4d5d-b185-353c8c71a00b",
    "method":"response",
    "version":"1.0",
    "timestamp":1705384610041,
    "code":200,
    "message":"success",
    "name":"contactFind",
    "params":{
        "_events":{},
        "_eventsCount":0,
        "id":"@a4c7b5b71eb79b5f790f5d4c2270b0f3b787952a3a8ca08b228efe4832021299",
        "payload":{
            "alias":"",
            "avatar":"/cgi-bin/mmwebwx-bin/webwxgeticon?seq=11550719&username=@a4c7b5b71eb79b5f790f5d4c2270b0f3b787952a3a8ca08b228efe4832021299&skey=@crypt_3f47a99d_1d59eeac1c79b47948fa86522b106b12",
            "friend":false,
            "gender":2,
            "id":"@a4c7b5b71eb79b5f790f5d4c2270b0f3b787952a3a8ca08b228efe4832021299",
            "name":"瓦力",
            "phone":[],
            "star":false,
            "type":1
        }
    }
}
```

### 向指定好友发消息

> 目前仅支持发送文本消息，图片、文件、视频、语音等消息即将支持

- 请求

```json
{
    "reqId":"2a42cff1-bfb5-4d5d-b185-353c8c71a00b",
    "method":"command",
    "version":"1.0",
    "timestamp":1705384610041,
    "name":"contactSay",
    "params":{
        "contacts":[
            "ledongmao",
            "choogoo"
        ],
        "messageType":"Text",
        "messagePayload":"hello"
    }
}
```

- 响应

返回发送结果列表

```json
{
    "reqId":"2a42cff1-bfb5-4d5d-b185-353c8c71a00b",
    "method":"response",
    "version":"1.0",
    "timestamp":1705384610041,
    "code":200,
    "message":"success",
    "name":"contactSay",
    "params":{
      "success":[],
      "fail":[]
    }
}
```

### 获取群列表

- 请求

```json
{
    "reqId":"2a42cff1-bfb5-4d5d-b185-353c8c71a00b",
    "method":"command",
    "version":"1.0",
    "timestamp":1705384610041,
    "name":"roomFindAll",
    "params":{
      "size":100
    }
}
```

- 响应

```json
{
    "reqId":"2a42cff1-bfb5-4d5d-b185-353c8c71a00b",
    "method":"response",
    "version":"1.0",
    "timestamp":1705384610041,
    "code":200,
    "message":"success",
    "name":"roomFindAll",
    "params":{
        "page":1,
        "size":100,
        "count":3500,
        "items":[
            {
                "_events":{},
                "_eventsCount":0,
                "id":"19036636423@chatroom",
                "payload":{
                    "adminIdList":[],
                    "avatar":"",
                    "external":false,
                    "id":"19036636423@chatroom",
                    "ownerId":"",
                    "topic":"宁波金茂汇项目团队"
                }
            },
            {
                "_events":{},
                "_eventsCount":0,
                "id":"19036636423@chatroom",
                "payload":{
                    "adminIdList":[],
                    "avatar":"",
                    "external":false,
                    "id":"19036636423@chatroom",
                    "ownerId":"",
                    "topic":"宁波金茂汇项目团队"
                }
            }
        ]
    }
}
```

### 获取群详情

- 请求

```json
{
    "reqId":"2a42cff1-bfb5-4d5d-b185-353c8c71a00b",
    "method":"command",
    "version":"1.0",
    "timestamp":1705384610041,
    "name":"roomFind",
    "params":{
        "id":"@a4c7b5b71eb79b5f790f5d4c2270b0f3b787952a3a8ca08b228efe4832021299",
        "topic":"瓦力是群主"
    }
}
```

- 响应

```json
{
  "reqId": "2a42cff1-bfb5-4d5d-b185-353c8c71a00b",
  "method": "response",
  "version": "1.0",
  "timestamp": 1705418869966,
  "name": "roomFind",
  "code": 200,
  "message": "success",
  "params": {
    "_events": {},
    "_eventsCount": 0,
    "id": "5550027590@chatroom",
    "payload": {
      "adminIdList": [
        "wxid_m7kf9tq12"
      ],
      "avatar": "",
      "external": false,
      "id": "5550027590@chatroom",
      "memberIdList": [
        "wxid_m7kf9tq12",
        "tyutl",
        "wxid_0f57221"
      ],
      "ownerId": "wxid_m7kf9tq12",
      "topic": "瓦力是群主"
    }
  }
}
```

### 查询群成员列表

- 请求

id、topic至少填写一项

```json
{
    "reqId":"2a42cff1-bfb5-4d5d-b185-353c8c71a00b",
    "method":"command",
    "version":"1.0",
    "timestamp":1705384610041,
    "name":"roomMemberAllGet",
    "params":{
        "size":100,
        "id":"@a4c7b5b71eb79b5f790f5d4c2270b0f3b787952a3a8ca08b228efe4832021299",
        "topic":"瓦力是群主"
    }
}
```

- 响应

```json
{
  "reqId": "2a42cff1-bfb5-4d5d-b185-353c8c71a00b",
  "method": "response",
  "version": "1.0",
  "timestamp": 1705462410448,
  "name": "roomMemberAllGet",
  "code": 200,
  "message": "success",
  "params": {
    "page": 1,
    "size": 100,
    "total": 3,
    "items": [
      {
        "_events": {},
        "_eventsCount": 0,
        "id": "wxid_pnza7m7kf9tq12",
        "payload": {
          "alias": "",
          "friend": true,
          "id": "wxid_pnza7m7kf9tq12",
          "name": "瓦力",
          "phone": [],
          "type": 1
        }
      },
      {
        "_events": {},
        "_eventsCount": 0,
        "id": "tyutluyc",
        "payload": {
          "alias": "超哥",
          "friend": true,
          "id": "tyutluyc",
          "name": "luyuchao",
          "phone": [],
          "type": 1
        }
      },
      {
        "_events": {},
        "_eventsCount": 0,
        "id": "wxid_0o1t51l3f57221",
        "payload": {
          "alias": "",
          "avatar": "https://wx.qlogo.cn/mmhead/ver_1/xUlBgtTamVSCTw5KnGTxusBCT6eSQSgwDoRnp18ICZRp5JcWWFju1aLCzj5UurApCMDcLmfGOyxvDSwPL9yyoIB9zug49f5QjbI2HTk4ib1w/132",
          "friend": false,
          "gender": 0,
          "id": "wxid_0o1t51l3f57221",
          "name": "大师",
          "phone": [],
          "type": 1
        }
      }
    ]
  }
}
```

### 向指定群发消息

> 目前仅支持发送文本消息，图片、文件、视频、语音等消息即将支持

- 请求

```json
{
    "reqId":"2a42cff1-bfb5-4d5d-b185-353c8c71a00b",
    "method":"command",
    "version":"1.0",
    "timestamp":1705384610041,
    "name":"roomSay",
    "params":{
        "rooms":[
            "ledongmao@chat.room"
        ],
        "messageType":"Text",
        "messagePayload":"hello"
    }
}
```

- 响应

```json
{
  "reqId": "2a42cff1-bfb5-4d5d-b185-353c8c71a00b",
  "method": "response",
  "version": "1.0",
  "timestamp": 1705461448305,
  "name": "roomSay",
  "code": 200,
  "message": "success",
  "params": {
    "id": "ledongmao@chat.room"
  }
}
```

### @指定群好友

有瑕疵，调试中...

- 请求

```json
{
    "reqId":"2a42cff1-bfb5-4d5d-b185-353c8c71a00b",
    "method":"command",
    "version":"1.0",
    "timestamp":1705384610041,
    "name":"roomSayAt",
    "params":{
        "contacts":[
            "ledongmao",
            "choogoo"
        ],
        "room":"ledongmao@chat.room",
        "messageType":"Text",
        "messagePayload":"hello"
    }
}
```

- 响应

```json
{
  "reqId": "2a42cff1-bfb5-4d5d-b185-353c8c71a00b",
  "method": "response",
  "version": "1.0",
  "timestamp": 1705462056289,
  "name": "roomSayAt",
  "code": 200,
  "message": "success"
}
```

### 回复消息

- 请求

```json
{
    "reqId":"2a42cff1-bfb5-4d5d-b185-353c8c71a00b",
    "method":"command",
    "version":"1.0",
    "timestamp":1705384610041,
    "name":"messageSay",
    "params":{
      "id":"xxxxx",
      "messageType":"Text",
      "messagePayload":"hello"
    }
}
```

- 响应

```json
{
  "reqId": "2a42cff1-bfb5-4d5d-b185-353c8c71a00b",
  "method": "response",
  "version": "1.0",
  "timestamp": 1705463098535,
  "name": "messageSay",
  "code": 200,
  "message": "success",
  "params": {
    "id": "clrh8op130002yszc9col2coq"
  }
}
```

```json
{
  "reqId": "2a42cff1-bfb5-4d5d-b185-353c8c71a00b",
  "method": "response",
  "version": "1.0",
  "timestamp": 1705463049890,
  "name": "messageSay",
  "code": 200,
  "message": "消息不存在",
  "params": {}
}
```

## 事件消息推送

事件被推送到 thing/chatbot/${clinetId}/event/post主题（其中${clinetId}表示具体的clientId）
订阅thing/chatbot/xxxx/event/post接收事件数据

### 扫码 scan

```json
{
  "reqId": "2a42cff1-bfb5-4d5d-b185-353c8c71a00b",
  "method": "event",
  "version": "1.0",
  "timestamp": 1705384610041,
  "name": "scan",
  "params": {
    "qrcode": "https://login.weixin.qq.com/l/gaD8UC3bcA==",
    "status": 2
  }
}
```

### 登录 login

```json
{
  "reqId": "6e80b9ce-3585-46d2-982c-9be0087f21e2",
  "method": "event",
  "version": "1.0",
  "timestamp": 1705384714220,
  "name": "login",
  "params": {
    "_events": {},
    "_eventsCount": 0,
    "id": "@a4c7b5b71eb79b5f790f5d4c2270b0f3b787952a3a8ca08b228efe4832021299",
    "payload": {
      "alias": "",
      "avatar": "/cgi-bin/mmwebwx-bin/webwxgeticon?seq=11550719&username=@a4c7b5b71eb79b5f790f5d4c2270b0f3b787952a3a8ca08b228efe4832021299&skey=@crypt_3f47a99d_1d59eeac1c79b47948fa86522b106b12",
      "friend": false,
      "gender": 2,
      "id": "@a4c7b5b71eb79b5f790f5d4c2270b0f3b787952a3a8ca08b228efe4832021299",
      "name": "瓦力",
      "phone": [],
      "star": false,
      "type": 1
    }
  }
}
```

### 登出 logout

```json
{
  "reqId": "e0e8fb62-42a0-4b7c-9217-c3aadb9059dd",
  "method": "event",
  "version": "1.0",
  "timestamp": 1705384609786,
  "name": "logout",
  "params": [
    {
      "_events": {},
      "_eventsCount": 0,
      "id": "@85ee0016d30773031e259b3de554c8542340104d9b7a2c2646f73076b990130a",
      "payload": {
        "alias": "",
        "avatar": "/cgi-bin/mmwebwx-bin/webwxgeticon?seq=1946854219&username=@85ee0016d30773031e259b3de554c8542340104d9b7a2c2646f73076b990130a&skey=@crypt_3f47a99d_291be9b57007f9ec09ccd9cfda807c9c",
        "friend": false,
        "gender": 2,
        "id": "@85ee0016d30773031e259b3de554c8542340104d9b7a2c2646f73076b990130a",
        "name": "瓦力",
        "phone": [],
        "star": false,
        "type": 1
      }
    },
    "logout()"
  ]
}
```

### 准备就绪 ready

```json
{
  "reqId": "46d3bcb9-f21b-4236-91a3-88efb99ff9f9",
  "method": "event",
  "version": "1.0",
  "timestamp": 1705384530329,
  "name": "ready",
  "params": []
}
```

### 错误 error

```json
{
  "reqId": "52bd5dae-6cf5-44bb-a76c-982744ba585e",
  "method": "event",
  "version": "1.0",
  "timestamp": 1705384604626,
  "name": "error",
  "params": [
    {
      "code": 2,
      "details": "Error: 连续3次同步失败，5s后尝试重启\n    at C:\\Users\\Administrator\\Documents\\GitHub\\plugin-contrib\\node_modules\\wechat4u\\src\\wechat.js:100:19\n    at processTicksAndRejections (node:internal/process/task_queues:96:5)",
      "message": "连续3次同步失败，5s后尝试重启",
      "name": "Error",
      "stack": "Error: 连续3次同步失败，5s后尝试重启\n    at C:\\Users\\Administrator\\Documents\\GitHub\\plugin-contrib\\node_modules\\wechat4u\\src\\wechat.js:100:19\n    at processTicksAndRejections (node:internal/process/task_queues:96:5)"
    }
  ]
}
```

### 消息 message

```json
{
  "reqId": "2ee3362b-3142-44d5-b859-a15cb4177445",
  "method": "event",
  "version": "1.0",
  "timestamp": 1705384017113,
  "name": "message",
  "params": {
    "_events": {},
    "_eventsCount": 0,
    "id": "8768818756347691243",
    "payload": {
      "id": "8768818756347691243",
      "talkerId": "@d3cc07d80f9add1b4acd87e49817d061",
      "text": "h",
      "timestamp": 1705384016,
      "type": 7,
      "roomId": "@@e3e01b946e9f19978ad0718269b78b4f5dc853fd8eda5aa2dd5601ac57cdb3a5",
      "mentionIdList": []
    }
  }
}
```

```json
{
  "reqId": "b8ae2de2-19a3-4533-a5d5-9c2b7630a113",
  "method": "thing.event.post",
  "version": "1.0",
  "timestamp": 1705385708810,
  "events": {
    "onMessage": {
      "_id": "clrfyma84006vropd2i4ecmax",
      "data": {
        "_events": {},
        "_eventsCount": 0,
        "id": "clrfyma84006vropd2i4ecmax",
        "payload": {
          "id": "clrfyma84006vropd2i4ecmax",
          "listenerId": "",
          "roomId": "19036636423@chatroom",
          "talkerId": "wxid_zqbyd7b5kwy322",
          "text": "各位领导、同事\n      关于集团第四季度模拟钓鱼邮件演练，金茂商业有7名同事点击钓鱼链接，按总部要求加强各单位钓鱼邮件的识别、防御和处理能力。\n我们项目出现中招情况，故此，进行项目全员钓鱼邮件专题考试，希望大家在钓鱼邮件方面进一步加强安全意识意识和处置技能。\n以下为考试链接，请全员在1月19日前完成考试，我们将在群内公示考试人数。\nhttps://docs.chinajinmao.cn/form/ew/HOasNVyb/\n邀你填写「预防钓鱼邮件专题考试」",
          "timestamp": 1705385708356,
          "toId": "",
          "type": 7
        }
      },
      "room": {
        "_events": {},
        "_eventsCount": 0,
        "id": "19036636423@chatroom",
        "payload": {
          "adminIdList": [],
          "avatar": "",
          "external": false,
          "id": "19036636423@chatroom",
          "ownerId": "",
          "topic": "宁波金茂汇项目团队"
        }
      },
      "talker": {
        "_events": {},
        "_eventsCount": 0,
        "id": "wxid_zqbyd7b5kwy322",
        "payload": {
          "friend": true,
          "id": "wxid_zqbyd7b5kwy322",
          "name": "清昼",
          "phone": [],
          "type": 1
        }
      },
      "time": "56011-07-10 20:19:16",
      "timestamp": 1705385708356000
    }
  }
}
```

### 心跳 heartbeat

### 好友关系 friendship

### 邀请进群 room-invite

### 加入群 room-join

### 离开群 room-leave

### 群名称变更 room-topic

### 重载 dirty

## 附录

### Wechaty文档

[https://wechaty.js.org/docs/api/wechaty](https://wechaty.js.org/docs/api/wechaty)

### Wechaty事件定义

| Name | Type | Description |
| --- | --- | --- |
| error | string | When the bot get error, there will be a Wechaty error event fired. |
| login | string | After the bot login full successful, the event login will be emitted, with a Contact of current logined user. |
| logout | string | Logout will be emitted when bot detected log out, with a Contact of the current login user. |
| heartbeat | string | Get bot's heartbeat. |
| friendship | string | When someone sends you a friend request, there will be a Wechaty friendship event fired. |
| message | string | Emit when there's a new message. |
| ready | string | Emit when all data has load completed, in wechaty-puppet-padchat, it means it has sync Contact and Room completed |
| room-join | string | Emit when anyone join any room. |
| room-topic | string | Get topic event, emitted when someone change room topic. |
| room-leave | string | Emit when anyone leave the room. |
| room-invite | string | Emit when there is a room invitation, see more in [RoomInvitation](https://wechaty.js.org/docs/api/room-invitation)
 If someone leaves the room by themselves, wechat will not notice other people in the room, so the bot will never get the "leave" event. |
| scan | string | A scan event will be emitted when the bot needs to show you a QR Code for scanning. </br> It is recommend to install qrcode-terminal(run npm install qrcode-terminal) in order to show qrcode in the terminal. |

### 案例

[https://www.yuque.com/atorber/chatflow/ei64dvh6ba21yfes](https://www.yuque.com/atorber/chatflow/ei64dvh6ba21yfes)

[https://github.com/atorber/chatbot-iot-terminal](https://github.com/atorber/chatbot-iot-terminal)
