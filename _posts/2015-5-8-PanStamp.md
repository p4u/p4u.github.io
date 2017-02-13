---
layout: post
title: Playing with arduino PanStamp
published: true
---

[PanStamp](http://www.panstamp.com) is a low-cost, low-power and small Arduino device. IMO the most beautiful feature is that it includes a Wireless module which can operate at 400, 800 or 900 MHz in addition to an accelerometer for just 16€ (the whole device).
So recently I bought two of them to test and have some fun. The first impressions are very good, it accomplish all my spectatives. The Wireless protocol implemented (SWAP) is quite powerfull, minimalist and very well designed (of course it does not implement the WiFi-802.11 stack).

![PanStamp](/images/PanStamp.jpg)

I can imagine many possibilities and nice projects where using the PanStamps, but to start getting familiarized with it I've implemented the following dummy thing.

___

At my place there are free chickens (hens), but lately we are not able to find the eggs (they are hidden somewhere in our garden). So I made a small, proof-of-concept locator device using the PanStamps.
The idea is simple: attach a PanStamp based transmitter powered by battery to the hen. Once we do not see her in the garden (she will probably be with the eggs) we use the receiver to find it.

This is a picture of the transmitter device (PanStamp) attached to two AAA batteries (~3.5V / ~2000mAh), which according my calculations has an autonomy of 3-4 months.
The transmitter sends a Wireless packet of 48 bytes every second at very low power (the antenna is mutilated).

![Transmitter](/images/PanStamp_transmitter.jpg)

The receiver (right device in the picture) is listening for the transmitter packets, once if receives one it blinks a LED attached in the PanStamp board. So as more blinking time, more packets received thus closer to the transmitter.

![Transmitter and Receiver](/images/PanStamp_both.jpg)

I've not tested it (yet) with the hens, but I did it with my cat. I've also tested it by hidding the device somewhere in the garden and finding it, works!

![Test with my cat](/images/PanStamp_kiara.jpg)

### Receiver code

```C
#include "HardwareSerial.h" 
#define RFCHANNEL        0       // Let's use channel 0
#define SYNCWORD1        0xB5    // Synchronization word, high byte
#define SYNCWORD0        0x47    // Synchronization word, low byte
#define SOURCE_ADDR      4       // Sender address
#define DESTINATION_ADDR 5       // Receiver address
#define PACKET_SIZE 48

CCPACKET txPacket;  // packet object

//This function is called whenever a wireless packet is received
void rfPacketReceived(CCPACKET *packet) { 
  if (packet->length > PACKET_SIZE-1) { 
    digitalWrite(LED, HIGH);
    Serial.println("ALARM!");
  }
}

void setup()
{
  Serial.begin(9600);
  // Setup LED output pin
  pinMode(LED, OUTPUT);
  for (int i=0; i<6; i++) {
    digitalWrite(LED, !digitalRead(LED));
    delay(400);
  }
  // Configure PanStamp to work at 915MHz
  panstamp.init(CFREQ_915);
  panstamp.radio.setChannel(RFCHANNEL);
  panstamp.radio.setSyncWord(SYNCWORD1, SYNCWORD0);
  panstamp.radio.setDevAddress(SOURCE_ADDR);
  panstamp.setLowTxPower();
  panstamp.radio.setCCregs();
  panstamp.radio.disableAddressCheck();
  // Declare RF callback function
  panstamp.setPacketRxCallback(rfPacketReceived);
}

void loop()
{
  digitalWrite(LED, LOW);
  delay(1000);                          
}
```

### Transmitter code

```C
#define RFCHANNEL        0       // Let's use channel 0
#define SYNCWORD1        0xB5    // Synchronization word, high byte
#define SYNCWORD0        0x47    // Synchronization word, low byte
#define SOURCE_ADDR      4       // Sender address
#define DESTINATION_ADDR 5       // Receiver address
#define PACKET_SIZE 48

CCPACKET txPacket;  // packet object

void setup()
{
  // Setup LED output pin
  pinMode(LED, OUTPUT);
  for (int i=0; i<6; i++) {
    digitalWrite(LED, !digitalRead(LED));
    delay(400);
  }
  
  panstamp.init(CFREQ_915);
  panstamp.radio.setChannel(RFCHANNEL);
  panstamp.radio.setSyncWord(SYNCWORD1, SYNCWORD0);
  panstamp.radio.setDevAddress(SOURCE_ADDR);
  panstamp.setLowTxPower();
  panstamp.rxOff();
  panstamp.radio.setCCregs();
}

void loop()
{
  digitalWrite(LED, LOW);
  txPacket.length = PACKET_SIZE;  // Let's send a single data byte plus the destination address
  txPacket.data[0] = DESTINATION_ADDR;  // First data byte has to be the destination address
  txPacket.data[1] = 69;  // Add some data to the packet
  panstamp.radio.sendData(txPacket);  // Transmit packet
  panstamp.sleepSec(1);  // Sleep 1 second
}
```

