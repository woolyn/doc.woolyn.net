+++
date = "2017-06-15T11:30:43+08:00"
title = "编程实现数据上报"
toc = true
weight = 5

+++


## 原理

创建数据源后，就可以编写硬件程序了。传感器产生数据后，将数据按规则拼接起来，通过数据信道发送给云端即可。一次可以发送多个数据。数据拼接规则如下：
 
    数据源变量名1=数据值1&数据源变量名2=数据值2&…
    
以温湿度传感器设备为例，每隔一段时间（例如30分钟）,读取当前温度（26度）、湿度（52.18%），假设温度变量名为 `tmp`,湿度变量名为`hum` ，则将数据拼接为 `tmp=26&hum=52.18` 即可。

## 执行数据发送

要发送数据，将数据拼接好后，通过MQTT数据信道 Topic 发送到云端即可。数据信道 Topic 可在客户端的设备配置界面中找到。一般形式为 `/d/设备变量名/data` 。

以[ESP8266 Arduino](https://github.com/esp8266/Arduino) 通过 [pubsubclient](https://github.com/knolleary/pubsubclient) 库连接为例，实现一个物联网温湿度计的核心代码逻辑供参考：

```c
#include <ESP8266WiFi.h>
#include <PubSubClient.h>
#include <SimpleDHT.h>

#define DHT_PIN 2     // what digital pin we're connected to

// WiFi 配置，修改为自己的
const char* ssid = "YOUR_WIFI";
const char* password = "YOUR_WIFI_PASSWORD";

// 物林MQTT接入参数，参考设备配置修改
const char* mqtt_server = "mqtt.woolyn.net";
const char* data_topic = "/d/my_dht/data"; //数据信道
const char* mqtt_client_id = "my_dht";
const char* mqtt_username = "my_dht";
const char* mqtt_passwd = "YOUR_MQTT_PASSWORD";

WiFiClient espClient;
PubSubClient client(espClient);
SimpleDHT11 dht;

void setup() {
  Serial.begin(115200);
  setup_wifi();
  client.setServer(mqtt_server, 1883);
}

/**
 * 设置并连接WiFi
 */
void setup_wifi() {
  delay(10);
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
  }
}


/**
 * 查询温湿度数据并发送到云端
 */
void queryAndSend() {
  byte tmp, hum;

  //读取温、湿度数据
  if (dht.read(DHT_PIN, &tmp, &hum, NULL)) {
    Serial.print("Read DHT11 failed.");
    return;
  }

  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    // 尝试连接 MQTT 服务器
    if (client.connect(mqtt_client_id, mqtt_username, mqtt_passwd)) {
      Serial.println("connected");
      char mqttMsg[30];
      memset(mqttMsg, 0, 30);
      //按“变量名1=数据值1&变量名2=数据值2”的规则拼接温度、湿度数据
      snprintf(mqttMsg, 30, "tmp=%d&hum=%d", tmp, hum); 
      //发布到云端
      client.publish(data_topic, mqttMsg);
      Serial.println(mqttMsg);
    } else {
      Serial.print("failed, rc=");
      Serial.print(client.state());
      Serial.println(" try again in 5 seconds");
      // 连接失败则等5秒后重新连接
      delay(5000);
    }
  } 
}

void loop() {
  queryAndSend();
  delay(60000);//每分钟发送一次数据，不能频率过高
}

```

{{% notice info %}}
完整代码可在 [Github 示例](https://github.com/woolyn/examples/tree/master/my_dht) 中查看。
{{%% /notice}}