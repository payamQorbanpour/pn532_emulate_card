برنامه ارتباط P2P با ماژول PN532

نیازمندی‌ها: سخت افزار

ماژول ان اف سی PN532 V3
ماژول ESP32 (برای استفاده از سریال سخت افزاری دوم آن برای اتصال به ماژول NFC)

نیازمندی‌ها: نرم افزاری

کتابخانه NFC module dev hsu
نحوه عملکرد NFC و پروتکل‌هایی نظیر NDEF و SNEP که برای ارتباط P2P استفاده می‌شود در این مستند شرح داده شده است.

نیازمندی‌ها: نرم‌افزار تست موبایل

برای اندروید: NFC Tools
برای آی‌او‌اس: NFC taginfo

ارتباطات: 

برای ارتباط ماژول ESP32 با ماژول PN532 از پروتکل HSU استفاده شده است. هر چند که در کتابخانه استفاده شده امکان اتصال با پروتکل‌های SPI و I2C نیز وجود دارد. 

نرم‌افزار:

برای بارگذاری نرم‌افزار طبق این آموزش ابتدا کتابخانه‌های مورد نیاز نصب می‌شوند.
کد مثالی که برای ارسال url به موبایل استفاده شده به صورت زیر است
/**
* This example demonstrates pushing a NDEF message from Arduino + NFC Shield to Android 4.0+ device
*
* This demo does not support UNO, because UNO board has only one HardwareSerial.
* Do not try to use SoftwareSerial to control PN532, it won't work.
* SotfwareSerial is not fast and stable enough.
*
* This demo only supports the Arduino board which has at least 2 Serial,
* Like Leonard(1 USB serial and 1 Hardware serial), Mega ect.
*
* Make sure your PN532 board is in HSU(High Speed Uart) mode.
*
* This demo is tested with Leonard.
*/


#include <PN5321.h>
#include <NFCLinkLayer.h>
#include <SNEP.h>

#include <NdefMessage.h>

/** Use hardware serial to control PN532 */
// PN532                    Arduino
// VCC            -->          5V
// GND            -->         GND
// RXD            -->      Serial1-TX
// TXD            -->      Serail1-RX
/** Serial1 can be  */
PN532 nfc(Serial1);
NFCLinkLayer linkLayer(&nfc);
SNEP snep(&linkLayer);


// NDEF messages
#define MAX_PKT_HEADER_SIZE  50
#define MAX_PKT_PAYLOAD_SIZE 100
uint8_t txNDEFMessage[MAX_PKT_HEADER_SIZE + MAX_PKT_PAYLOAD_SIZE];
uint8_t *txNDEFMessagePtr;
uint8_t txLen;

void setup(void) {
 Serial.begin(115200);
 while(!Serial);
 Serial.println(F("----------------- nfc ndef push url --------------------"));


 txNDEFMessagePtr = &txNDEFMessage[MAX_PKT_HEADER_SIZE];
 NdefMessage message = NdefMessage();
 message.addUriRecord("http://elechouse.com");
 txLen = message.getEncodedSize();
 if (txLen <= MAX_PKT_PAYLOAD_SIZE) {
   message.encode(txNDEFMessagePtr);
 }
 else {
   Serial.println("Tx Buffer is too small.");
   while (1) {
   }
 }

 nfc.initializeReader();

 uint32_t versiondata = nfc.getFirmwareVersion();
 if (! versiondata) {
   Serial.print("Didn't find PN53x board");
   while (1); // halt
 }
 // Got ok data, print it out!
 Serial.print("Found chip PN5");
 Serial.println((versiondata>>24) & 0xFF, HEX);
 Serial.print("Firmware ver. ");
 Serial.print((versiondata>>16) & 0xFF, DEC);
 Serial.print('.');
 Serial.println((versiondata>>8) & 0xFF, DEC);
 Serial.print("Supports ");
 Serial.println(versiondata & 0xFF, HEX);

 nfc.SAMConfig();
}

void loop(void)
{
 Serial.println();
 Serial.println(F("---------------- LOOP ----------------------"));
 Serial.println();

 uint32_t txResult = GEN_ERROR;

 if (IS_ERROR(nfc.configurePeerAsTarget(SNEP_SERVER))) {
   extern uint8_t pn532_packetbuffer[];

   Serial.println(F("\nSNEP Sever:Blocking wait response."));
   nfc.readspicommand(PN532_TGINITASTARGET, (PN532_CMD_RESPONSE *)pn532_packetbuffer, 0);
 }

 txResult = snep.pushPayload(txNDEFMessagePtr, txLen);
 Serial.print(F("Result: 0x"));
 Serial.println(txResult, HEX);    
 if(txResult == 0x00000001){
   delay(3000);
 }
}

در خط ۳۰ این کد NFC_Module_DEV-HSU/example/nfc_ndef_push_url/nfc_ndef_push_url.ino:30 سریال ۲ 
PN532 nfc(Serial2);


جایگزین سریال ۱ شده است.
PN532 nfc(Serial1);

سوالات:
هنگام خواندن ماژول توسط موبایل خطای زیر در سریال مانیتور مشاهده می‌شود:
-------------Mobile-----------------
Read Ack
 0 0 FF 0 FF 0Read response:
SNEP Sever:Blocking wait response.
Read response:  0 0 FF 28 D8 D5 8D 16 25 D4 0 1 FE F BB BA A6 C9 89 0 0 0 0 0 32 46 66 6D 1 1 12 2 2 7 FF 3 2 0 3 4 1 64 7 1 3 2F 0
Opening SNEP Client Link.
Read Ack
 0 0 FF 0 FF 0Read response:  0 0 FF 28 D8 D5 8D 16 25 D4 0 1 FE F BB BA A6 C9 89 0 0 0 0 0 32 46 66 6D 1 1 12 2 2 7 FF 3 2 0 3 4 1 64 7 1 3 2F 0
Read Ack
 0 0 FF 0 FF 0Read response:  0 0 FF 3 FD D5 87 29 7B 0
Connection Failed.
Result: 0x80000004

	برای آزمایش ماژول، نرم‌افزار زیر روی ESP8266 بارگذاری شد تا دو ماژول PN532 بتوانند برای هم داده بفرستند.
/**
  @file    nfc_p2p_initiator.ino
  @author  www.elechouse.com
  @brief   example of Peer to Peer communication for NFC_MODULE.
  
    For this demo, initiator waiting for target proximity. As soon as the
  target sensed, the initiator exchange data with the target.
    By this demo, initiator sends "Hi, this message comes from NFC INITIATOR."
 
    NOTE: this library only support MAX 50 bytes data packet, that is the tx_len must
  less than 50..
  
  @section  HISTORY
  
  V1.0 initial version
  
    Copyright (c) 2012 www.elechouse.com  All right reserved.
*/

/** include library */
#include "nfc.h"

/** define a nfc class */
NFC_Module nfc;
/** define RX and TX buffers, and length variable */
u8 tx_buf[50]="Hi, this message comes from NFC INITIATOR.";
u8 tx_len;
u8 rx_buf[50];
u8 rx_len;

void setup(void)
{
  Serial.begin(115200);
  /** nfc initial */
  nfc.begin();
  Serial.println("P2P Initiator Demo BY ELECHOSUE!");
  
  uint32_t versiondata = nfc.get_version();
  if (! versiondata) {
    Serial.println("Didn't find PN53x board");
    while (1); // halt
  }
  
  // Got ok data, print it out!
  Serial.print("Found chip PN5"); Serial.println((versiondata>>24) & 0xFF, HEX); 
  Serial.print("Firmware ver. "); Serial.print((versiondata>>16) & 0xFF, DEC); 
  Serial.print('.'); Serial.println((versiondata>>8) & 0xFF, DEC);
  
  /** Set normal mode, and disable SAM */
  nfc.SAMConfiguration();
}

void loop(void)
{
  
  /** device is configured as Initiator */
  if(nfc.P2PInitiatorInit()){
    Serial.println("Target is sensed.");
    
    /**
      send data with a length parameter and receive some data,
      tx_buf --- data send buffer
      tx_len --- data send legth
      rx_buf --- data recieve buffer, return by P2PInitiatorTxRx
      rx_len --- data receive length, return by P2PInitiatorTxRx
    */
    tx_len = strlen((const char*)tx_buf);
    if(nfc.P2PInitiatorTxRx(tx_buf, tx_len, rx_buf, &rx_len)){
      /** send and receive successfully */
      Serial.print("Data Received: ");
      Serial.write(rx_buf, rx_len);
      Serial.println();
    }
    Serial.println();
  }
  
}

در این حالت خطای زیر مشاهده شد
----------------NFC_module----------------
Opening SNEP Client Link.
Read Ack
 0 0 FF 0 FF 0Read response:  0 0 FF 14 EC D5 8D 25 11 D4 0 98 C4 3D 2 68 46 37 56 E5 3A 0 0 0 0 9F 0
Read Ack
 0 0 FF 0 FF 0Read response:  0 0 FF 2D D3 D5 87 0 48 69 2C 20 74 68 69 73 20 6D 65 73 73 61 67 65 20 63 6F 6D 65 73 20 66 72 6F 6D 20 4E 46 43 20 49 4E 49 54 49 41 54 4F 52 2E E7 0
Read Ack
 0 0 FF 0 FF 0Read response:  0 0 FF 3 FD D5 8F 0 9C 0
Read Ack
 0 0 FF 0 FF 0Read response:  0 0 FF 3 FD D5 87 25 7F 0
Connection Complete Failed.
Result: 0x80000008

پس از بررسی‌های انجام شده در کتابخانه مشخص شد که این خطاها از فایل NFC_Module_DEV-HSU/NFCLinkLayer.cpp و خط‌های ۴۸ و ۹۰ برمی‌گردند.

پ.ن. ۱ این آزمون‌ها با موبایل‌های مختلف و نسخه‌های نرم‌افزاری مختلف انجام شده‌اند.
پ.ن. ۲ برای اطمینان از عملکرد ماژول با افراد مختلف اعم از پشتیبانی فنی سایت‌های آنلاین فروش صحبت شد. برخی اعتقاد داشتند که این امکان برای تبادل داده وجود ندارد و پیشنهاد دادند که مداری برای ذخیره سازی داده مانند مدار کارت‌های Mifare پیاده‌سازی شود. در مقابل برخی هم اعتقاد داشتند که با تغییر کد این امکان وجود دارد. صحبت‌های صورت گرفته در فروم آردوینو کمک کننده است.

پ.ن.۳ طبق صحبت انجام شده با برنامه‌نویس اندروید ممکن است عدم اجازه دسترسی و تبادل داده مربوط به نسخه اندروید باشد. که احتمال وجود این مورد با آزمایش نسخ مختلف اندروید کم است.

