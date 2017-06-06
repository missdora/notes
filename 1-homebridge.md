# 搭建自定义 HomeKit

### homebridge 是什么

 homebridge 的核心是 [HAP-NodeJS](https://github.com/KhaosT/HAP-NodeJS) 。HAP-NodeJS 逆向了 HomeKit 服务端协议（HAP: HomeKit Accessory Protocol） 从而满足了 geek 玩家的自定义智能家居需求

[homebridge](https://github.com/nfarina/homebridge) 封装了  [HAP-NodeJS](https://github.com/KhaosT/HAP-NodeJS) ，搭建了一个桥接平台，在制定了一套插件规范的基础上，实现了 HomeKit 的自定义插件化， homebridge 的插件全部通过 npm 安装



### Synology 上安装 homebridge 

1. 首先在 Synology 上安装 Docker 套件

2. 打开 Docker ，在注册表中搜索 homebridge，找到 marcoraddatz-homebridge ，下载安装，安装流程参照 https://registry.hub.docker.com/u/marcoraddatz/homebridge/

3. 在启动 homebridge 之前，需要在安装路径  `/docker/homebridge/`  下准备两个文件 `install.sh` 和 `config.json`

   1. `install.sh`用来安装服务自身依赖的模块和插件模块，示例如下：

      ```shell
      #!/bin/bash
      apt-get install libavahi-compat-libdnssd-dev
      npm install -g homebridge-synology
      npm install -g homebridge-broadlink-rm
      ```

      这里是我安装过的模块，由于每次重启 docker 的 homebridge 服务都会执行 `install.sh`脚本，所以在第一次安装完所有依赖之后，可以注释掉

   2. `config.json` 用来向 homebridge 注册自定义平台和配件，示例如下：

      ```json
      {
          "bridge": {
              "name": "Homebridge",
              "username": "{ MAC 地址 }",
              "port": 53699,//Homebridge启动的端口，默认即53699
              "pin": "111-22-333"//用来在 HomeKit 中配对 homebridge 服务，保持这个格式，随便写
          },
         "platforms": [//平台级插件
          ],
          "accessories": [//配件级插件
              {
                  "accessory": "Synology",
                  "name": "custom",
                  "ip": "{synology ip address}",
                  "mac": "{synology mac}",
                  "port": "{synology port}",
                  "secure": false,
                  "account": "{synology 用户名}",
                  "password": "{synology 密码}",
                  "version": 6,//{synology DSM version}
                  "disabled": ["switch"]
              }
          ]
      }
      ```

   ​

   3. 在示例中，我注册了 synology 插件。启动 homebridge ，可以在 docker 日志中查看服务是否正常启动。现在你可以打开 HomeKit 应用添加 homebridge 服务了，需要用到刚才配置项中的 pin 码。服务添加成功之后，正常情况下，HomeKit 首页中会出现 synology 温度信息的小方块。

      ​

### 通过 HomeKit 控制红外或者射频类家电

1. 我们需要一个支持学习红外及射频的万能遥控器。购买 Broadlink RM Pro，为什么买这个，因为目前市面上支持射频比较好的也就这个设备了，虽然它家的 App 烂的一逼，但是不要担心，接入 HomeKit 中，就不需要再打开这个 App 了。

2. 设备到手之后，先将 RM Pro 与 App 连接上。这个过程可能很顺利，也可能很不顺利，请多重复 N 次。尝试在 App 中添加红外遥控和射频遥控，测试遥控是否正常

3.  在 homebridge 中安装 `npm install -g homebridge-broadlink-rm` 其实已经出现在刚才的 `install.sh`文件中。

4. 设置 homebridge-broadlink-rm 配置项，在 `config.json`中的 `platforms`数组中添加配置项。很重要，这个配置表示我在 homebridge 中注册了 broadlink 的学习功能：

   ```json
   "platforms": [//平台级插件
       {
         	"platform":"BroadlinkRM",
           "name":"Broadlink RM",
           "accessories": [
                {
                   "name":"learn",
                   "type":"learn-code",
                   "disableAutomaticOff": true,
                   "scanFrequency": false
                }
           ]
       }
   ]
   ```

5. 重启 homebridge 服务，打开 homebridge 日志面板。如无意外，HomeKit 中会出现 learn 方块按钮。如果有意外，请查看日志中是否输出了成功检测到 broadlink 的日志信息。如果没有捕获到，请尝试重启 broadlink 设备和 homebridge 服务 N 次。这种情况一般会出现在意外将 broadlink 断电之后。正常情况下99%的概率可以检测到设备。

6. 按下 learn 按钮。如无意外，broadlink 的黄灯会亮起，然后，拿着遥控器使劲朝着 broadlink 狂按，同时不停刷新日志，如无意外，日志中会输出捕获到红外遥控或者射频遥控的 hex code。如果有意外，请重复尝试 6 这个步骤 N 次。

7. 重复 6 的步骤 N 次，直到记录下所有需要的 hex code。

8. 开始在 `config.json` 中添加红外和射频遥控配置项。完整配置项可参考 https://github.com/lprhodes/homebridge-broadlink-rm 。示例如下：

   ```json
    // 在 platforms --> BroadlinkRM --> accessories 数组中添加
    
   {
       "name":"Mubu On",
       "type":"switch-repeat",//推荐使用这个类型，因为遥控偶尔会没有触发成功，重复5次，遥控成功率可达 100%
       "sendCount": 5,
       "interval": 1,
       "data": "788f32...00000"// hex code
   }
   ```

9. 重启 homebridge 服务，如无意外， HomeKit 中会出现 `Mubu On`的方块按钮，现在请尝试按下，测试遥控是否正常触发。

10. 如有意外，请仔细查看日志排查问题。如无意外，请重复 8、9 步骤直到添加完所有遥控

11. 由于我们是将非智能家电添加到 HomeKit 中，所以推荐将「开家电」和「关家电」的操作分开作为两个配置项

### 设置 HomeKit 场景

1. 在 HomeKit 中切换到 「房间」面板，点击右上角 + 添加场景，在场景中添加需要执行的配件即可。



### 如无意外，你现在可以通过 Siri 控制家电了。





下期预告： 

* 将 homeassistant 平台接入 homebridge 服务
* 自己动手写 homebridge 插件

