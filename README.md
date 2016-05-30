# Arduino_W5100_Yeelink_Tempertrue
Arduino_W5100_Yeelink_Tempertrue
本文介绍了，使用Arduino W5100 DS18B20上传数据到yeelink上。


首先，从理论上来说，获取温度并上传并不算复杂，但是在尝试的过程中遇到各种坑，现在记录下来，希望后来者不要再次步入此坑。

## 使用正确的连线 ##
刚开始时，我从yeelink官网上看到一个demo，说的是LM35传感器获取温度，我从文章里看到连线图如下所示：
![](http://img.yeelink.net/yeelink/resource/images/example_basic/3_LM35/13.jpg)
照着这个图连下来，然后再串口调试中看到输出的数据完全是错误的，有负数，有200多，完全不是室内温度，后来看到半天我才知道，原来我自己使用的温度传感器是DS18B20，并不是LM35,然后根据下面这张图连线：
![](http://www.arduino.cn/data/attachment/forum/201208/01/2158082p12p1eplg16lvwj.jpg)

使用下面代码获取温度：

    #include <OneWire.h>
	OneWire  ds(10);  // 连接arduino10引脚
     
    void setup(void) {
      Serial.begin(9600);
    }
     
    void loop(void) {
      byte i;
      byte present = 0;
      byte type_s;
      byte data[12];
      byte addr[8];
      float celsius, fahrenheit;
       
      if ( !ds.search(addr)) {
    Serial.println("No more addresses.");
    Serial.println();
    ds.reset_search();
    delay(250);
    return;
      }
       
      Serial.print("ROM =");
      for( i = 0; i < 8; i++) {
    Serial.write(' ');
    Serial.print(addr[i], HEX);
      }
     
      if (OneWire::crc8(addr, 7) != addr[7]) {
      Serial.println("CRC is not valid!");
      return;
      }
      Serial.println();
      
      // the first ROM byte indicates which chip
      switch (addr[0]) {
    case 0x10:
      Serial.println("  Chip = DS18S20");  // or old DS1820
      type_s = 1;
      break;
    case 0x28:
      Serial.println("  Chip = DS18B20");
      type_s = 0;
      break;
    case 0x22:
      Serial.println("  Chip = DS1822");
      type_s = 0;
      break;
    default:
      Serial.println("Device is not a DS18x20 family device.");
      return;
      } 
     
      ds.reset();
      ds.select(addr);
      ds.write(0x44,1); // start conversion, with parasite power on at the end
       
      delay(1000); // maybe 750ms is enough, maybe not
      // we might do a ds.depower() here, but the reset will take care of it.
       
      present = ds.reset();
      ds.select(addr);
      ds.write(0xBE); // Read Scratchpad
     
      Serial.print("  Data = ");
      Serial.print(present,HEX);
      Serial.print(" ");
      for ( i = 0; i < 9; i++) {   // we need 9 bytes
    data[i] = ds.read();
    Serial.print(data[i], HEX);
    Serial.print(" ");
      }
      Serial.print(" CRC=");
      Serial.print(OneWire::crc8(data, 8), HEX);
      Serial.println();
     
      // convert the data to actual temperature
     
      unsigned int raw = (data[1] << 8) | data[0];
      if (type_s) {
    raw = raw << 3; // 9 bit resolution default
    if (data[7] == 0x10) {
      // count remain gives full 12 bit resolution
      raw = (raw & 0xFFF0) + 12 - data[6];
    }
      } else {
    byte cfg = (data[4] & 0x60);
    if (cfg == 0x00) raw = raw << 3;  // 9 bit resolution, 93.75 ms
    else if (cfg == 0x20) raw = raw << 2; // 10 bit res, 187.5 ms
    else if (cfg == 0x40) raw = raw << 1; // 11 bit res, 375 ms
    // default is 12 bit resolution, 750 ms conversion time
      }
      celsius = (float)raw / 16.0;
      fahrenheit = celsius * 1.8 + 32.0;
      Serial.print("  Temperature = ");
      Serial.print(celsius);
      Serial.print(" Celsius, ");   
      Serial.print(fahrenheit);
      Serial.println(" Fahrenheit");
    }


## 使用正确的上传数据代码 ##
什么是最坑的？最坑的不过是yeelink官网给我们的demo，无法运行取得正确的结果。
官网给的例子：[http://www.yeelink.net/developer/doc/48](http://www.yeelink.net/developer/doc/48 "通过网络查看室内温度变化")
代码如下：
[https://github.com/Yeelink/example_basic_3_LM35/blob/master/LM35.ino](https://github.com/Yeelink/example_basic_3_LM35/blob/master/LM35.ino)

经过测试，发现这段代码根本无法上传数据，并不知道是为什么，而且从yeelink官网上的blog专栏也可以看到，之前别人使用的都是使用EthernetClient 上传数据。[http://blog.yeelink.net/?p=94](http://blog.yeelink.net/?p=94)

所以，根据修改这篇文章里的`sendData(int data)` 发送数据到后台。

但是，这个函数只能发送int类型的变量，修改为float类型后发现并不能成功，原来在EthernetClient发送数据时，有一个数据长度校验位，Content-Length，把这个后面的值修改为16就可以发送了。

## 使用正确的引脚 ##
在网上看到好像W5100和10号引脚有冲突，由于我们在第一步中获取温度就是使用10号引脚，所以后来到上传数据时，并不能显示温度了。修改为5号引脚，问题解决。
然后把手指，放到传感器附近，可以看到数据的变化过程。

![](http://i.imgur.com/WpmMbFQ.jpg)

这里如果需要设置报警时发送微博，可以参考这篇文章：[http://www.yeelink.net/developer/doc/49](http://www.yeelink.net/developer/doc/49)

本文源代码在这里：
[https://github.com/hust201010701/Arduino_W5100_Yeelink_Temperature](https://github.com/hust201010701/Arduino_W5100_Yeelink_Temperature)

----------
好的，这篇文章就到这里，谢谢大家。
