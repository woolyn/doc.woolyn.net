+++
date = "2017-06-13T23:53:15+08:00"
title = "点亮第一个物联网小灯"
toc = true
weight = 3

+++

感谢开源社区给我们提供了丰富的资源，开发制作一个能通过手机APP控制的小灯非常容易。下面以市面上常见的 ESP-01 WiFi 模块为例，制作我们的第一个物联网小灯。

{{% notice note %}}
本章提到的所有电子元件均可在淘宝网买到
{{% /notice %}}

## 新建自定义设备

APP 登录后，按下主界面右下角的浮动按钮，点击“创建自定义设备” 按钮：

![点击按钮](/images/start_1.png?width=300) 

进入创建自定义设备表单界面，填写如下内容：

| 表单项   | 填写内容 |
|----------|----------|
| 设备名称 | 我的小灯 |
| 变量名   | my_light |
| 地理位置 | 点击地图，选择您的设备所在的地理位置 |
| 简介     | 我的第一个物联网设备 |
| 设备照片 | 可先留空，以后拍照补上 |

![创建设备](/images/start_2.png?width=300)

{{% notice tip %}}
变量名短的好
{{% /notice %}}

填写完毕后，点击右上角的保存按钮，保存。这样，就在物林平台创建了一个新设备。接下来，我们给设备添加一个控制器。

## 添加控制器

由于我们的目的是通过物林APP控制“物联网小灯”这个硬件设备，因此需要给刚才APP上创建的新设备添加一个开关**控制器**，这样才能在APP上进行开关操作。

回到APP主界面，点击刚创建的“我的小灯”卡片右下角菜单，点击“配置”，进入设备配置界面后，切换到“控制器”选项卡：

![设备配置-控制器](/images/start_3.png?width=300)

点击右下角的浮动按钮，进入添加控制器界面，填写如下内容：

| 表单项   | 填写内容 |
|----------|----------|
| 名称     | 小灯开关 |
| 变量名   | light    |
| （控制类型） | 选择“拨动开关” |
| 允许他人操作 | 关 |

![设备配置-控制器](/images/start_4.png?width=300)

填写完毕后，点击右上角的保存按钮。回到APP主界面，点击“我的小灯”卡片空白处，进入设备主界面，切换到“控制”选项卡，可以看到刚创建的小灯开关。

这样，物林平台上的设备就已经完全配置好了。下面，我们开始搭建实际的硬件电路。

## 搭建电路原型

按下图所示，搭建好电路原型。

![ESP-01 物联网小灯电路](/images/esp-01-led.png)

{{% notice warning %}}
ESP-01 模块标准电压为 **3.3V**，严禁直接使用 USB 端口输出的5V电压，否则会烧毁该模块
{{% /notice %}}


{{% notice note %}}
ESP8266 连接WiFi时所需的瞬间电流较大，从USB转串口模块上取电时，往往无法得到足够的电流，因此上图搭配一个 470uF 电容器，以避免引起异常重启。
{{% /notice %}}



## 编写代码

物林提供了 MQTT 和 HTTP 两种接入方式，对于需要接收控制信号的设备来说，应该首选MQTT以确保实时性。在代码中，我们使用开源的 pubsubclient MQTT库连接物林MQTT服务器。

进入下一步之前，请先确保您已安装 Arduino ESP8266 开发环境，如果还没有安装，请先参阅[ESP8266 ARDUINO 开发环境搭建]({{< ref "util/arduino_esp8266_setup.md" >}})一章装好开发环境。

打开 Arduino 软件，新建一个项目，在`工具`->`开发板……`菜单中选择`Generic ESP8266 Module`，将当前项目的目标开发板切换为ESP8266；

点击 `项目`->`加载库…`->`管理库…`，在打开的库管理器窗口中，搜索“pubsubclient”，找到后点击`安装` 按钮，安装该 MQTT 客户端库。

完成后如图所示：

![Arduino 安装 pubsubclient 库](/images/start_5.png)


终于可以写代码了，将如下代码复制到 Arduino 编辑器中（我不会说你偷懒的）。修改代码中WiFi配置部分为自己的WiFi名称和密码，还有**控制信道** `ctrl_topic`、**MQTT接入账号用户名**`mqtt_username`、**MQTT接入密码**`mqtt_passwd`三个参数的值需要修改。关于这三个参数应该填写的值，请打开您的物林APP，在“我的小灯”设备配置界面的`接入参数`选项卡中查看。

如果您感兴趣，这里可以多花点时间阅读一下代码，了解下基本的逻辑（其实很简单 ^_^）



```
/*
 ESP8266 连接物林 MQTT 服务示例

 本代码演示 ESP8266 接入物林平台，根据接收到的控制信号开关LED的功能。

 基本流程为：
  - 连接WiFi
  - 连接MQTT服务器并订阅控制信道
  - 控制信道收到消息时，如果消息内容为 light=on ，则点亮 LED，否则关闭LED。

 更多信息请参考 http://doc.woolyn.net.cn 
*/

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
const char* mqtt_passwd = "QfTUfL0PQhfs0yY8PIZvE-oyXZXWe5vE";

WiFiClient espClient;
PubSubClient client(espClient);

void setup() {
  pinMode(LED_PIN, OUTPUT);     // 设置端口为输出模式
  Serial.begin(115200);
  setup_wifi();
  client.setServer(mqtt_server, 1883);
  client.setCallback(callback);
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
  Serial.print("Message arrived :");
  Serial.println(msg);

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
    Serial.print("Attempting MQTT connection...");
    // 尝试连接 MQTT 服务器
    if (client.connect(mqtt_client_id, mqtt_username, mqtt_passwd)) {
      Serial.println("connected");
      // 连接成功则订阅控制信道（Topic)
      client.subscribe(ctrl_topic);
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
  if (!client.connected()) {
    reconnect();
  }
  client.loop();
}
```



## 编译、上传

确保[电路原型图示]({{< ref "start/hello_world.md#搭建电路原型">}})中紫色的线（GPIO0）接地后，将USB转TTL模块插入电脑的USB端口。查看 Arduino 中 `工具`->`端口` 是否已经自动选择了有效的串口，如果没有，请先确保 USB转TTL 模块的驱动程序已安装好。如果是Windows系统，在设备管理器中查看`端口（COM和LPT）`下面出现的端口号（COM？），然后重新在 Arduino 的`端口`子菜单中选择相应的串口即可。

点击 Arduino 软件中的 `上传` 按钮（或按 `Ctrl+U`快捷键），就可以执行编译、上传了。上传完毕后，点击右上角的`串口监视器`按钮，正常情况下可以看到连接WiFi和MQTT的相关信息输出。

## 测试

试着点击切换物林APP中“我的小灯”中的“小灯开关”控制项，怎么样？面包板上的LED是不是可以被控制点亮、熄灭了？

大功告成！恭喜！

如果没有顺利实现，也没有关系，写程序总要有调试的，硬件也一样，仔细检查上面的步骤，耐心尝试，一定会成功的！

## 更上一层楼

如果您了解更多的电子基础知识，可以考虑修改电路原型，把LED换成继电器，就可以遥控家里的照明电灯或热水器了！

如果您有更多的创意，也可以开始实践了，比如做一个遥控逗狗神器……

{{% notice warning %}}
高压危险！控制高压电路时，一定要注意用电安全！
{{% /notice %}}

当然，物林平台不仅可以实现遥控，还可以接收处理设备传来的数据，具体的实现就交给聪明的您去进一步探索了。