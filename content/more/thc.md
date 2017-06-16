+++
date = "2017-06-16T13:51:14+08:00"
title = "物联网温湿度计"
toc = true
weight = 1

+++

想不想在手机上可以随时查看家里的温湿度？本文将带您实现自己的物联网温湿度计，非常简单，一起来吧！

## 所需元件

+ ESP-01 WiFi 模块一个
+ DHT-11 温湿度传感器一个
+ 10K 电阻一只
+ 470uF 电容器一个
+ 导线若干

当然，您还需要一个 USB 转 TTL 的下载模块一个，用于将电脑上编写的程序上传到 ESP-01 模块芯片中。

## 电路原型

按下图所示，搭建好电路原型：

![电路原型](/images/thc_2.png?width=800)

{{% notice note %}}
ESP8266 连接WiFi时所需的瞬间电流较大，如果上述电路原型无法正常下载代码，或者下载后运行异常（无响应、频繁重启），请更换为至少能输出500mA的3.3V独立电源供电。
{{% /notice%}}

## 创建设备，添加数据源

在物林客户端中，创建一个名为“温湿度计”的新设备。然后为其添加两个数据源，这里分别以 tmp、hum 变量名作为示例。

![配置新设备](/images/thc_1.png?width=300)

{{% notice tip %}}
创建新设备具体步骤请参考 [创建云端设备]({{< ref "arch/device.md" >}}) 一节。添加数据源请参考 [配置数据源]({{< ref "data/config.md" >}}) 一节。
{{% /notice %}}

## 编程实现

请确保使用的 Arduino 平台已经安装好 ESP 系列开发板的支持库，且已安装好 pubsubclient 和 SimpleDHT 库。如果还没有安装好，请先参考[ESP8266 Arduino 开发环境搭建]({{< ref "util/arduino_esp8266_setup" >}})一节完成安装。

如下代码供参考：

```c
#include <ESP8266WiFi.h>
#include <PubSubClient.h>
#include <SimpleDHT.h>

#define DHT_PIN 2    

// WiFi 配置，修改为自己的
const char* ssid = "CU_vxCa";
const char* password = "YOUR_WIFI_PASSWD";

// 物林MQTT接入参数，参考设备配置修改
const char* mqtt_server = "mqtt.woolyn.net";
const char* data_topic = "/d/my_thc/data";
const char* mqtt_client_id = "my_thc";
const char* mqtt_username = "my_thc";
const char* mqtt_passwd = "YOUR_MQTT_PASSWD";

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
  
  Serial.println();
  Serial.print("Connecting to ");
  Serial.println(ssid);

  // 连接WiFi
  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  //连接成功则通过串口打印 IP 地址
  Serial.println("");
  Serial.println("WiFi connected");
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());
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

## 测试，完成

编码完成后，将电路原型图中紫色的线接地后，接通电源进入下载模式，点击 Arduino 软件中的上传按钮，上传完毕后，断开紫色线，否则重启后会重新进入下载模式。正常情况下，上传完毕就会自动连接 WiFi 并向物林云端发送数据了，串口输出如下：

![串口输出](/images/thc_3.png)

物林客户端中也可以看到数据输出：


![物林](/images/thc_4.jpg?width=300)

![历史数据](/images/thc_5.png?width=300)