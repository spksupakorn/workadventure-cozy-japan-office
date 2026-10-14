Link หลัก

https://wa.alternatepro.com/

ลิงก์นี้จะพาไปที่ starter map ตาม START*ROOM_URL ให้เอง ถ้าต้องการเข้าห้องนั้นตรงๆ ใช้:
https://wa.alternatepro.com/*/global/workadventure.github.io/map-starter-kit/office.tmj

แชร์ link เดียวกันนี้ให้คนอื่น เมื่อเข้าห้องเดียวกันจะเห็นกันและคุยกันได้

ใช้ map ของตัวเอง

1. ใน .env ตั้ง MAP_STORAGE_AUTHENTICATION_USER และ MAP_STORAGE_AUTHENTICATION_PASSWORD ตอนนี้สองค่านี้ว่างอยู่ แต่เปิด basic auth ไว้ ทำให้ upload ไม่ได้
2. รัน docker compose up -d
3. ใน project map-starter-kit รัน npm run upload แล้วใส่ URL https://wa.alternatepro.com/map-storage/ กับ user/password จากข้อ 1
4. เข้า map ด้วย https://wa.alternatepro.com/~/maps/<ชื่อ map>.wam

ข้อควรระวัง

- Map editor: ตอนนี้ .env ตั้ง MAP_EDITOR_ALLOW_ALL_USERS=true และ DISABLE_ANONYMOUS=false แปลว่าใครก็ตามที่มี link เข้ามาแก้ map ได้ ถ้าเปิดให้คนนอกใช้ ควรตั้งเป็น false แล้วระบุคนที่แก้ได้ใน MAP_EDITOR_ALLOWED_USERS
- Video: ตอนนี้ยังไม่ได้ตั้ง TURN_SERVER และ LiveKit ผู้ใช้ที่อยู่หลัง NAT หรือ firewall ขององค์กรอาจเห็น video กันไม่ได้ ถ้าเจอปัญหานี้ต้องติดตั้ง coturn เพิ่ม
- gRPC port: GRPC_PORT 50051 เปิดออก public อยู่ ถ้าไม่ได้ใช้ Room API ให้ปิด port นี้ใน firewall (ufw)

มี 2 วิธี ใช้คู่กันได้ map ปัจจุบัน (`office.tmj` บน `workadventure.github.io`) อยู่นอก server ของคุณ จึงแก้ไม่ได้ ต้อง upload map ของตัวเองเข้า map-storage ก่อน

## วิธีที่ 1: Tiled + map-starter-kit

ใช้แก้ layout, พื้น, ผนัง และ tile ต่างๆ ต้องมี Node.js 18 ขึ้นไป และ [Tiled](https://www.mapeditor.org/)

1. Clone และติดตั้ง
   ```bash
   git clone https://github.com/workadventure/map-starter-kit.git
   cd map-starter-kit
   npm install
   npm run dev        # preview ในเครื่องก่อน
   ```
2. เปิดไฟล์ `.tmj` ใน Tiled (เช่น `office.tmj`) แก้แล้ว save ถ้าเปิด `npm run dev` ค้างไว้ หน้า preview จะอัปเดตตาม
3. Upload ขึ้น server:
   ```bash
   npm run upload
   ```

   - Map storage URL: `https://wa.alternatepro.com/map-storage/`
   - API key: ค่า `MAP_STORAGE_API_TOKEN` ใน `.env` ตอนนี้ค่านี้ว่างอยู่ ระบบจึงใช้ `SECRET_KEY` แทน (`docker-compose.yml` ตั้งไว้เป็น `${MAP_STORAGE_API_TOKEN:-${SECRET_KEY}}`) แนะนำให้ตั้ง `MAP_STORAGE_API_TOKEN` แยกเป็นค่าสุ่มยาวๆ แล้วรัน `docker compose up -d`
   - Directory: ชื่อโฟลเดอร์ เช่น `office`
4. เข้า map ด้วย `https://wa.alternatepro.com/~/maps/office/office.wam` URL จริงขึ้นกับ directory และชื่อไฟล์ หลัง upload เสร็จสคริปต์จะแสดง URL ให้

ผมต้องแก้สิ่งที่บอกไปก่อนหน้านี้: สคริปต์ upload ใช้ **API key** ไม่ใช่ user/password ส่วน `MAP_STORAGE_AUTHENTICATION_USER` และ `MAP_STORAGE_AUTHENTICATION_PASSWORD` ใช้กับหน้าเว็บ map-storage ไม่ใช่การ upload

## วิธีที่ 2: Map editor ใน browser

ใช้กับ map ที่ upload เข้า map-storage แล้ว (`.wam`) เท่านั้น

1. เปิด map `.wam` ของคุณ
2. กดไอคอน map editor ใน toolbar ได้เพราะตอนนี้ตั้ง `ENABLE_MAP_EDITOR=true`
3. สิ่งที่แก้ได้: วางเฟอร์นิเจอร์และ object, สร้าง area (meeting room, Jitsi, link, silent zone), ตั้งค่า map
4. ระบบ save อัตโนมัติ และคนที่อยู่ในห้องจะเห็นการเปลี่ยนแปลงทันที

Map editor **แก้ tile, พื้น และผนังไม่ได้** ถ้าต้องแก้ส่วนนี้ให้กลับไปใช้วิธีที่ 1 แล้ว upload ใหม่

## ตั้งให้ map ใหม่เป็นหน้าแรก

ใน `.env`:

```
START_ROOM_URL=/~/maps/office/office.wam
```

จากนั้นรัน `docker compose up -d` แล้ว `https://wa.alternatepro.com/` จะพาไปที่ map ใหม่

**คำเตือน:** การ upload ซ้ำด้วย directory เดิมจะเขียนทับ map บน server ของที่วางไว้ด้วย map editor อาจหายได้ ให้ backup volume `map-storage-data` ก่อน upload ทับ
