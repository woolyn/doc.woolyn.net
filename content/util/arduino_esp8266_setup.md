+++
date = "2017-06-14T11:05:40+08:00"
title = "ESP8266 Arduino 开发环境搭建"
toc = true
weight = 5

+++

**Arduino** 是非常流行的开源硬件平台，提供了 Arduino 开发软件和相应的 Arduino 开发板，但普通的 Arduino 开发板没有网络连接功能。**ESP8266** 是 Espressif 公司研发的 WiFi 芯片，市面上常见的 **ESP-01**、**ESP-12** 等 WiFi 模块就是基于此芯片生产的，它在开源硬件爱好者群体中也非常受欢迎。幸运的是，Arduino 软件支持添加第三方硬件开发板和开发库，ESP8266 系列就是它支持的开发板之一。下文就将带您搭建好支持 ESP8266 系列模块的 Arduino 开发环境。

## 安装 Arduino 软件

如果您还没有安装过 Arduino 软件，请先[从 Arduino 官网上下载](https://www.arduino.cc/en/main/software) 安装。建议使用 Arduino 1.8.x 版本，兼容性较好。

## 为 Arduino 软件添加 ESP8266 系列开发板的支持

启动安装好的 Arduino 软件，打开`文件`->`首选项`，在 `附加开发板管理器网址` 后的输入框中填上 `http://arduino.esp8266.com/stable/package_esp8266com_index.json` ，如果之前有其他网址，用逗号隔开即可。然后确认退出对话框。

![首选项](/images/util_1.png)

点击 `工具`->`开发板` 在下拉菜单中安装 ESP8266 平台。这样，后面做 ESP 系列模块的开发时在这儿选择相应的开发板即可。

{{% notice tip %}}
ESP-01 模块请选择 `Generic ESP8266 Module`
{{% /notice %}}

## Arduino 添加第三方库

Arduino 平台有丰富的第三方库供我们使用，例如用于连接 MQTT 服务的 pubsubclient、用于读取 DHT 系列温湿度传感器的 SimpleDHT 库等。有了这些库，我们就可以在开发相关硬件时，直接安装引入而无需自行编写底层代码，节省很多开发时间。

以安装 pubsubclient 库为例，点击 `项目`->`加载库…`->`管理库…`，在打开的库管理器窗口中，搜索“pubsubclient”，找到后点击`安装` 按钮，Arduino 就开始为我们自动下载并安装库文件了。

完成后如图所示：

![Arduino 安装 pubsubclient 库](/images/start_5.png)`

