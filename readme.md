# กล่องยาอัจฉริยะ SmartMedicine

นี่คือเอกสารการใช้งานโค้ดของ SmartMedicine แห่ง UMR-LAB

ถ้าเกิดว่าเอกสารนี้ไม่มีความชัดเจนมากพอ ผู้เขียนเอกสารขอแนะนำว่า ให้ไปไล่โค้ดเองนะจ๊ะ

### ส่วนประกอบหลัก
โปรแกรมนี้ประกอบไปด้วย 3 ส่วนได้แก่
1. [Main.py](#mainpy)
2. [checkAlert.py](#checkalertpy)
3. [popup.py](#popuppy)
4. [launcher.sh](#launchersh)
5. [Fall.ino](#fallino)

## Main.py
*/home/pi/Desktop/SmartMedicine2/Main.py*

ตัวโปรแกรมหลัก ไฟล์นี้จะทำหน้าที่ในการสร้าง GUI และเชื่อมต่อ Serial Port ทั้งหมด 
โดยตัวโปรแกรม จะเริ่มต้นทำงานที่บรรทัดล่างสุดของไฟล์

```python
if __name__ == "__main__":
    import sys
    print('Initialzing SmartMedicine...')
    app = QtGui.QApplication(sys.argv)
    MainWindow = QtGui.QMainWindow()
    ui = Ui_MainWindow() # Initialze GUI
    ui.setupUi(MainWindow)
    MainWindow.showMaximized()
    sys.exit(app.exec_())
```

จากโค้ด จะเห็นได้ว่า เมื่อทำการรันไฟล์นี้ ตัวโปรแกรมจะทำการสร้าง Object ของคลาส **Ui_MainWindow**
ที่ถูกประกาศไว้ด้านบน และทำการเรียกเมทธอด **setupUi** เพื่อทำการสร้างปุ่มต่าง ๆ ของหน้าจอหลัก

```python
def setupUi(self, MainWindow):
    #...
    self.btm.clicked.connect(self.bt1)
    self.btc.clicked.connect(self.bt2)
    self.bth.clicked.connect(self.bt3)
```

จากโค้ดข้างบน หลังจากที่ตัวโปรแกรมทำการสร้างหน้าจอ UI เสร็จแล้ว ตัวโค้ดจะทำเชื่อมปุ่ม btm, btc และ bth
เข้ากับเมทธอด bt1, bt2 และ bt3 ตามลำดับ ซึ่งแต่ละเมทธอดจะทำการเรียกใช้ไฟล์ Python อื่น ๆ ของระบบ

ยกตัวอย่างเช่นปุ่ม btm

```python
from user import Ui_manageuser

#...

def bt1 (self):
    self.userWindow = QtGui.QDialog()
    self.ui = Ui_manageuser() # See user.py
    self.ui.setupUi(self.userWindow)
    self.userWindow.show()
```

ทันทีที่กดปุ่ม btm โปรแกรมจะมาทำงานที่ bt1 ทันทีและมันจะทำการเรียกใช้ไฟล์ **user.py** ที่
ถูก import มาไว้ข้างบน และหน้าจอจัดการผู้สูงอายุจะโผล่ขึ้นมา

## checkAlert.py
*/home/pi/Desktop/SmartMedicine2/checkAlert.py*

ไฟล์นี้ทำหน้าที่ในการตรวจสอบเวลาปัจจุบันแบบวินาทีต่อวินาทีเพื่อคอยตรวจสอบว่า ถึงเวลากินยาแล้วหรือไม่

ทุก ๆ 1 วินาที ไฟล์นี้จะทำการเรียกใช้เมทธอด **tick()** ซึ่งทำหน้าที่ในการดึงข้อมูลเวลากินยาจาก
ฐานข้อมูล SQlite ของโปรแกรม

```python
c = con.cursor()
c1 = con.execute("SELECT time(strftime('%J', br)-(b_br*0.000694444868713617)) FROM user WHERE u_id =1")
x1 = c1.fetchone()
```

และทำการเทียบกับเวลาปัจจุบันเพื่อตรวจสอบว่าถึงเวลากินยาแล้วหรือไม่

```python
from Alert1Bbr import Ui_Alert1

#...

if (str(x1[0]) == CurrentTime ):
    stat = 1
    publishMsg(stat) # Publish MQTT message
    app = QtGui.QApplication(sys.argv)
    Alert = QtGui.QDialog()
    ui = Ui_Alert1(stat) # See Alert1Bbr.py
    ui.setupUi(Alert)
    Alert.show()
    Ui_Alert1(stat)
    app.exec_()
```

หากถึงเวลากินยาแล้ว ตัวโปรแกรมจะทำการ Publish MQTT message ไปที่อุปกรณ์ที่ติดตัวผู้สูงอายุ
และแสดงหน้าจอบนโปรแกรมว่าถึงเวลากินยาแล้ว โดย MQTT message ที่ Publish ออกไปจะส่งออกไปที่
topic ***umr/pills/ไอดีผู้สูงอายุ*** โดยใช้โค้ดข้างล่างนี้

```python
def publishMsg(stat):
    print("Publishing to topic 'umr/pills/'" + str(stat) + "....")
    client.publish("umr/pills/" + str(stat), "time for pills", 0)
```

## popup.py
*/home/pi/Desktop/pyutil/popup.py*

ไฟล์นี้จะทำการ Subscribe ไปที่ topic ***umr/fall*** เพื่อคอยรับข้อควาจากอุปกรณ์ที่ติดตัว
สูงอายุ โดยใช้เมทธอด ***on_connect()*** เมื่อตัวไฟล์ทำการเชื่อมต่อ MQTT Broker สำเร็จ

```python
def on_connect(client, userdata, flags, rc):
    print("Connected!")
    client.subscribe("umr/fall")

def on_message(client, userdata, msg):
    print("Message recieved on topic" + msg.topic)
    call = threading.Thread(target=process_msg, args=(msg,))
    call.start()
```

และเมื่อมีข้อความเข้ามาที่ topic นี้ โปรแกรมจะทำการเรียกใช้เมทธอด ***on_message()***
และทำการสร้าง Process ที่เมทธอด ***process_msg()*** ใหม่เพื่อแสดงหน้าจอแจ้งเตือนว่ามีคนล้ม

```python
def process_msg(msg):
    con = sqlite3.connect('/home/pi/Desktop/SmartMedicine2/DB/DB')
    cursor = con.execute("SELECT o_name FROM user WHERE u_id = '%s' "% (msg.payload))
    result = cursor.fetchone()
    name = result[0]

    pygame.mixer.init()
    pygame.mixer.music.load('/home/pi/Desktop/SmartMedicine2/alarm.mp3')
    pygame.mixer.music.play()

    root = Tk()
    root.option_add('*Dialog.msg.font', 'Helvetica 24')
    root.withdraw()
    tkMessageBox.showwarning('ฉุกเฉิน!', name.encode('utf-8') + ' ล้ม!')
    root.update()
    root.destroy()

    time.sleep(5)
    pygame.mixer.music.stop()
```
เมทธอด ***process_msg()*** จะทำการอ่าน Payload ของข้อความว่าไอดีของผู้สูงอายุที่ล้มมีไอดีอะไร
จากนั้น จะทำการดึงชื่อของผู้สูงอายุออกมาจากฐานข้อมูล SQlite ของโปรแกรม และเล่นไฟล์เสียงแจ้งเตือน
และแสดงหน้าจอ Popup ว่ามีคนล้ม

## launcher.sh
*/home/pi/launcher.sh*

ไฟล์ Bash Script ที่จะถูกเรียกทันทีที่ Raspberry Pi ทำการเรียกใช้งานหน้าจอ GUI เมื่อเปิดเครื่อง
ทำให้ทุกโปรแกรมที่จำเป็นถูกเปิดขึ้นมาทันที Raspberry Pi เปิด

```bash
#!/bin/sh
nohup python /home/pi/Desktop/SmartMedicine2/Main.py &
nohup python /home/pi/Desktop/SmartMedicine2/checkAlert.py &
nohup python /home/pi/pyutil/popup.py &
```

ใช้ nohup และ & ทำให้ทุกโปรแกรมเปิดขึ้นมาโดยไม่ขัดขวางการทำงานของโปรแกรมอื่น ๆ ใน Raspberry Pi

## Fall.ino
*โปรแกรมสำหรับอุปกรณ์ติดตัวผู้สูงอายุ*

โปรแกรมนี้จะทำงานบน Wemos LOLIN32 ร่วมกับ ADXL345 เพื่อทำการตรวจสอบว่าผู้สูงอายุล้มหรือไม่
ซึ่งการที่โปรแกรมจะรู้ได้ว่าผู้สูงอายุล้มหรือไม่ จะขึ้นอยู่กับ Threshold ที่กำหนดไว้ในโค้ด

```c
ADXL345 adxl = ADXL345();

/** ... */

adxl.setFreeFallThreshold(10);
adxl.setFreeFallDuration(15); 
```

และถ้าหากว่ามีการล้มเกิดขึ้น โปรแกรมจะทำการ Publish MQTT Message ไปที่ตู้ยาทันที และ
ตู้ยาจะทำการแสดงข้อความว่าผู้สูงอายุล้มที่หน้าจอ

โดยข้อความที่ Publish ไปจะเป็น **ไอดีของผู้สูงอายุ** ซึ่งในตัวอย่างนี้คือ 1

```c
if(adxl.triggered(interrupts, ADXL345_FREE_FALL)){
    client.publish("umr/fall", "1");
    Serial.println("*** FREE FALL ***");
    beep(1000);
} 
```

ถ้าหากว่ามีข้อความจากตู้ยาว่า ถึงเวลากินยาแล้ว ตัวโปรแกรมจะทำการส่งเสียงเตือนให้ผู้สูง
อายุได้ยิน

```c
// What to do when the message arrives
void callback(char* topic, byte* payload, unsigned int length) {
  Serial.print("Message arrived [");
  Serial.print(topic);
  Serial.print("] ");
  
  beep(500);
  beep(500);
  beep(500);
  
  beep(1000);
  
  beep(500);
  beep(500);
  beep(500);
}
```