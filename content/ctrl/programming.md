+++
date = "2017-06-14T23:02:53+08:00"
title = "编程实现"
toc = true
weight = 2

+++

## 原理

创建控制器后，就可以在客户端进行操作了。
以拨动开关类型的控制器为例：点击客户端APP中的拨动开关，开关切换为“打开”状态，APP会要求云端服务器向该拨动开关所属设备的控制信道 Topic 发送控制命令，命令格式为：
 
    控制器变量名=命令值
    
更具体地说，如果这个拨动开关的变量名是 light, 这时云端服务器会向订阅了这个设备控制信道Topic的硬件发送 `light=on` 这样一个命令。
再次点击这个拨动开关，云端服务器便会向相应的硬件发送 `light=off` 命令。

当然，硬件设备要想接收到这个命令，就需要我们编写代码来实现了。

## 接收控制命令

要接收控制命令，按照 MQTT协议，需要先“订阅（subscribe）”控制信道 Topic。控制信道 Topic 可在客户端的设备配置界面中找到。一般形式为 `/d/设备变量名/ctrl` 。
一般情况下，代码中连接MQTT服务器后，可以指定一个回调函数。当控制信道中收到云端发来的命令时，就会调用该回调函数。回调函数中可得到发来的命令内容，我们根据命令内容让硬件执行相应的操作即可。

以[ESP8266 Arduino](https://github.com/esp8266/Arduino) 通过 [pubsubclient](https://github.com/knolleary/pubsubclient) 库连接为例，关键部分代码逻辑供参考：

```c

#include <ESP8266WiFi.h>
#include <PubSubClient.h>

// LED 连接在 GPIO2 上
#define LED_PIN 2

// WiFi 配置，修改为自己的
const char* ssid = "YOUR_WIFI";
const char* password = "WIFI_PASSWORD";

// 物林MQTT接入参数，参考设备配置修改
const char* mqtt_server = "mqtt.woolyn.net";
const char* ctrl_topic = "/d/my_light/ctrl";
const char* mqtt_client_id = "my_light";
const char* mqtt_username = "my_light";
const char* mqtt_passwd = "YOUR_MQTT_PASSWORD";

WiFiClient espClient;
PubSubClient client(espClient);

void setup() {
  pinMode(LED_PIN, OUTPUT);     // 设置端口为输出模式
  setup_wifi();
  client.setServer(mqtt_server, 1883);
  client.setCallback(callback);
}

/**
 * 设置并连接WiFi
 */
void setup_wifi() {
  delay(10);
  // 连接WiFi
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
  }
}

/** 
 * 收到控制消息后执行的回调函数
 * 
 * @param topic 收到的 Topic ，正常情况下是控制信道 Topic， 如 /d/my_light/ctrl
 * @param payload 收到的消息内容, 如 light=on 或 light=off
 * @param length 消息长度
 */
void callback(char* topic, byte* payload, unsigned int length) {
  char* msg = (char*)malloc(length + 1);
  memcpy(msg, payload, length);
  msg[length] = '\0';
  
  if(strncmp(msg, "light=", 6) == 0 && length >= 8 ) {
    if(msg[6] == 'o' && msg[7] == 'n') { // 判断收到的消息内容是否为 light=on
       digitalWrite(LED_PIN, HIGH);
    } else {
       digitalWrite(LED_PIN, LOW);
    }
  }
  
  free(msg);
}

/**
 * 自动重连
 */
void reconnect() {
  while (!client.connected()) {
    // 尝试连接 MQTT 服务器
    if (client.connect(mqtt_client_id, mqtt_username, mqtt_passwd)) {
      // 连接成功则订阅控制信道（Topic)
      client.subscribe(ctrl_topic);
    } else {
      // 连接失败则等5秒后重新连接
      delay(5000);
    }
  }
}

void loop() {
  if (!client.connected()) {
    reconnect();
  }
  client.loop();
}
```



