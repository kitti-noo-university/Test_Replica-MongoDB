# ทดลอง MongoDB Replica Set บนเครื่องเดียว

ตัวอย่างเพื่อเรียนรู้การเริ่ม MongoDB 3 process เป็น replica set ชื่อ `cmpos` การเลือก PRIMARY และการเขียน/อ่านข้อมูลผ่าน replica set โดยใช้พอร์ต `27018`, `27019` และ `27020`

> **สำหรับทดลองในเครื่องเท่านั้น:** ตัวอย่างนี้ไม่มี authentication และจำกัดการรับ connection ด้วย `--bind_ip localhost` ห้ามเปิดพอร์ตสู่เครือข่ายหรือใช้กับข้อมูลสำคัญ ไม่เหมาะกับ production สมาชิกทั้งสามอยู่บนเครื่องเดียวกันจึงไม่ได้ให้ high availability เมื่อเครื่องนั้นเสีย

## ไฟล์ที่มีอยู่ใน repository

```text
Test_Replica-MongoDB/
├── README.md
├── r1.log
├── r2.log
├── r3.log
├── r1/
├── r2/
└── r3/
```

- `r1/`, `r2/`, `r3/` มีไฟล์ฐานข้อมูล WiredTiger เช่น `WiredTiger.wt`, `_mdb_catalog.wt`, `storage.bson` และ `journal/`
- `r1.log`, `r2.log`, `r3.log` เป็น log จากการทดลองเดิม โดย Build Info ใน `r1.log` ระบุ MongoDB `5.0.5` บน Windows นี่เป็นบริบทของไฟล์เดิม ไม่ใช่ข้อกำหนดให้ติดตั้งเวอร์ชันนี้
- ไม่มี application, dependency manifest หรือชุดทดสอบอัตโนมัติ ไม่ควรอนุมานว่ามี document ใดอยู่ในฐานข้อมูลจากชื่อไฟล์เหล่านี้

**อย่านำไฟล์ข้อมูลที่ commit ไว้มารันกับ MongoDB เวอร์ชันอื่นโดยตรง และอย่าลบหรือแก้ไฟล์เหล่านั้น** ขั้นตอนต่อไปใช้โฟลเดอร์ทดลองใหม่ที่อยู่นอก repository ไม่ต้องคัดลอกข้อมูลเดิม

## สิ่งที่ต้องเตรียม

- MongoDB Community Server ที่รองรับระบบปฏิบัติการของคุณ พร้อมคำสั่ง `mongod` ใน `PATH`
- MongoDB Shell (`mongosh`) ที่รองรับ server รุ่นที่เลือก ติดตั้ง shell แยกหากยังไม่มี
- พื้นที่เก็บข้อมูลและหน่วยความจำสำหรับ `mongod` 3 process และพอร์ต `27018–27020` ที่ยังว่าง
- Terminal สำหรับ server 3 หน้าต่าง และอีก 1 หน้าต่างสำหรับ shell ทุก process ต้องทำงานบนเครื่องเดียวกัน

README เดิมใช้ legacy shell `mongo` ส่วนตัวอย่างหลักด้านล่างใช้ `mongosh` ตามแนวทางปัจจุบัน [MongoDB 6.0 ถอด `mongo` ออก](https://www.mongodb.com/docs/manual/release-notes/6.0-compatibility/#removed-mongo-shell) จึงไม่ควรคาดว่าจะมีคำสั่งนี้หลังติดตั้ง server รุ่นใหม่ ตรวจการรองรับรุ่นจาก [เอกสาร MongoDB Shell](https://www.mongodb.com/docs/mongodb-shell/) สำหรับ shell ที่ติดตั้ง ไม่ได้กำหนดว่า shell รุ่นล่าสุดต้องรองรับไฟล์ข้อมูลเดิมเสมอ

ตรวจคำสั่งก่อนเริ่ม:

```sh
mongod --version
mongosh --version
```

## ขั้นตอนทดลอง

### 1. สร้างโฟลเดอร์ข้อมูลใหม่

เปิด terminal ที่ home directory ของผู้ใช้ **นอก repository** แล้วใช้ชื่อ `cm_replica_demo` ที่ยังไม่มีอยู่ หากชื่อนี้มีข้อมูลอยู่แล้ว ให้เลือกชื่อโฟลเดอร์ใหม่แทน ไม่ลบหรือใช้ข้อมูลเดิมซ้ำโดยไม่ตรวจสอบ

```sh
mkdir cm_replica_demo
cd cm_replica_demo
mkdir r1
mkdir r2
mkdir r3
```

จด absolute path ของโฟลเดอร์นี้ไว้ ใน terminal อีกสามหน้าต่างให้ `cd` มาที่โฟลเดอร์เดียวกันก่อนรันคำสั่ง เพื่อให้ `./r1`, `./r2`, `./r3` และไฟล์ log ชี้ไปที่ข้อมูลทดลองใหม่

### 2. เริ่มสมาชิกทั้งสามในสาม terminal

**Terminal 1** — พอร์ต `27018`, ข้อมูล `./r1`, log `./r1.log`:

```sh
mongod --replSet cmpos --bind_ip localhost --logappend --logpath ./r1.log --dbpath ./r1 --port 27018
```

**Terminal 2** — พอร์ต `27019`, ข้อมูล `./r2`, log `./r2.log`:

```sh
mongod --replSet cmpos --bind_ip localhost --logappend --logpath ./r2.log --dbpath ./r2 --port 27019
```

**Terminal 3** — พอร์ต `27020`, ข้อมูล `./r3`, log `./r3.log`:

```sh
mongod --replSet cmpos --bind_ip localhost --logappend --logpath ./r3.log --dbpath ./r3 --port 27020
```

ปล่อย terminal ทั้งสามทำงานไว้ คำสั่งไม่ใช้ `--fork` และ log จะลงไฟล์ จึงอาจไม่มีข้อความต่อเนื่องใน terminal ตรวจ log ว่าสมาชิกพร้อมรับ connection ก่อนทำขั้นตอนถัดไป `--logappend` ช่วยเก็บ log เดิมเมื่อเริ่ม process ใหม่

### 3. Initialize replica set เพียงครั้งเดียว

ใน **Terminal 4** เชื่อมต่อสมาชิกแรกโดยตรง:

```sh
mongosh "mongodb://localhost:27018/?directConnection=true"
```

รันใน MongoDB shell (ไม่ใช่ terminal ของระบบ):

```javascript
config = {
  _id: "cmpos",
  members: [
    { _id: 0, host: "localhost:27018" },
    { _id: 1, host: "localhost:27019" },
    { _id: 2, host: "localhost:27020" }
  ]
};
rs.initiate(config);
rs.status();
```

รอการเลือก PRIMARY แล้วตรวจ `rs.status()` อีกครั้ง สมาชิกที่พร้อมใช้งานควรมีหนึ่ง `PRIMARY` และสอง `SECONDARY` อย่าสมมติว่าพอร์ต `27018` จะเป็น PRIMARY เสมอ หากเคย initialize โฟลเดอร์ทดลองนี้แล้ว ให้ตรวจ `rs.conf()` และ `rs.status()` แทนการเรียก `rs.initiate()` ซ้ำ

### 4. เชื่อมต่อ replica set และตรวจ PRIMARY

ออกจาก shell เดิมด้วย `exit` แล้วเชื่อมต่อผ่าน seed list:

```sh
mongosh "mongodb://localhost:27018,localhost:27019,localhost:27020/?replicaSet=cmpos"
```

URI ระบุทั้งชื่อ replica set และสมาชิกเพื่อให้ shell ค้นหา PRIMARY ได้ ตรวจใน shell:

```javascript
db.hello();
rs.status();
```

ดู `isWritablePrimary: true` และ `primary` ในผล `db.hello()` พร้อมตรวจชื่อ set และ `stateStr` ของสมาชิกใน `rs.status()` ถ้ายังไม่มี PRIMARY ให้รอและตรวจ log ไม่เขียนข้อมูลจนกว่า PRIMARY จะพร้อม

**กรณีใช้สภาพแวดล้อมเดิมที่มี legacy `mongo` และรองรับ server ที่ใช้อยู่** ใช้คำสั่งต่อไปนี้แทนคำสั่งเชื่อมต่อ `mongosh` ไม่ต้องเปิดทั้งสอง shell พร้อมกัน:

```sh
mongo --host localhost --port 27018
mongo --host cmpos/localhost:27018,localhost:27019,localhost:27020
```

คำสั่งแรกสำหรับ initialize สมาชิกแรก คำสั่งที่สองสำหรับเชื่อมต่อทั้ง set ตัวอย่าง shell ด้านบนใช้กับ server ที่รองรับ `db.hello()`; หากเป็น server เก่าที่ไม่มีคำสั่งนี้ ให้ตรวจ PRIMARY ผ่าน `rs.status()` อย่าเปลี่ยนเวอร์ชันหรือเปิดไฟล์ข้อมูลเดิมเพียงเพื่อให้คำสั่งทำงาน

### 5. ทดลองสร้างฐานข้อมูลและ collection ด้วยการเขียนข้อมูล

รันเฉพาะใน replica set ทดลองใหม่ที่ PRIMARY พร้อมแล้ว:

```javascript
use test_data
db.test.insertOne({ message: "ทดลอง replica set cmpos" });
db.getCollection("test").find({});
```

`use test_data` เพียงเลือกฐานข้อมูล MongoDB จะสร้างฐานข้อมูล `test_data` และ collection `test` แบบ lazy เมื่อเขียนข้อมูลครั้งแรก ไม่ต้องสร้างไว้ล่วงหน้า การเรียก `find({})` อย่างเดียวไม่ได้สร้างข้อมูล ตัวอย่างนี้เพิ่ม document ใหม่พร้อม `_id` ที่ MongoDB สร้างให้ หากรัน `insertOne()` ซ้ำจะเพิ่ม document อีกตัว ไม่ใช่การแก้ตัวเดิม ผล query ขึ้นกับสิ่งที่คุณเขียนในรอบทดลองนี้ ไม่ใช่คำรับรองเนื้อหาของไฟล์ใน repository

## แก้ปัญหาโดยไม่ลบข้อมูล

| อาการ | สิ่งที่ควรตรวจ |
| --- | --- |
| ไม่พบ `mongod` / `mongosh` | ตรวจการติดตั้งและ `PATH` รวมถึงรุ่น shell ที่รองรับ server; server และ shell เป็นคนละโปรแกรม |
| หา data directory ไม่เจอ | ตรวจ working directory และ absolute path ว่าชี้ไปที่ `cm_replica_demo` ใหม่ และมี `r1`, `r2`, `r3` ครบ |
| พอร์ตถูกใช้งานแล้ว | ตรวจ process ที่ใช้พอร์ตผ่านเครื่องมือของระบบ เช่น `netstat` หรือ `lsof` อย่าหยุด process ที่ไม่ทราบเจ้าของ |
| Connection refused / timeout | ตรวจว่า server ทั้งสามยังทำงาน ดู log ตามสมาชิก และตรวจว่า `localhost` resolve ไปยัง loopback ที่เข้าถึงได้ ไม่แก้ด้วยการ bind `0.0.0.0` หรือ `--bind_ip_all` |
| Already initialized | ตรวจ `rs.conf()` และ `rs.status()` ก่อน ไม่ initialize ซ้ำหรือแก้ config โดยเดาสุ่ม |
| ไม่มี PRIMARY / เขียนไม่ได้ | ตรวจ `rs.status()` และ log ให้สมาชิกอย่างน้อยสองตัวสื่อสารกันได้ รอ election แล้วเชื่อมต่อด้วย URI ที่มี `replicaSet=cmpos` แทนการเขียนไปยัง SECONDARY โดยตรง |
| Data directory ถูกล็อก / เปิดข้อมูลไม่ได้ | ตรวจว่ามี process อื่นใช้ path เดียวกันหรือไม่ และตรวจรุ่น server จาก log อย่าลบ lock, journal หรือ WiredTiger และอย่าใช้ `--repair` เป็นวิธีแก้เริ่มต้น |

หากต้องการเริ่มทดลองอีกรอบ ให้ปิด process อย่างเรียบร้อยและใช้โฟลเดอร์ใหม่ เก็บข้อมูลเดิมไว้ ไม่ต้องใช้คำสั่งลบฐานข้อมูลหรือไฟล์เพื่อแก้ปัญหา

## ปิดการทดลองอย่างเรียบร้อย

1. ออกจาก MongoDB shell ด้วย `exit` (การออกจาก shell ไม่ได้หยุด server)
2. กด `Ctrl+C` ใน terminal ของ `mongod` ทีละหน้าต่างทั้งสาม แล้วรอให้ process จบและ log แสดงว่าปิดเรียบร้อย หลีกเลี่ยง force kill
3. เก็บโฟลเดอร์ข้อมูลและ log ไว้ เมื่อต้องการรันต่อ ใช้ path เดิมและคำสั่งเริ่มทั้งสาม ไม่ต้อง initialize ใหม่

## เอกสารอ้างอิงและการตรวจสอบ

- [Deploy a Replica Set for Testing](https://www.mongodb.com/docs/manual/tutorial/deploy-replica-set-for-testing/)
- [ตัวเลือก `mongod` รวมถึง `--bind_ip`](https://www.mongodb.com/docs/manual/reference/program/mongod/)
- [เชื่อมต่อด้วย MongoDB Shell](https://www.mongodb.com/docs/mongodb-shell/connect/)
- [`rs.initiate()`](https://www.mongodb.com/docs/manual/reference/method/rs.initiate/) และ [`db.hello()`](https://www.mongodb.com/docs/manual/reference/method/db.hello/)
- [Databases and Collections](https://www.mongodb.com/docs/manual/core/databases-and-collections/)

การปรับ README ครั้งนี้ตรวจชื่อ set พอร์ต และ path เทียบกับไฟล์จริง พร้อมตรวจรูปแบบคำสั่งและเอกสารอ้างอิง **ไม่ได้รันทดสอบ MongoDB runtime หรือเขียนข้อมูล** ในสภาพแวดล้อมงานนี้ไม่มี `mongod`, `mongo` หรือ `mongosh` และไม่ได้เปิดหรือแก้ไขไฟล์ฐานข้อมูลที่ commit ไว้
