# SDS Management — Google Apps Script + GitHub Pages

สำเนาสำหรับ **บริษัท เทพวัฒนา จำกัด** — ข้อมูลแยกจากระบบเดิมทั้งหมด (Google Sheet, Drive folder, Apps Script และ browser storage ของตัวเอง)

- เว็บไซต์: https://watanathep8-dotcom.github.io/sds-management-thepwatana/
- Repository: https://github.com/watanathep8-dotcom/sds-management-thepwatana
- ข้อมูลใน browser (localStorage / IndexedDB) ใช้ key ที่ขึ้นต้นด้วย `thep_` เพื่อไม่ปะปนกับเว็บ SDS เดิมที่อยู่บนโดเมน `watanathep8-dotcom.github.io` เดียวกัน

ชุดนี้แปลงแดชบอร์ด `SDS.html` ให้ใช้ Google Sheet เป็นฐานข้อมูลกลาง และใช้ Google Drive เก็บไฟล์ PDF ของ SDS ส่วนหน้าเว็บสาธารณะอยู่ใน `docs/` สำหรับเผยแพร่ด้วย GitHub Pages

## โครงสร้าง

- `apps-script/Code.gs` — Google Apps Script backend, Google Sheet, Google Drive และ API
- `apps-script/Index.html` — หน้า admin สำหรับเพิ่ม/แก้ไข/ลบข้อมูลผ่าน Apps Script
- `appsscript.json` — manifest ของ Apps Script
- `docs/index.html` — หน้าเว็บสำหรับ GitHub Pages
- `docs/config.js` — URL ของ Apps Script API

ระบบเริ่มต้นแบบว่าง (ไม่มีรายการ SDS ติดมากับหน้าเว็บ) รายการทั้งหมดมาจาก Google Sheet ผ่าน Apps Script หรือเพิ่มผ่านฟอร์มในหน้าเว็บ

ตารางหลักแสดง Chemical, CAS No., Flash Point, Emergency Response, SDS ภาษาไทย, SDS ภาษาอังกฤษ, Edit และ Delete โดยมีตัวค้นหาและตัวกรอง Hazard ส่วนข้อมูลเดิมที่มีฟิลด์เพิ่มเติมยังรองรับเพื่อความเข้ากันได้กับ Google Sheet เดิม

เมื่อเลือกไฟล์ PDF ในฟอร์ม ระบบจะอ่านข้อความจากทุกหน้าด้วย PDF.js และเติมชื่อสาร, CAS No., Hazard, Supplier, Flash Point, Emergency Response และรายการภาษาที่พบให้ โดยรองรับชื่อหัวข้อหลายรูปแบบ, CAS ที่มี/ไม่มีป้ายกำกับ, GHS/คำอธิบายอันตราย และข้อมูล SDS สองภาษา ส่วน PDF ที่เป็นภาพสแกนหรือไม่มี text layer จะแจ้งให้ตรวจสอบและกรอกชื่อสารหรือ CAS เองได้

## 1) ตั้งค่า Google Apps Script

1. เปิด [script.google.com](https://script.google.com) แล้วสร้างโปรเจกต์ใหม่
2. สร้างไฟล์ `Code.gs` และ `Index.html` แล้วคัดลอกไฟล์จากโฟลเดอร์ `apps-script/` ไปวาง
3. เปิด Project Settings แล้วเปิด `Show "appsscript.json" manifest file in editor` จากนั้นแทนที่ manifest ด้วยไฟล์ `appsscript.json` ในชุดนี้
4. เลือกฟังก์ชัน `setupSystem` แล้วกด Run หนึ่งครั้ง จากนั้นอนุญาตสิทธิ์ Google Sheets และ Google Drive
5. ถ้าต้องการกำหนดผู้ดูแลหลายคน ให้ไปที่ Project Settings > Script properties แล้วตั้งค่า `ADMIN_EMAILS` เป็นอีเมลคั่นด้วย comma เช่น `name@example.com,admin@example.com`
   - ตั้ง Script property `SDS_ACTION_PASSWORD` (รหัสผ่านสำหรับแก้ไข/ลบรายการ SDS และบันทึกการตั้งค่า) — ห้ามใส่รหัสผ่านไว้ในโค้ด
   - (แนะนำ) Run ฟังก์ชัน `installKeepWarmTrigger` หนึ่งครั้ง เพื่อสร้าง time trigger ทุก 10 นาทีที่อุ่น cache ของข้อมูล (ยกเลิกได้ด้วย `removeKeepWarmTrigger`)
6. สำหรับการใช้งานจริง ให้สร้าง deployment 2 ตัวจากโปรเจกต์เดียวกัน:
   - `Public API`: Execute as เจ้าของสคริปต์, Who has access = Anyone เพื่อให้หน้า GitHub โหลดข้อมูลได้
   - `Admin`: Execute as ผู้ใช้ที่เปิดเว็บ หรือจำกัดเฉพาะบัญชี/โดเมนของผู้ดูแล เพื่อใช้เพิ่ม/แก้ไข/ลบข้อมูล
7. คัดลอก URL ที่ลงท้ายด้วย `/exec` ของ `Public API` ไปใส่ใน `apiUrl` และ URL ของ `Admin` ไปใส่ใน `adminUrl`

หน้า Apps Script จะเป็นหน้าสำหรับผู้ดูแลระบบ ส่วนการอ่านข้อมูลสาธารณะใช้ Public API โดยเติม `action=api` ซึ่งหน้า GitHub จะเรียกให้อัตโนมัติ ฟังก์ชันเขียนข้อมูลตรวจสอบ `ADMIN_EMAILS` และไม่ควรเปิดให้ผู้ใช้นิรนามใช้งาน

ปุ่ม `ตั้งค่าความกว้าง / Column widths` จะบันทึกค่าความกว้างตารางไว้ใน Script Properties ผ่าน `action=saveColumnWidths` และหน้าเว็บจะอ่านค่ากลางผ่าน `action=columnWidths` ดังนั้นผู้ใช้ทุกคนจะเห็นความกว้างเดียวกัน ไม่ขึ้นกับ `localStorage` ของเครื่องใดเครื่องหนึ่ง หลังแก้ `apps-script/Code.gs` ต้องอัปเดต deployment ของ Apps Script ให้ใช้เวอร์ชันใหม่ด้วย

การเปิด PDF จาก Google Drive ใช้ฟังก์ชัน `getSdsFile()` ส่งไฟล์ผ่าน Web app แล้วสร้างตัวแสดง PDF ในหน้าเว็บ ผู้ชมจึงไม่ต้องเปิดหรือ Login Google Drive โดยตรง แต่ Public API ต้องตั้งเป็น `Execute as เจ้าของสคริปต์` และ `Who has access = Anyone` เพื่อให้ Apps Script อ่านไฟล์แทนผู้ชมได้

## 2) เชื่อมต่อหน้า GitHub Pages

แก้ไฟล์ `docs/config.js`:

```js
window.SDS_CONFIG = {
  apiUrl: "https://script.google.com/macros/s/DEPLOYMENT_ID/exec",
  adminUrl: "https://script.google.com/macros/s/DEPLOYMENT_ID/exec",
  publicReadOnly: true
};
```

เมื่อ `publicReadOnly: true` คนทั่วไปจะดู ค้นหา กรอง และ export CSV ได้ แต่จะไม่เห็นปุ่มเพิ่ม/แก้ไข/ลบ ข้อมูลที่บันทึกจะมาจาก Google Sheet แบบกลาง

## 3) นำขึ้น GitHub

สร้าง repository แบบ Public แล้วรันคำสั่งจากโฟลเดอร์ชุดนี้:

```powershell
git init
git add .
git commit -m "Build SDS management website"
git branch -M main
git remote add origin https://github.com/watanathep8-dotcom/sds-management-thepwatana.git
git push -u origin main
```

จากนั้นเปิด GitHub repository > Settings > Pages:

- Source: `Deploy from a branch`
- Branch: `main`
- Folder: `/docs`

URL ที่คนอื่นใช้เปิดจะเป็น `https://watanathep8-dotcom.github.io/sds-management-thepwatana/`

## หมายเหตุด้านสิทธิ์และข้อมูล

- หน้า GitHub เป็นโหมดอ่านอย่างเดียวโดยตั้งใจ เพื่อไม่ให้ผู้ชมทั่วไปลบหรือแก้ข้อมูลใน Sheet
- การเพิ่ม/แก้ไข/ลบจากหน้า Apps Script จะตรวจสอบอีเมลใน `ADMIN_EMAILS`
- PDF ที่อัปโหลดจะถูกอ่านผ่าน Apps Script Web app เพื่อไม่บังคับให้ผู้ชม Login Google Drive โดยตรง การตั้ง `Anyone with the link / Viewer` ยังทำไว้เป็นสิทธิ์สำรอง แต่ไม่ใช่ช่องทางหลักในการแสดง PDF
- อย่าใส่ข้อมูล SDS ที่เป็นความลับหรือข้อมูลส่วนบุคคลใน deployment ที่เปิดสาธารณะ
- GitHub Pages ไม่สามารถรัน `google.script.run` ได้โดยตรง จึงใช้ Apps Script เป็น backend/API แยกจากหน้าเว็บ GitHub

คอลัมน์ QR Code จะสร้างลิงก์สาธารณะพร้อมข้อมูลสรุปของแต่ละรายการไว้ใน QR ทำให้สแกนจากมือถือเครื่องอื่นแล้วเปิดข้อมูลได้แม้ไม่มี localStorage เดิม และสามารถสั่งพิมพ์ป้าย QR ได้จากปุ่มในตาราง
