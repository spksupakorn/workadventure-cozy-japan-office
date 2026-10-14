# Cozy Japanese Office Map — Design

- วันที่: 2026-10-08
- สถานะ: อนุมัติแล้ว (แก้ไขครั้งที่ 2 หลังตรวจ map-starter-kit 3.3.18)
- แทนที่บางส่วนโดย: `docs/superpowers/specs/2026-10-14-cozy-office-visual-redesign-design.md` (ภาพ ผังห้อง และการสร้างผนังกับพื้น)
- ปลายทาง: WorkAdventure self-hosted ที่ `https://wa.alternatepro.com` (map-storage)

## 1. เป้าหมาย

สร้าง map ออฟฟิศเสมือนสไตล์ cozy ญี่ปุ่น สำหรับทีม 10-20 คน มีพื้นที่ทำงาน ห้องประชุม พื้นที่พักผ่อน สัตว์เลี้ยง และมุมเล่น
ผลลัพธ์คือ map ที่ upload ขึ้น map-storage ได้ และตั้งเป็น `START_ROOM_URL` ของ server

### สิ่งที่ผู้ใช้กำหนด
- สไตล์ cozy ญี่ปุ่น
- ออฟฟิศขนาดเล็กถึงกลาง รองรับ 10-20 คน
- มีพื้นที่อำนวยความสะดวกครบ
- มีสัตว์เลี้ยงและอุปกรณ์เล่น
- tileset: GuttyKreum "The Japan Collection: Interior Essentials" (https://guttykreum.itch.io/japanese-interior-essentials, $8.99) ผู้ใช้ซื้อและดาวน์โหลดเอง

### สมมติฐาน
- ใช้ map-starter-kit เป็นฐาน (Vite, `npm run dev`, `npm run upload`)
- tile ขนาดจริงของ pack ยังไม่รู้ จะตรวจเมื่อได้ไฟล์ (ดูข้อ 5)
- Jitsi ใช้ `meet.jit.si` ตามค่า `JITSI_URL` ปัจจุบันใน `.env`

### เกณฑ์ความสำเร็จ
1. map โหลดได้บน `wa.alternatepro.com` โดยไม่มี error ใน browser console
2. จากจุดเกิด เดินไปถึงทุกโซนได้
3. ห้องประชุมทั้ง 2 ห้องเปิด Jitsi ได้ focus booth ปิดเสียงได้
4. interaction ของสัตว์เลี้ยง, ปลาคาร์ฟ และ omikuji ทำงาน
5. ผู้ใช้เปิด `cozy-office.tmj` แก้ต่อใน Tiled ได้

### นอกขอบเขต
ห้องน้ำ, ลิฟต์, หลายชั้น, สัตว์ที่เดินได้เองและทุกคนเห็นตำแหน่งตรงกัน (WorkAdventure ไม่มีระบบ NPC)

## 2. Layout และโซน

ขนาด map 56x38 tiles (1792x1216 px ที่ 32x32) ผังด้านล่างเป็นภาพคร่าวๆ พิกัดจริงกำหนดใน `tools/layout.py`

```
 x: 0         14                    28                42          55
 y0 +----------+--------------------+-----------------+-----------+
    | ห้อง     |  Lounge            |  Pantry 給湯室   |  遊び場    |
    | ประชุม   |  (โคทัตสึ, แมว 1)  |  (เคาน์เตอร์ชา)  |  Play     |
    | ใหญ่ 8   |                    |                 |  corner   |
 y13+----------+--------------------+-----------------+           |
    | ทาทามิ   |                                      +-----------+
    | ประชุม 4 |        Open office                   |  Engawa   |
 y23+----------+        โต๊ะ 20 ที่ (5 กลุ่ม x 4)       |  + สวน    |
    | Focus x2 |                                      |  บ่อปลา   |
 y29+----------+                                      |  ศาลเจ้า  |
    | White-   |                                      |  omikuji  |
    | board    |                                      |  แมว 2    |
 y34+----------+--------+  玄関 Genkan  +-------------+           |
    |                   |  จุดเกิด, ชิบะ |                         |
 y39+-------------------+--------------+-------------------------+
```

| โซน | รายละเอียด | ฟีเจอร์ WorkAdventure |
|---|---|---|
| 玄関 Genkan | ทางเข้า ชั้นวางรองเท้า ป้ายต้อนรับ ชิบะ | `start` layer |
| Open office | โต๊ะไม้ 20 ที่ แบ่ง 5 กลุ่ม กลุ่มละ 4 | proximity chat ปกติ |
| ห้องประชุมใหญ่ | โต๊ะยาว 8 ที่นั่ง ผนังโชจิ | area `jitsiRoom: "cozy-meeting-large"`, `jitsiTrigger: "onaction"`, `focusable: true` |
| ห้องประชุมทาทามิ | โต๊ะเตี้ย 4 ที่นั่ง เบาะ zabuton | area `jitsiRoom: "cozy-meeting-tatami"`, `jitsiTrigger: "onaction"`, `focusable: true` |
| Focus booth x2 | ห้องเดี่ยว 1 คน | area `silent: true` |
| Pantry 給湯室 | เคาน์เตอร์ชา ตู้เย็น โต๊ะกินข้าว | proximity chat ปกติ |
| Lounge | โคทัตสึหรืออิโรริ เบาะ zabuton แมว 1 ตัว | proximity chat ปกติ |
| Engawa + สวน | ระเบียงไม้ บ่อปลาคาร์ฟ ซากุระ โคมหิน ศาลเจ้าเล็ก แมว 1 ตัว | interaction (ข้อ 4) |
| Whiteboard corner | กระดานไวท์บอร์ด | area `openWebsite` ไปที่ Excalidraw, `openWebsiteTrigger: "onaction"` |
| 遊び場 Play corner | โต๊ะปิงปอง กระดานโชงิหรือโกะ ตู้ arcade มุม lo-fi | area `openWebsite` + `openWebsiteTrigger: "onaction"` ต่อชิ้น |

หมายเหตุ Play corner: ทุก URL ต้องทดสอบว่าเว็บยอมให้แสดงใน iframe ก่อนใส่ลง map เว็บที่ไม่ยอมจะไม่ถูกใส่ โต๊ะปิงปองเป็นของตกแต่งเท่านั้น

## 3. โครงไฟล์

```
maps/cozy-office/              (clone จาก workadventure/map-starter-kit 3.3.18)
├── tilesets/WA_Special_Zones.png   มาจาก starter kit ใช้ tile จุดเกิดและ collision
├── tilesets/guttykreum/       pack ต้นฉบับที่ผู้ใช้ซื้อ ห้าม commit
├── tilesets/guttykreum32/     pack ที่แปลงเป็น 32x32 แล้ว (ถ้าต้องแปลง) ห้าม commit
├── tilesets/custom/           sprite สำรองที่วาดด้วย code (สัตว์ ปลา ของเล่น)
├── tools/
│   ├── mapgen/model.py        dataclass: Rect, Fill, Stamp, Area, Popup, Layout, Tileset, ObjectDef, Registry
│   ├── mapgen/render.py       Layout + Registry -> grid ของแต่ละ layer และ grid ช่องที่เดินไม่ได้
│   ├── mapgen/validate.py     การตรวจทั้งหมดในข้อ 6
│   ├── mapgen/tmj.py          เขียนโครงสร้าง Tiled JSON (tileset ฝังใน .tmj)
│   ├── vocab.py               รายชื่อ tile และ object ที่ layout ใช้ได้ พร้อมขนาด
│   ├── layout.py              ผังออฟฟิศ
│   ├── tiles.py               ชื่อ -> tile id ของ pack และ sprite สำรอง
│   ├── draw_custom.py         วาด tilesets/custom/cozy_custom.png
│   ├── inspect_tileset.py     สร้างภาพ contact sheet ที่มีเลข tile id
│   ├── rescale_tileset.py     แปลงขนาด tile เป็น 32x32
│   ├── check_embed.py         ตรวจว่าเว็บใน Play corner ยอมให้แสดงใน iframe
│   └── build_map.py           CLI: validate แล้วเขียน cozy-office.tmj
├── cozy-office.tmj            ไฟล์ output เปิดแก้ใน Tiled ได้
├── src/omikuji.ts             logic omikuji (ไม่ใช้ WA API ทดสอบด้วย node --test ได้)
├── src/interactions.ts        รายการ interaction: area, popup, layer, ข้อความ
└── src/main.ts                ผูก interaction เข้ากับ WA API
```

แต่ละ unit มีหน้าที่เดียว:
- `tiles.py` รู้เรื่อง tileset อย่างเดียว ถ้าเปลี่ยน pack แก้แค่ไฟล์นี้
- `layout.py` รู้เรื่องผังอย่างเดียว ใช้ชื่อจาก `vocab.py` (เช่น `"floor_tatami"`) ไม่ใช้ tile id
- `vocab.py` เป็นสัญญาระหว่าง `layout.py` กับ `tiles.py`
- `mapgen/` แปลงผังเป็นรูปแบบ Tiled และตรวจความถูกต้อง ไม่รู้จักออฟฟิศนี้
- `src/main.ts` อ้าง area ตามชื่อเท่านั้น ไม่ผูกกับพิกัด

## 4. โครง map และ script

### Layer
| Layer | ชนิด | หน้าที่ |
|---|---|---|
| `floor` | tile | พื้นไม้ ทาทามิ หิน หญ้า น้ำ |
| `walls` | tile | ผนัง โชจิ ประตู |
| `furniture` | tile | เฟอร์นิเจอร์ใต้ตัวละคร |
| `collisions` | tile | ช่องที่เดินไม่ได้ วาดด้วย tile id 2 ของ `WA_Special_Zones` (`collides: true`) ซ่อนไว้ |
| `start` | tile | จุดเกิดที่ genkan property `startLayer: true` ซ่อนไว้ |
| `floorLayer` | objectgroup | ระดับที่วาดตัวละคร (WorkAdventure บังคับให้มี) และเก็บ area object ทุกตัว (`type: "area"`) กับ popup anchor |
| `above` | tile | หลังคา กิ่งซากุระ ส่วนบนของเฟอร์นิเจอร์สูง |
| `koi-feeding` | tile | animation ให้อาหารปลา ซ่อนไว้ตั้งต้น |

Collision ใช้ layer `collisions` ตามแบบ starter kit ไม่ต้องแก้ tileset ที่ซื้อมา generator รู้อยู่แล้วว่าช่องไหนเดินไม่ได้ (ผนัง เฟอร์นิเจอร์ น้ำ) จึงเขียน layer นี้ให้อัตโนมัติ
สัตว์และปลาใช้ animated tile ของ Tiled

### Interaction (`src/main.ts`)
| Area | การทำงาน |
|---|---|
| `pet-cat-1`, `pet-cat-2`, `pet-shiba` | เมื่อเข้า area: `WA.ui.displayActionMessage` แสดง "กด SPACE ลูบ..." กดแล้วเปิด `WA.ui.openPopup` แสดงข้อความของสัตว์ ("にゃ〜" / "わん!") เมื่อออกจาก area: ปิด message และ popup |
| `koi-pond` | action message "กด SPACE ให้อาหารปลา" กดแล้ว `WA.room.showLayer("koi-feeding")` 3 วินาทีแล้ว `hideLayer` แต่ละคนเห็นแยกกันในเครื่องตัวเอง |
| `omikuji` | action message "กด SPACE เสี่ยงเซียมซี" กดแล้วสุ่มจาก 7 ระดับ (大吉, 中吉, 小吉, 吉, 末吉, 凶, 大凶) แต่ละระดับมีคำอธิบายภาษาไทย ใช้วันที่กับ `WA.player.name` เป็น seed วันเดียวกันได้ผลเดิม |

popup ทุกตัวต้องมี object anchor ใน `floorLayer` ชื่อตรงกับที่ `openPopup` เรียก

## 5. ขนาด tile

ตรวจขนาด tile จริงของ pack หลังผู้ใช้วางไฟล์:
- 32x32: ใช้ได้เลย
- 16x16: ขยาย 2x แบบ nearest-neighbor ภาพคมเท่าเดิม
- 48x48: ย่อเป็น 32x32 ภาพ pixel art จะเบลอบ้าง ต้องสร้างภาพตัวอย่างให้ผู้ใช้ดูและอนุมัติก่อนทำต่อ

การแปลงขนาดทำด้วย Pillow และเก็บไฟล์ที่แปลงแล้วแยกจากไฟล์ต้นฉบับ

### ของที่ pack อาจไม่มี
`draw_custom.py` วาด sprite สำรองไว้เสมอสำหรับ แมว, ชิบะ, ปลาคาร์ฟ, อาหารปลา, กระดานโกะ, วิทยุ, ตู้ arcade และโต๊ะปิงปอง
`tiles.py` ใช้ sprite สำรองเป็นค่าตั้งต้น แล้วเขียนทับด้วยของจาก pack เมื่อ pack มีของที่ดีกว่า
ของอื่นที่ pack ไม่มี ให้จัดการตามลำดับนี้:
1. ใช้ของที่ใกล้เคียงที่สุดใน pack
2. ให้ผู้ใช้เลือกว่าจะซื้อ pack อื่นของ GuttyKreum เพิ่ม หรือให้วาดเพิ่มใน `draw_custom.py`

## 6. Validation และการทดสอบ

`build_map.py` ต้องหยุดพร้อม error ที่ชัดเจนเมื่อเจอกรณีต่อไปนี้:
1. ชื่อ tile หรือ object ใน `layout.py` ไม่มีใน `tiles.py` หรือ layer ไม่รู้จัก
2. โซนหรือ object อยู่นอกขอบ map
3. จุดเกิดทับ tile ที่ collide
4. BFS จากจุดเกิดเดินไปไม่ถึงทุกโซน (เช่น ห้องที่ถูกผนังปิดตาย)
5. area, popup หรือ layer ที่ `src/interactions.ts` อ้างถึงไม่มีอยู่ใน map
6. ขนาดภาพ tileset ไม่ตรงกับที่ประกาศใน `tiles.py`

ขั้นตอนตรวจหลัง build:
1. `npm run dev` เปิด preview ใน browser เดินทุกโซน capture หน้าจอให้ผู้ใช้ดู
2. ทดสอบ interaction ทุกตัว, Jitsi ทั้ง 2 ห้อง, silent zone และ `openWebsite` ทุกจุด
3. `npm run build` ให้ wa-map-optimizer ตรวจโครงสร้าง
4. `npm run upload` ไปที่ `https://wa.alternatepro.com/map-storage/` แล้วตั้ง `START_ROOM_URL` ใน `.env` ของ server เป็น URL ของ map ใหม่ แล้วรัน `docker compose up -d`

## 7. สิ่งที่ผู้ใช้ต้องเตรียม

- ซื้อ pack และ unzip ไว้ที่ `maps/cozy-office/tilesets/guttykreum/`
- อ่านเงื่อนไขในไฟล์ license ของ pack ว่าอนุญาตให้ใช้งานแบบนี้
- เครื่องมี Node 18 ขึ้นไป และ Python 3 + Pillow
- ตั้ง `MAP_STORAGE_API_TOKEN` ใน `.env` ของ server เป็นค่าสุ่มยาว สำหรับ `npm run upload`

## 8. ความเสี่ยง

| ความเสี่ยง | การรับมือ |
|---|---|
| tile ไม่ใช่ 32x32 ภาพเบลอหลังย่อ | สร้างภาพตัวอย่างให้ผู้ใช้อนุมัติก่อน (ข้อ 5) |
| pack ไม่มีของบางชิ้น | ทำตามลำดับในข้อ 5 |
| เว็บใน Play corner ไม่ยอมแสดงใน iframe | ทดสอบก่อน ใส่เฉพาะเว็บที่ใช้ได้ |
| upload ทับ map เดิม ของที่วางด้วย map editor หาย | backup volume `map-storage-data` ก่อน upload ทับ |
