---
layout: post
title: Tiny and cheap GPS for Arduino
published: true
---
The Global Positioning System (GPS) is something which has many utilities when playing with small computers.
In addition to the Geolocation it can also be used to synchronize the exact time and date with the satellites.
For Arduino this feature is quite important since the only way for having the correct time (without network) is by using an external RTC module.
So I decided to take a look on the current GPS chips and its compatibility with the Arduino board.

I want something cheap, small, with integrated antenna and low consumption. So looking at my favorite Chinese shops I found the 
[L80 module with MTK3339 chip](http://www.aliexpress.com/item/GPS-MODULE-L80-Integrated-with-Patch-Antenna-MT3339-Chip-with-Antenna-TTL-Replace-FGPMMOPA6H-PA6H-PA6C/32342876963.html).
![GPS device](/images/gps.jpg)
It seems to be some kind of replacement module but I don't care because in the description of the article there are the pin schematics.
![GPS pins](/images/gps_pins.jpg)
So it uses a UART/TTL interface (as many other GPS devices) to send the data following the [NMEA 0183](https://en.wikipedia.org/wiki/NMEA_0183) standard.
As it can be seen in the picture it is very small (not more than a stamp size) and consumes only 20mA in tracking mode.
Finally it costs only $10 and it can be found for even less, looks PERFECT!

Once I got it (it took around 2 weeks to land here) I weld the wires to the pins I need:

1. RX/TX for the serial communication
2. VCC/GND for the power supply (not more than 3.3V, very important!)
3. VCC backup (V_BCKP) with VCC or to an external battery (it won't work if this pin is not powered!)

![GPS wires](/images/gps_wires.jpeg)

I tested it using a standard UART/USB adaptor (again, remember to plug the GPS to 3.3V, not to 5V), the standard GPS daemon for Linux (gpsd -N -n /dev/ttyUSB0) and the gpsmon tool.

![GPS linux](/images/gps_working.png)

Ok, it works like a charm! So next step is to connect it to the Arduino. I'm using the digital pin 10 for receive (so TX of the GPS device) and pin 9 to send (pin RX of GPS device).

![GPS Arduino](/images/gps_arduino.jpeg)

To make it easy (NMEA protocol is quite a mess) I use the [tinyGPS++ library](http://arduiniana.org/libraries/tinygpsplus). This is the code I made to test it.

```C
#include <TinyGPS++.h>
#include <SoftwareSerial.h>
#define rxGPS 10
#define txGPS 9     

    long lat,lon; 
    int counter = 0;     
    SoftwareSerial gpsSerial(rxGPS,txGPS); 
    TinyGPSPlus gps;
     
    void setup(){
      Serial.begin(9600); // connect serial
      gpsSerial.begin(9600); // connect gps sensor
    }
     
    void loop(){
      while(gpsSerial.available()){ // check for gps data
       if(gps.encode(gpsSerial.read())){ // encode gps data
       if(counter > 50) {
        Serial.print("SATS: ");
        Serial.println(gps.satellites.value());
        Serial.print("LAT: ");
        Serial.println(gps.location.lat(), 6);
        Serial.print("LONG: ");
        Serial.println(gps.location.lng(), 6);
        Serial.print("ALT: ");
        Serial.println(gps.altitude.meters());
        Serial.print("SPEED: ");
        Serial.println(gps.speed.mps());
        
        Serial.print("Date: ");
        Serial.print(gps.date.day()); Serial.print("/");
        Serial.print(gps.date.month()); Serial.print("/");
        Serial.println(gps.date.year());
        
        Serial.print("Hour: ");
        Serial.print(gps.time.hour()); Serial.print(":");
        Serial.print(gps.time.minute()); Serial.print(":");
        Serial.println(gps.time.second());
       Serial.println("---------------------------");
        counter = 0;
       }
       else counter++;

       }
      }
    }
```

And it worked very good. The first time the GPS is turned on it takes some minutes to syncronize. But afterwards, when turned off and on, it is very fast (just a few seconds).

So the conclusion is that this GPS device accomplishes all my needs thus I'm willing to make some experiment mixing it with the [PanStamp](http://www.dabax.net/PanStamp/). I will keep you updated!

### Additional pin schematics

![Pins 1](/images/gps_pins1.jpg)
![Pins 2](/images/gps_pins2.png)

