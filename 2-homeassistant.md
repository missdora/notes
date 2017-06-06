## HomeAssistant 接入 HomeKit

### HomeAssistant 是什么

 [HomeAssistant](http://home-assistant.io) 是国外一个成熟的，开源的智能家居平台。这个平台上面的插件数量比 homebridge 多得多。下面介绍如何通过 homebridge 将 HomeAssistant 接入 HomeKit

### Synology 上安装 HomeAssistant

1. 首先在 Synology 上安装 Docker 套件
2. 打开 Docker ，在注册表中搜索 homeassistant，找到 homeassistant/home-assistant，下载安装。
3. 启动 HomeAssistant , 如果服务正常启动，在 8123 端口可以访问到 HomeAssistant 服务。

### HomeAssistant 上安装 Synology 插件

1. 在 HomeAssistant 安装目录下有一个 `configuration.yml`文件，编辑

2. 参考配置项 https://home-assistant.io/components/sensor.synologydsm/ 

3. 我添加了四个指标：NAS 的机器温度、CPU 负载、内存占用、存储使用。示例如下：

   ```yaml
   homeassistant:
     # Name of the location where Home Assistant is running
     name: Home
     # Location required to calculate the time the sun rises and sets
     latitude: 30.2936
     longitude: 120.1614
     # Impacts weather/sunrise data (altitude above sea level in meters)
     elevation: 0
     # metric for Metric, imperial for Imperial
     unit_system: metric
     # Pick yours from here: http://en.wikipedia.org/wiki/List_of_tz_database_time_zones
     time_zone: Asia/Shanghai

     # 我添加的配置项
     customize:
       sensor.cpu_load_total:
         homebridge_sensor_type: humidity
       sensor.memory_usage_real:
         homebridge_sensor_type: humidity
       sensor.volume_used_volume_1:
         homebridge_sensor_type: humidity
             
   # Show links to resources in log and frontend
   introduction:

   # Enables the frontend
   frontend:

   # Enables configuration UI
   config:

   http:
     # Uncomment this to add a password (recommended!)
     # 我添加的配置项
     api_password: 123
     # Uncomment this if you are using SSL or running in Docker etc
     # base_url: example.duckdns.org:8123

   # Checks for available updates
   # Note: This component will send some information about your system to
   # the developers to assist with development of Home Assistant.
   # For more information, please see:
   # https://home-assistant.io/blog/2016/10/25/explaining-the-updater/
   updater:

   # Discover some devices automatically
   discovery:

   # Allows you to issue voice commands from the frontend in enabled browsers
   conversation:

   # Enables support for tracking state changes over time.
   history:

   # View all events in a logbook
   logbook:

   # Track the sun
   sun:

   # Weather Prediction
   sensor:
    - platform: yr
    # 我添加的配置项
    - platform: synologydsm
      host: 10.0.0.3
      port: 5000
      username: {synology username}
      password: {synology password}
      monitored_conditions:
        - cpu_total_load
        - memory_real_usage
        - network_up
        - volume_percentage_used

   # Text to speech
   tts:
     platform: google

   group: !include groups.yaml
   automation: !include automations.yaml
   ```

4. 要注意下面这段配置，这是针对 homebridge 做的自定义配置。后续解释

   ```yaml
     # 我添加的配置项
     customize:
       sensor.cpu_load_total:
         homebridge_sensor_type: humidity
       sensor.memory_usage_real:
         homebridge_sensor_type: humidity
       sensor.volume_used_volume_1:
         homebridge_sensor_type: humidity
             
   ```

5. 重启 HomeAssistant 服务，访问 8123 端口服务，如无意外，可以在 HomeAssistant 服务首页看到四个 synology 指标



### 接入 homebridge

1. 在 homebridge 中安装 [homebridge-homeassistant](https://github.com/home-assistant/homebridge-homeassistant) 插件模块，参考上一篇

2. 安装完成之后，在 `config.json`中添加平台插件配置项：

   ```json
   // 在 platforms 数组中添加
   {
       "platform": "HomeAssistant",
       "name": "HomeAssistant",
       "host": "http://127.0.0.1:8123",
       "password": 123,//刚才在 configuration.yml 中设置的密码
       "supported_types": ["binary_sensor", "climate", "cover", "device_tracker", "fan", "group", "input_boolean", "light", "lock", "media_player", "scene", "sensor", "switch"],
       "logging": true
   }
   ```

3. 现在来解释一下为什么在 `configuration.yml ` 中添加了 **customize** 字段，在 homebridge-homeassistant 的文档中有介绍过目前 HomeKit 支持的 Service 类型：

   1. **Binary Sensor** - door, leak, moisture, motion, smoke, and window state
   2. **Climate** - current temperature, target temperature, heat/cool mode
   3. **Cover** - exposed as a garage door or window covering (see notes)
   4. **Device Tracker** - home/not home status appears as an occupancy sensor
   5. **Fan** - on/off/speed
   6. **Group** - on/off
   7. **Input boolean** - on/off
   8. **Lights** - on/off/brightness
   9. **Lock** - lock/unlock lock
   10. **Media Players** - exposed as an on/off switch
   11. **Scenes** - exposed as an on/off switch
   12. **Sensors** - carbon dioxide (CO2), humidity, light, temperature sensors
   13. **Switches** - on/off

   HomeAssistant 将数据传给 homebridge 时，是以 sensors 类数据传过去的。 机器温度默认是 TemperatureSensor 类型，不需要修改。而由于 **CPU 负载、内存占用、存储使用** 这三个指标都不能匹配任何一个 Sensors 系列中的 Service，我们需要手动指定这三个指标的 Service 类型。

   这三个指标都是百分比格式，所以比较适合的就是 Sensors 系列中的 **湿度 HumiditySensor** 了。指定的方法很简单，就是在 homebridge_sensor_type 中带上 humidity 关键词，HomeAssistant 就会自动识别为 HumiditySensor 类型数据。

4. 现在重启 homebridge 服务。如无意外，可以在 HomeKit 中看到这四个服务器指标了。如有意外，请检查配置项是否写错。



