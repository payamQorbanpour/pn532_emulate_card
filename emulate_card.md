برنامه شبیه‌ساز کارت ماژول PN532

نیازمندی‌ها: سخت افزار

ماژول ان اف سی PN532 V3
ماژول ESP32 (برای استفاده از سریال سخت افزاری دوم آن برای اتصال به ماژول NFC)

نیازمندی‌ها: نرم افزاری

کتابخانه seeed arduino nfc

نیازمندی‌ها: نرم‌افزار تست موبایل

برای اندروید: NFC Tools
برای آی‌او‌اس: NFC taginfo

ارتباطات: 

برای ارتباط ماژول ESP32 با ماژول PN532 از پروتکل HSU استفاده شده است. به این صورت که پایه‌های TX و RX روی ماژول PN532 به ترتیب(به صورت ضربدری) به پایه‌های RX2 و TX2 به ماژول ESP32 متصل شده است. همچنین پایه های GND و VCC هم به ترتیب به پایه GND و 3.3v ماژول ESP32 متصل شده‌اند. اتصال پایه VCC به پایه 5v ماژول ESP32 تفاوتی در رفتار نرم‌افزار ایجاد نمی‌کند.
توجه: روی ماژول PN532 دو سوئیچ برای تغییر حالت اتصال وجود دارد که مطابق جدول چاپ شده روی ماژول، باید روی حالت HSU قرار بگیرد. (هر دو سوئیچ روی وضعیت 0 باشند)

نرم‌افزار:

برای نوشتن نرم‌افزار و بارگذاری آن روی ماژول ESP32 از محیط توسعه ArduinoIDE استفاده شده است.
مراحل بارگذاری نرم افزار به صورت زیر است:

نصب کتابخانه‌های مورد نیاز 
کپی کردن کد شبیه ساز کارت
#include "SPI.h"
#include "PN532_SPI.h"
#include "emulatetag.h"
#include "NdefMessage.h"

PN532_SPI pn532spi(SPI, 10);
EmulateTag nfc(pn532spi);

uint8_t ndefBuf[120];
NdefMessage message;
int messageSize;

uint8_t uid[3] = { 0x12, 0x34, 0x56 };

void setup()
{
  Serial.begin(115200);
  Serial.println("------- Emulate Tag --------");
  
  message = NdefMessage();
  message.addUriRecord("http://www.seeedstudio.com");
  messageSize = message.getEncodedSize();
  if (messageSize > sizeof(ndefBuf)) {
      Serial.println("ndefBuf is too small");
      while (1) { }
  }
  
  Serial.print("Ndef encoded message size: ");
  Serial.println(messageSize);

  message.encode(ndefBuf);
  
  // comment out this command for no ndef message
  nfc.setNdefFile(ndefBuf, messageSize);
  
  // uid must be 3 bytes!
  nfc.setUid(uid);
  
  nfc.init();
}

void loop(){
    // uncomment for overriding ndef in case a write to this tag occured
    //nfc.setNdefFile(ndefBuf, messageSize); 
    
    // start emulation (blocks)
    nfc.emulate();
        
    // or start emulation with timeout
    /*if(!nfc.emulate(1000)){ // timeout 1 second
      Serial.println("timed out");
    }*/
    
    // deny writing to the tag
    // nfc.setTagWriteable(false);
    
    if(nfc.writeOccured()){
       Serial.println("\nWrite occured !");
       uint8_t* tag_buf;
       uint16_t length;
       
       nfc.getContent(&tag_buf, &length);
       NdefMessage msg = NdefMessage(tag_buf, length);
       msg.print();
    }

    delay(1000);
}




تغییر کد به صورت زیر برای تغییر کتابخانه‌های داخلی و تغییر SPI به HSU:
جایگزینی کد
#include <NfcAdapter.h>
#include <PN532/PN532/PN532.h>
#include <PN532/PN532/emulatetag.h>
#include <PN532/PN532_HSU/PN532_HSU.h>
#include "NdefMessage.h"

PN532_HSU pn532hsu(Serial2);
EmulateTag nfc(pn532hsu);


	به جای کد
#include "SPI.h"
#include "PN532_SPI.h"
#include "emulatetag.h"
#include "NdefMessage.h"

PN532_SPI pn532spi(SPI, 10);
EmulateTag nfc(pn532spi);

	
تغییر کد کتابخانه در فایل Seeed_Arduino_NFC-master/src/PN532/PN532/emulatetag.cpp:76 طبق این ایشو در گیت‌هاب برای کار با گوشی‌های اندروید
جایگزینی کد
 uint8_t command[] = {
      PN532_COMMAND_TGINITASTARGET,
      5, // MODE: PICC only, Passive only

      0x04, 0x00,       // SENS_RES
      0x00, 0x00, 0x00, // NFCID1
      0x20,             // SEL_RES

      0, 0, 0, 0, 0, 0, 0, 0,
      0, 0, 0, 0, 0, 0, 0, 0, // FeliCaParams
      0, 0,

      0, 0, 0, 0, 0, 0, 0, 0, 0, 0, // NFCID3t

      0, // length of general bytes
      0  // length of historical bytes
  };

	با کد
 uint8_t command[] = {
      PN532_COMMAND_TGINITASTARGET,
      0x05,                  // MODE: 0x04 = PICC only, 0x01 = Passive only (0x02 = DEP only)

      // MIFARE PARAMS
      0x04, 0x00,         // SENS_RES (seeeds studio set it to 0x04, nxp to 0x08)
      0x00, 0x00, 0x00,   // NFCID1t	(is set over sketch with setUID())
      0x20,               // SEL_RES    (0x20=Mifare DelFire, 0x60=custom)

      // FELICA PARAMS
      0x01, 0xFE,         // NFCID2t (8 bytes) https://github.com/adafruit/Adafruit-PN532/blob/master/Adafruit_PN532.cpp FeliCa NEEDS TO BEGIN WITH 0x01 0xFE!
      0x05, 0x01, 0x86,
      0x04, 0x02, 0x02,
      0x03, 0x00,         // PAD (8 bytes)
      0x4B, 0x02, 0x4F, 
      0x49, 0x8A, 0x00,   
      0xFF, 0xFF,         // System code (2 bytes)
      
      0x01, 0x01, 0x66,   // NFCID3t (10 bytes)
      0x6D, 0x01, 0x01, 0x10,
      0x02, 0x00, 0x00,

	  0x00, // length of general bytes
      0x00  // length of historical bytes
  };


نکته ۱: طی آزمایشی که با استفاده از حدود ۲۰ موبایل، اعم از Android و iOS انجام شد، حدود ۲۰ درصد موبایل‌های اندرویدی (بعضا دارای اندروید beam) امکان اسکن ماژول ان‌اف‌سی را نداشتند. گوشی‌های دارای ‌سیستم عامل iOS مشکلی در اسکن ماژول نداشتند.
نکته ۲: استاندارد تولید شده توسط ماژول ان‌اف‌سی ISO 14443-4 است در حالی که کارت‌های مایفر ISO 14443-3A است که ممکن است این امر دلیل اسکن نشدن ماژول توسط بعضی از موبایل‌های اندرویدی مذکور در نکته ۱ باشد.



سوالات:
خواندن ماژول توسط موبایل نیازمند جای دهی دقیق موبایل روی ماژول به مدت تقریبی ۱ ثانیه است. آیا امکان سرعت دهی به این فرآیند وجود دارد؟

در کد شبیه‌ساز کارت، در خط ۴۸، از متود 
nfc.emulate();

	استفاده شده است که در این خط برنامه(loop) به صورت کلی می‌ایستد و منتظر خواندن توسط موبایل می‌ماند. این موضوع باعث عدم عملکرد درست نرم‌افزار می‌شود چون ادامه برنامه، همچنین تابع loop اجرا نمی‌شود.

