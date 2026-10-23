# OPD Med CNT — โปรแกรมจัดตารางเวร OPD อายุรกรรม รพ.ชัยนาทนเรนทร

> **ใช้งานจริงกับแพทย์ทั้งแผนก** — ตารางที่ประกาศไปแล้วต้องไม่เปลี่ยน/ไม่หาย ทุกการแก้ต้องคิดถึงข้อนี้ก่อน

## ภาพรวม

- **ไฟล์เดียว `index.html`** (~2,750 บรรทัด · HTML + CSS + JS ในไฟล์เดียว) ไม่มี build ไม่มี npm ไม่มีเทสต์อัตโนมัติ
- **Deploy:** GitHub Pages จาก branch `main` → https://osamunae-champ.github.io/opd-med-cnt/
  **push ขึ้น `main` = ขึ้นเว็บจริงทันที (~1 นาที)**
- **ไลบรารีจาก CDN:** Firebase compat SDK 9.23.0 (app + firestore) · SheetJS `xlsx` 0.18.5 · ExcelJS 4.4.0 (โหลดเมื่อกด Export ตาราง OPD เท่านั้น — SheetJS รุ่นฟรีเขียนสี/เส้นไม่ได้) · Font Awesome 6.4.0 · Google Fonts (Sarabun, Chakra Petch)
- ประวัติ commit เดิมเป็น "Delete index.html / Add files via upload" (เจ้าของอัปโหลดผ่านหน้าเว็บ GitHub) → **`git pull` ก่อนแก้ทุกครั้ง**

## สถาปัตยกรรมข้อมูล

```
localStorage (ของแต่ละเครื่อง)  ⇄  Firestore doc  opd-scheduler/settings  (เอกสารเดียว เก็บทุกอย่าง)
```

### คีย์ข้อมูล (ทั้งใน localStorage และเป็น field ใน Firestore — ค่าเป็น JSON string)

| คีย์ | เนื้อหา | ค่าเริ่มต้น |
|---|---|---|
| `opd_ward_v11` | `wardData` = `{monthColumns:[...30 เดือน], doctors:[{seq,name,isNP,schedule:{"กรกฎาคม 2569":{h1,h2}}}]}` — Loop วอร์ด ก.ค. 2569–ธ.ค. 2571, 16 คน | `DEFAULT_WARD` (บรรทัดยาว ~32KB) |
| `opd_clinic_v11` | `clinicSchedule[week 1–5][dow 1–5] = [ชื่อ]` คลินิกพิเศษ | `DEFAULT_CLINIC` |
| `opd_hol_v11` | `holidays["YYYY-MM-DD" (ค.ศ.)] = ชื่อวันหยุด` | `{}` (โหลดจาก `THAI_HOLIDAYS[ปี พ.ศ.]`) |
| `opd_weight_v11` | `clinicWeightMap[ชื่อ] = ตัวคูณ` (ไม่มีในแมพ = 1.5) | `DEFAULT_WEIGHT` |
| `opd_hide_weight_v11` | ซ่อน badge ตัวคูณ | `false` |
| `opd_sched_<ปีพ.ศ.>_<mIdx 0–11>` | **ตาราง OPD ที่สร้าง/แก้แล้วของแต่ละเดือน** `{sched:[...], meta:{mIdx,yearBE,perDay}}` | — |
| `opd_sched_index` | รายชื่อคีย์ `opd_sched_*` ทั้งหมด | — |
| `_ts`, `_writeSecret` | เฉพาะใน Firestore: เวลา push และ secret ที่ (น่าจะ) ถูกตรวจใน Firestore rules | — |

แถวของ `sched`: `{date:"YYYY-MM-DD", dow, isHol, holName, half:'h1'|'h2', assigned:[ชื่อ...], clinicDocs:[ชื่อ...]}` (เฉพาะวันจันทร์–ศุกร์)

### การ sync (บรรทัด ~13–103, สคริปต์แรกใน `<head>`)

- `fbPush()` — **Admin เท่านั้น** · อ่าน BASE_KEYS + ทุกคีย์ใน `opd_sched_index` จาก localStorage แล้ว **`DOC.set(payload)` เขียนทับทั้งเอกสาร**
  ถูกเรียกจาก `markSaved()` แบบ debounce 1.5 วินาที — ทุก `save*()` จึง push ขึ้น cloud อัตโนมัติ
- `fbPull()` — ตอน `DOMContentLoaded` และปุ่ม Pull · คัดค่าจาก cloud ทับ localStorage ถ้าต่าง → `location.reload()`
- `DOC.onSnapshot` — ฟัง realtime · ถ้า `_ts` ใหม่กว่าเดิม >2 วิ และค่าต่าง → เขียน localStorage แล้ว reload
- pull/snapshot **เพิ่ม/ทับคีย์อย่างเดียว ไม่ลบ** · cloud เป็นแหล่งจริง (local ที่ต่างจาก cloud จะถูกทับตอนเปิดหน้า)

## สิทธิ์ (บรรทัด ~268–476)

- `_PW_ADMIN = 'cnt2569'` → `isAdmin = true` · `_PW_STAFF = 'cntstaff'` → ดูอย่างเดียว · จำใน `sessionStorage.opd_auth`
- **เป็นแค่การซ่อน UI ฝั่ง client** — รหัสผ่านและ `_WRITE_SECRET` อยู่ในซอร์สที่ใครก็เปิดดูได้ ไม่ใช่ความปลอดภัยจริง
- `applyAuthMode()` ซ่อน/แสดง `.admin-only` / `.staff-only` · ฟังก์ชันเขียนทุกตัวเช็ค `isAdmin` ซ้ำอีกชั้น (`save*`, `markSaved`, `fbPush`, `saveOPDSched`, `editOPDCell`)
- ⚠ ปุ่มหลายปุ่มเขียน `class="btn ..." class="admin-only"` (attribute ซ้ำ → `admin-only` ไม่มีผล) แต่ส่วนใหญ่อยู่ในแท็บที่ staff เข้าไม่ได้

## แท็บ

| Admin | Staff |
|---|---|
| Loop วอร์ด (ทั้งสองสิทธิ์เห็น) · แก้ชื่อแพทย์ · คลินิกพิเศษ · ตาราง OPD · สรุปภาระงาน · วันหยุด · ตัวคูณคลินิก · Import นับภาระ (Excel) | Loop วอร์ด · ตาราง OPD (`renderStaffOPD`) · สรุป OPD รายแพทย์ (`renderStaffSummary`) |

## ฟังก์ชันหลัก

| ฟังก์ชัน | หน้าที่ |
|---|---|
| `loadState()` / `loadWeights()` | อ่าน localStorage (หรือค่า default) ตอนเริ่ม |
| `getOPDPool(mIdx, yearBE)` | แพทย์ที่ Loop วอร์ดครึ่งนั้นขึ้นต้นด้วย `OPD` = อยู่ใน pool ของ h1 (1–15) / h2 (16–สิ้นเดือน) |
| `getClinicDocs(dateStr)` | คลินิกพิเศษของวันนั้น · สัปดาห์ = `ceil(วันที่/7)` (1–7=W1 … 29+=W5) **ไม่ใช่สัปดาห์ปฏิทิน** |
| **`buildSchedule(mIdx, yearBE, perDay, seed)`** | อัลกอริทึมจัด OPD ("CORE SCHEDULER v7") — ข้ามเสาร์-อาทิตย์/วันหยุด · คนติดคลินิกวันนั้นไม่ถูกจัด · แยกกลุ่มมี/ไม่มีคลินิกในครึ่งนั้น · gap rule ให้ OPD คนไม่มีคลินิก มากกว่าคนมีคลินิกสูงสุด 2–3 วัน · tie-break ด้วย xorshift RNG จาก seed (seed เดิม = ผลเดิม ถ้าข้อมูลนำเข้าเหมือนเดิม) |
| `genOPD()` | สร้างตารางเดือนที่เลือก (seed 1) → `saveOPDSched()` |
| `reshuffleOPD()` | จัดใหม่ด้วย seed ใหม่ (มี confirm) |
| `renderOPDTable` / `editOPDCell` / `commitOPDEdit` / `addOPDSlot` | แสดงและแก้ช่องแพทย์ด้วยมือ (popup เรียงตาม pool → ว่าง → ภาระน้อย) → `saveOPDSched()` |
| `saveOPDSched()` / `loadSchedForMonth()` / `onMonthYearChange()` | เก็บ/โหลดตารางรายเดือนที่คีย์ `opd_sched_<ปี>_<เดือน>` |
| `renderSummary(...)` | สรุปภาระ: OPD + คลินิก×ตัวคูณ · `checkRules` เตือนเมื่อผิดกฎ gap 2–3 |
| `saveWard` / `saveClinic` / `saveHol` / `saveWeights` → `markSaved()` → `fbPush()` | ทุกการบันทึกขึ้น cloud |
| `openWardSwap` / `doWardSwap` / `resetWardHalf` | Admin กดช่องในตาราง Loop วอร์ด → แลกตำแหน่งกับแพทย์อีกคนในเดือน/ครึ่งเดียวกัน (ตำแหน่งครบเสมอ) · ช่องที่ต่างจาก `DEFAULT_WARD` ไฮไลต์ (`wardPatternCell`) · คืนค่า pattern ทั้งครึ่งได้ · บันทึกด้วย `saveWard()` → การจัด OPD ใช้ loop ใหม่ผ่าน `getOPDPool()` · ตาราง OPD ที่บันทึกแล้วไม่เปลี่ยนเอง ต้องสร้างใหม่ |
| `saveDoctorNames()` | เปลี่ยนชื่อใน wardData + clinicSchedule (ไม่แก้ชื่อในตาราง OPD ที่บันทึกไว้แล้ว) |
| `exportWard/Clinic/Summary/ImportResult` | Excel ผ่าน SheetJS |
| `exportOPD()` | Excel รูปแบบประกาศ: วัน · วันที่ · แพทย์เวร (เว้นว่าง — ระบบไม่มีข้อมูลเวร) · OPD 1–n · Med F1 / Med F2 / Med F3 / SU / ICU (เว้นว่าง) · ทุกวันในเดือน · เสาร์-อาทิตย์/วันหยุด แถวเทา · ผ่าน ExcelJS |
| `exportSettings` / `importSettings` | JSON ของ ward/clinic/holidays/weights (ไม่รวมตาราง OPD) |
| `handleImport` ... | แท็บ Import: อ่าน Excel นับชื่อในคอลัมน์ OPD 1–4 (ไม่แตะข้อมูลระบบ) |

## 🔴 จุดที่ห้ามแตะ (เว้นเจ้าของสั่งชัด ๆ และมีแผนย้ายข้อมูล)

1. **ชื่อคีย์ทั้งหมด** (`opd_*_v11`, `opd_sched_<ปี>_<เดือน>`, `opd_sched_index`) และ **path Firestore `opd-scheduler/settings`** — เปลี่ยน = ข้อมูลที่ประกาศแล้วหายจากหน้าจอทุกเครื่อง
2. **รูปแบบข้อมูลของ `sched` / `wardData` / `clinicSchedule` / `holidays`** — ของเก่าใน cloud ต้องยังอ่านได้ (เพิ่ม field ใหม่ได้ แต่ต้องรับมือกรณีไม่มี field นั้น)
3. **`buildSchedule()` และ `getClinicDocs()` / `getOPDPool()`** — ระวังเป็นพิเศษ: ฝั่ง Staff คำนวณตารางใหม่สด ๆ จากฟังก์ชันเหล่านี้ (ดูปัญหาข้อ 1 ด้านล่าง) แก้อัลกอริทึม = ตารางที่ staff เห็นของเดือนที่ประกาศไปแล้วเปลี่ยนทันที
4. **`firebaseConfig`, `_WRITE_SECRET`** — ต้องตรงกับโปรเจกต์ Firebase `opd-med-cnt` และ Firestore rules (rules ไม่อยู่ใน repo)
5. **`fbPush` ใช้ `DOC.set` เขียนทับทั้งเอกสาร** — ห้ามเพิ่มทางเขียน cloud ที่ส่งข้อมูลไม่ครบ ไม่งั้นเดือนอื่นหาย
6. **`DEFAULT_WARD` / `DEFAULT_CLINIC`** — ใช้เมื่อ localStorage ว่างเท่านั้น แต่ `resetToDefault()` จะ push ค่า default ทับ cloud
7. **รหัสผ่าน** — เปลี่ยนได้เมื่อเจ้าของสั่ง แต่ต้องแจ้งแพทย์ทั้งแผนก

## ⚠ ปัญหาที่พบตอนอ่านโค้ด (ยังไม่แก้ — รอเจ้าของตัดสิน)

1. **Staff ไม่ได้เห็นตารางที่ Admin บันทึกไว้** — `renderStaffOPD()` / `renderStaffSummary()` เรียก `buildSchedule(mIdx, yearBE, 4, 1)` ใหม่ทุกครั้ง ไม่อ่าน `opd_sched_<ปี>_<เดือน>`
   → การแก้ด้วยมือ, "จัดใหม่" (seed อื่น), perDay ≠ 4 และการเปลี่ยนวันหยุด/คลินิก/ชื่อ/Loop วอร์ดหลังประกาศ ทำให้ตาราง staff ต่างจากที่ admin เห็น
2. **`genOPD()` ทับตารางเดือนที่บันทึกไว้แล้วโดยไม่ถาม** (ต่างจาก `reshuffleOPD()` ที่มี confirm) — การแก้ด้วยมือหายทันทีและ push ขึ้น cloud
3. ปุ่ม "เพิ่มแถว" เรียก `addOPDRow()` ซึ่งไม่มีอยู่ → error ใน console
4. `importSettings()` เขียน localStorage แล้ว reload แต่ไม่ push → `fbPull()` ตอนเปิดหน้าจะดึงค่า cloud กลับมาทับ (Import ใช้ไม่ได้จริงเมื่อ cloud มีข้อมูล)
5. `commitOPDEdit()` ใช้เดือน/ปีจาก selector ไม่ใช่ `_lastOPDMeta` (สรุปผิดเดือนได้ถ้าเปลี่ยน selector ระหว่างแก้)
6. Admin สองเครื่องแก้พร้อมกัน → push ทีหลังชนะทั้งเอกสาร (last-write-wins)
7. `getClinicDocs` ใช้ `new Date("YYYY-MM-DD")` (UTC) แล้ว `getDay()` (local) — ถูกในเวลาไทย แต่เพี้ยนถ้าเครื่องอยู่ timezone ติดลบ
8. แพทย์ที่ไม่อยู่ใน Loop วอร์ด (ตอนนี้ `'ธมนวรรณ'`) อยู่ใน `EXTRA_DOCS` → `allDoctorNames()` ใช้ในหน้าคลินิกพิเศษ ตัวคูณ และแก้ OPD · เพิ่มแพทย์นอก loop = เพิ่มชื่อใน `EXTRA_DOCS`
10. `ALWAYS_OPD` (ตอนนี้ `'ธมนวรรณ'`) ถูกเติมเข้า pool OPD ทั้ง h1/h2 ใน `getOPDPool()` ตั้งแต่ `ALWAYS_OPD_FROM` = พ.ย. 2569 · เดือนก่อนหน้าไม่แตะเพื่อไม่ให้ตารางที่ประกาศแล้วเปลี่ยน · แสดงเป็นแถวในตาราง Loop วอร์ด/สรุป/Export ผ่าน `wardRowsForDisplay()` (แถวสำหรับแสดงผล ไม่อยู่ใน `wardData` · แก้ชื่อในแท็บแก้ชื่อแพทย์ไม่ได้)
11. `NO_OPD` (ตอนนี้ `'นิภาพร'`) ถูกตัดออกจาก pool OPD ใน `getOPDPool()` ตั้งแต่ `NO_OPD_FROM` = พ.ย. 2569 · Loop วอร์ดและคลินิกพิเศษของเธอยังเหมือนเดิม (ช่อง OPD ในตาราง Loop ยังแสดงตาม pattern)
9. `THAI_HOLIDAYS` มีเฉพาะ 2568–2571 · ข้อความในแท็บวันหยุดยังเขียนว่า "ครอบคลุมปี 2568"

## Workflow การแก้

1. `git pull` ก่อนเสมอ (เจ้าของอาจอัปโหลดไฟล์ใหม่ผ่านเว็บ GitHub)
2. แก้ให้เล็กที่สุด · ไม่จัดรูปแบบไฟล์ใหม่ทั้งไฟล์ (diff จะอ่านไม่ออก) · ไม่แยกไฟล์/ไม่เพิ่ม build step เว้นเจ้าของสั่ง
3. **ทดสอบในเครื่องก่อน push** — เปิดผ่าน local server เช่น `python -m http.server 8000` แล้วเข้า http://localhost:8000
   - ⚠ **หน้า local ต่อ Firestore จริง** — login Admin แล้วกดบันทึกอะไรก็ตาม = เขียนทับข้อมูลจริงของทั้งแผนก
   - ทดสอบฝั่ง Admin ให้ปิด push ก่อน: ใน DevTools console รัน `window.fbPush = async () => console.log('push blocked')` ก่อน login (onSnapshot อาจ reload หน้า → ต้องรันใหม่) หรือทดสอบด้วยสิทธิ์ Staff
   - เทียบผล `buildSchedule` ของเดือนที่ประกาศแล้ว ก่อน–หลังแก้ ต้องได้ตารางเดิม (ถ้าการแก้ไม่ได้ตั้งใจเปลี่ยนอัลกอริทึม)
   - เช็ค console ไม่มี error · ทดสอบทั้งสองสิทธิ์ · ทดสอบจอมือถือ
4. ก่อนเปลี่ยนอะไรที่กระทบข้อมูลใน cloud: กด **Export** (settings JSON) เก็บไว้ และเก็บสำเนาเอกสาร Firestore (ตาราง OPD ไม่อยู่ใน Export)
5. commit ข้อความภาษาไทย ทีละเรื่อง · `git add index.html` ระบุไฟล์ (ห้าม `git add -A`)
6. รัน `node D:\ai-bridge\prepush-check.mjs` ผ่านแล้วค่อย `git push origin main` → รอ ~1 นาที แล้วเปิดเว็บจริงตรวจ (hard refresh)
7. งานที่แตะอัลกอริทึม/คีย์ข้อมูล/sync = งานเสี่ยง → รีวิวข้ามโมเดลก่อน push
