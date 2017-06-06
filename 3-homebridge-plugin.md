# 自己动手写 homebridge 插件

前面说了 homebridge 定义了一套插件规范，所以只要根据按照规范来，你就可以制作满足自己需求的 homebridge 插件接入 HomeKit 中。

下面，我以接入我家的空气传感器为例，制作一个 homebridge 插件

### 365 空气传感器

我购买的是 https://item.taobao.com/item.htm?id=38549066385 这款 geek 风满满的传感器。其实这套传感器就是基于 arduino 开发。当初买这款，就是看中它的 hack 特性。

目前我已经基于卖家提供的上传工具，将数据上传到了第三方的物联网数据平台。我接下来要做的就是将平台上的空气数据呈现到 HomeKit 上。

### HomeKit 数据类型

https://github.com/KhaosT/HAP-NodeJS/blob/master/lib/gen/HomeKitTypes.js 这里可以查到目前所有支持的数据类型。每个 Service 是一个大类，在 HomKit 中表现为一个方块区域，里面又包括了多个 Characteristic 指标类型，可以点击方块详情看到。

这一次我们要使用到的 Service 类型有： **AirQualitySensor **, **HumiditySensor**, **TemperatureSensor**, **Switch**

### 插件规范

[简单的 demo](https://github.com/nfarina/homebridge/blob/6500912f54a70ff479e63e2b72760ab589fa558a/example-plugins/homebridge-lockitron/index.js) 这是官方给出的简单 accessory 级别插件 demo。

可以猜出，插件必须要做的几件事有：

1. 向 homebridge 注册这个配件

   ```
   Line 8: homebridge.registerAccessory("homebridge-lockitron", "Lockitron", LockitronAccessory);
   ```

2.  拥有一个 Service 实例

   ```
   Line 17:   this.service = new Service.LockMechanism(this.name);
   ```

3. 监听 Service 下 Characteristic  的 get 、set 事件， get 事件会在你打开 homekit 查看方块数据时触发，set 事件是在你按下方块后触发。所以并不是所有 Characteristic  都需要监听 set 事件。

   ```
   // start from Line 19

   this.service
       .getCharacteristic(Characteristic.LockCurrentState)
       .on('get', this.getState.bind(this));
     
   this.service
       .getCharacteristic(Characteristic.LockTargetState)
       .on('get', this.getState.bind(this))
       .on('set', this.setState.bind(this));
   ```

4. 必须有 getServices 方法，返回拥有的 Service 实例数组。

   ```
   //start from line 80

   LockitronAccessory.prototype.getServices = function() {
     return [this.service];
   }
   ```



看到这里，基本上你已经可以开始自己写一个 homebridge 插件了。

### 我的自定义插件

我在里面顺便实现了电视盒子的 off 控制

数据读取示例代码 index.js：

```javascript
var Service, Characteristic;
var request = require("request");

module.exports = function (homebridge) { 
  Service = homebridge.hap.Service;
  Characteristic = homebridge.hap.Characteristic;
  try{
    homebridge.registerAccessory("homebridge-custom-sensor", "CustomSensor", CustomSensor);
  } catch(e) {console.log(e)}     
};
function CustomSensor(log, config) { 
  this.log = console.log; 
  this.name = config.name;
  this.service = config.service; 
  this.apiUrl = config.apiUrl;
  this.forceRefreshDelay = config.forceRefreshDelay || 0;
  this.log(this.name, this.apiUrl);
  this.enableSet = false;
  this.rules = config.rules;//空气质量区间
  this.default = config.default;   
  this.value = config.value;
}
CustomSensor.prototype = {
  identify: function (callback) {
    this.log("Identify requested!");
    callback(null);
  },              
  getName: function (callback) {
    this.log("getName :", this.name); 
    var error = null;
    callback(error, this.name);
  },
  getServices: function () {
    var self = this;
    var getDispatch = function (callback, characteristic) {
      
      var actionName = "get" + characteristic.displayName.replace(/\s/g, '');

      if (typeof self.value !== 'undefined') {
        return callback(null, self.value);
      }
      
      request.get({ url: self.apiUrl }, function (err, response, body) {
        if (!err && response.statusCode == 200) {
          console.log("getDispatch:returnedvalue: ", JSON.parse(body));
          var value = JSON.parse(body).Data[0].value;

          if (self.service == 'AirQualitySensor'){//针对 lewei API 接口返回的数据做处理
            var _value = value;  
            value = self.default; 
            // 返回空气质量的枚举类型数据
            // Characteristic.AirQuality.UNKNOWN = 0;
		   // Characteristic.AirQuality.EXCELLENT = 1;
            // Characteristic.AirQuality.GOOD = 2;
            // Characteristic.AirQuality.FAIR = 3;
            // Characteristic.AirQuality.INFERIOR = 4;
            // Characteristic.AirQuality.POOR = 5;
            for (var i = 0; i < self.rules.length;i++) {
              value = Math.min(5, value + 1);
              if (self.rules[i+1] && _value >= self.rules[i] && _value <= self.rules[i + 1]) {
                break;               
              }                   
            }                   
          }
          callback(null,value);  
        } else {
          console.log("Error getting state: %s", actionName, err);
          callback(err);
        }
      }     
    }

    	
    var setDispatch = function (value, callback, characteristic) {
      if (self.enableSet == false) {
        callback() 
      } else {    
        // switch 类型会用到 set
        var actionName = "set" + characteristic.displayName.replace(/\s/g, '');
        if (value == 0 && self.service == 'Switch') {//如果按钮已经置灰，不做任何操作
          return callback();
        }
        
        request.get({ url: self.apiUrl, timeout: 1500}, function (err, response, body) {
          if (!err && response.statusCode == 200) {
            if (self.service == 'Switch') {
               setTimeout(function () {//2s 之后将按钮置灰
                newService.setCharacteristic(Characteristic.On, 0);
              }, 1000 * 2)  
            }
            callback(null); 
          } else {
            self.log("Error getting state: %s", actionName, err);        
            callback(err); 
          }
        }
      }       
    }

    var informationService = new Service.AccessoryInformation();
    informationService.setCharacteristic(Characteristic.Manufacturer, "HTTP Manufacturer")
      .setCharacteristic(Characteristic.Model, "HTTP Model")
      .setCharacteristic(Characteristic.SerialNumber, "HTTP Serial Number");
    var newService = null;

    switch (this.service) { 
      case "AccessoryInformation": newService = new Service.AccessoryInformation(this.name); break;
      case "AirQualitySensor": newService = new Service.AirQualitySensor(this.name); break;        
      case "HumiditySensor": newService = new Service.HumiditySensor(this.name); break;
      case "Switch": newService = new Service.Switch(this.name); break;         
      case "TemperatureSensor": newService = new Service.TemperatureSensor(this.name); break;
      default: newService = null
    }

    var counters = [];
    for (var characteristicIndex in newService.characteristics) {
      var characteristic = newService.characteristics[characteristicIndex];
      counters[characteristicIndex] = makeHelper(characteristic);

      //监听每个指标的 get set 事件
      characteristic.on('get', counters[characteristicIndex].getter)
      characteristic.on('set', counters[characteristicIndex].setter);
    }

    function makeHelper(characteristic) {
      return {
        getter: function (callback) { 
          getDispatch(callback, characteristic);
        },
        setter: function (value, callback) { 
          setDispatch(value, callback, characteristic) 
        }
      };
    }
    return [informationService, newService];
  }
}; 
```

homebridge 相关配置项示例代码：

```json
 "accessories": [
        {
            "accessory": "CustomSensor",
            "service": "AirQualitySensor",
            "name": "voc",
            "apiUrl": "http://www.lewei50.com/api/v1/sensor/gethistorydata...",
            "enableSet": false,
            "rules": [0, 80, 120, 200, 400],
            "default": 0
        },
        {
            "accessory": "CustomSensor",
            "service": "AirQualitySensor",
            "name": "PM2.5",
            "apiUrl": "http://www.lewei50.com/api/v1/sensor/gethistorydata...",
            "enableSet": false,
            "rules": [0, 12, 35, 55, 150],
            "default": 0
        },
        {
            "accessory": "CustomSensor",
            "service": "TemperatureSensor",
            "name": "temperature",
            "apiUrl": "http://www.lewei50.com/api/v1/sensor/gethistorydata...",
            "enableSet": false
        },
        {
            "accessory": "CustomSensor",
            "service": "HumiditySensor",
            "name": "humidity",
            "apiUrl": "http://www.lewei50.com/api/v1/sensor/gethistorydata...",
            "enableSet": false
        },
        {
          "accessory": "CustomSensor",
          "service": "Switch",
          "name": "tv off",
          //抓取了电视盒子手机遥控器的 HTTP 请求
          "apiUrl": "{mobile remote http request}",
          "enableSet": true,
          "value": 0
        }
    ]
```

将插件发布为 NPM 模块，然后安装到 homebridge 上，重启服务即可

