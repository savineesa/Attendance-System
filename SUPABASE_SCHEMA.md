# Supabase Schema — ระบบเช็คชื่อด้วย QR Code และ GPS

เอกสารนี้ออกแบบจาก SRS ล่าสุด: มีบัญชีเฉพาะอาจารย์ผ่าน Google, นักศึกษาไม่มีบัญชี, QR Code หมุนเวียน และอนุญาตเช็คชื่อเมื่ออยู่ห่างจากเครื่องอาจารย์ไม่เกิน 10 เมตร

## หลักการออกแบบ

- ใช้ `uuid` เป็น Primary Key ของตารางธุรกิจทั้งหมด และใช้ `gen_random_uuid()` เป็นค่าเริ่มต้น
- ใช้ `timestamptz` ทุกจุดที่เกี่ยวกับเวลา เพื่อป้องกันปัญหาเขตเวลา
- เปิดส่วนขยาย `postgis` และบันทึกพิกัดเป็น `geography(Point,4326)` เพื่อคำนวณระยะทางเป็นเมตรด้วย `ST_DWithin`
- บัญชีอาจารย์อยู่ใน `auth.users` ของ Supabase Auth และตาราง `teacher_profiles` เก็บข้อมูลเพิ่มเติม
- ไม่บันทึก QR Code ดิบในฐานข้อมูล แต่บันทึกค่า hash (`token_hash`) เพื่อไม่ให้ token ถูกนำไปใช้ซ้ำหากฐานข้อมูลรั่วไหล

> ก่อนสร้างตาราง ให้เปิด extensions `pgcrypto` และ `postgis` ใน Supabase Dashboard หรือด้วย SQL: `create extension if not exists pgcrypto; create extension if not exists postgis;`

---

## ความสัมพันธ์ของตาราง

```text
auth.users ── 1:1 ── teacher_profiles
teacher_profiles ── 1:N ── courses
courses ── N:N ── students (ผ่าน enrollments)
courses ── 1:N ── attendance_sessions
attendance_sessions ── 1:N ── qr_tokens
attendance_sessions ── 1:N ── attendance_records
students ── 1:N ── attendance_records
```

---

## 1) `teacher_profiles`

ข้อมูลอาจารย์ที่ผูกกับ Google Account ใน `auth.users`

| Column | Type | รายละเอียด |
|---|---|---|
| `id` | `uuid` PK, FK → `auth.users.id` | รหัสผู้ใช้จาก Supabase Auth |
| `email` | `text` unique, not null | อีเมล Google ของอาจารย์ |
| `full_name` | `text` not null | ชื่อ–นามสกุล |
| `avatar_url` | `text` nullable | URL รูปโปรไฟล์ Google |
| `is_active` | `boolean` not null default `true` | สถานะอนุญาตใช้งาน |
| `created_at` | `timestamptz` not null default `now()` | วันที่สร้าง |
| `updated_at` | `timestamptz` not null default `now()` | วันที่แก้ไขล่าสุด |

### ตัวอย่างข้อมูล 5 แถว

| id | email | full_name | is_active |
|---|---|---|---|
| `10000000-...-0001` | somchai@university.ac.th | อาจารย์สมชาย ใจดี | true |
| `10000000-...-0002` | siriporn@university.ac.th | อาจารย์ศิริพร รัตน์ดี | true |
| `10000000-...-0003` | anan@university.ac.th | อาจารย์อนันต์ มั่นคง | true |
| `10000000-...-0004` | malee@university.ac.th | อาจารย์มาลี แสงทอง | true |
| `10000000-...-0005` | wirat@university.ac.th | อาจารย์วีรัตน์ พูนทรัพย์ | false |

---

## 2) `courses`

รายวิชาที่อาจารย์สร้างและจัดการเอง

| Column | Type | รายละเอียด |
|---|---|---|
| `id` | `uuid` PK default `gen_random_uuid()` | รหัสรายวิชาภายในระบบ |
| `teacher_id` | `uuid` not null FK → `teacher_profiles.id` | เจ้าของรายวิชา |
| `course_code` | `varchar(30)` not null | เช่น `CS101` |
| `course_name` | `text` not null | ชื่อรายวิชา |
| `semester` | `smallint` not null check (`semester in (1,2,3)`) | ภาคเรียน |
| `academic_year` | `smallint` not null | ปีการศึกษา เช่น 2569 |
| `is_archived` | `boolean` not null default `false` | ซ่อนรายวิชาที่สิ้นสุดแล้ว |
| `created_at` | `timestamptz` not null default `now()` | วันที่สร้าง |
| `updated_at` | `timestamptz` not null default `now()` | วันที่แก้ไขล่าสุด |

แนะนำ unique constraint: `unique (teacher_id, course_code, semester, academic_year)`

### ตัวอย่างข้อมูล 5 แถว

| id | teacher_id | course_code | course_name | semester | academic_year |
|---|---|---|---|---:|---:|
| `20000000-...-0001` | `...0001` | CS101 | การเขียนโปรแกรมเบื้องต้น | 1 | 2569 |
| `20000000-...-0002` | `...0001` | CS205 | ระบบฐานข้อมูล | 1 | 2569 |
| `20000000-...-0003` | `...0001` | CS310 | วิศวกรรมซอฟต์แวร์ | 1 | 2569 |
| `20000000-...-0004` | `...0002` | IT201 | การพัฒนาเว็บ | 1 | 2569 |
| `20000000-...-0005` | `...0003` | CS450 | ปัญญาประดิษฐ์ | 2 | 2569 |

---

## 3) `students`

ข้อมูลนักศึกษา นักศึกษาไม่ต้องมีบัญชี Supabase Auth

| Column | Type | รายละเอียด |
|---|---|---|
| `id` | `uuid` PK default `gen_random_uuid()` | รหัสภายในระบบ |
| `student_code` | `varchar(30)` unique, not null | รหัสนักศึกษา ใช้ยืนยันตอนเช็คชื่อ |
| `first_name` | `text` not null | ชื่อ |
| `last_name` | `text` not null | นามสกุล |
| `email` | `text` nullable | อีเมล |
| `phone` | `varchar(30)` nullable | เบอร์โทร |
| `faculty` | `text` nullable | คณะ |
| `major` | `text` nullable | สาขา |
| `year_level` | `smallint` nullable check (`year_level between 1 and 8`) | ชั้นปี |
| `created_at` | `timestamptz` not null default `now()` | วันที่สร้าง |
| `updated_at` | `timestamptz` not null default `now()` | วันที่แก้ไขล่าสุด |

### ตัวอย่างข้อมูล 5 แถว

| id | student_code | first_name | last_name | faculty | major | year_level |
|---|---|---|---|---|---|---:|
| `30000000-...-0001` | 65123456 | สมชาย | ใจดี | วิทยาศาสตร์ | วิทยาการคอมพิวเตอร์ | 3 |
| `30000000-...-0002` | 65123457 | สมหญิง | รักเรียน | วิทยาศาสตร์ | วิทยาการคอมพิวเตอร์ | 3 |
| `30000000-...-0003` | 65123458 | กิตติชัย | เก่งดี | วิทยาศาสตร์ | วิทยาการคอมพิวเตอร์ | 3 |
| `30000000-...-0004` | 65123459 | พิมพ์ชนก | สุขใจ | วิทยาศาสตร์ | วิทยาการคอมพิวเตอร์ | 3 |
| `30000000-...-0005` | 65123460 | ณัฐพงศ์ | แสงทอง | วิทยาศาสตร์ | วิทยาการคอมพิวเตอร์ | 3 |

---

## 4) `enrollments`

ตารางเชื่อมรายวิชากับนักศึกษา

| Column | Type | รายละเอียด |
|---|---|---|
| `id` | `uuid` PK default `gen_random_uuid()` | รหัสรายการลงทะเบียน |
| `course_id` | `uuid` not null FK → `courses.id` | รายวิชา |
| `student_id` | `uuid` not null FK → `students.id` | นักศึกษา |
| `enrolled_at` | `timestamptz` not null default `now()` | วันที่เพิ่มเข้ารายวิชา |
| `is_active` | `boolean` not null default `true` | สถานะลงทะเบียน |

แนะนำ unique constraint: `unique (course_id, student_id)`

### ตัวอย่างข้อมูล 5 แถว

| id | course_id | student_id | is_active |
|---|---|---|---|
| `40000000-...-0001` | `...0001` CS101 | `...0001` 65123456 | true |
| `40000000-...-0002` | `...0001` CS101 | `...0002` 65123457 | true |
| `40000000-...-0003` | `...0001` CS101 | `...0003` 65123458 | true |
| `40000000-...-0004` | `...0002` CS205 | `...0001` 65123456 | true |
| `40000000-...-0005` | `...0002` CS205 | `...0004` 65123459 | true |

---

## 5) `attendance_sessions`

หนึ่งแถวต่อหนึ่งรอบเช็คชื่อ พิกัดอาจารย์ ณ เวลาที่เริ่มรอบเป็นจุดอ้างอิง

| Column | Type | รายละเอียด |
|---|---|---|
| `id` | `uuid` PK default `gen_random_uuid()` | รหัสรอบเช็คชื่อ |
| `course_id` | `uuid` not null FK → `courses.id` | รายวิชาที่เปิดรอบ |
| `teacher_id` | `uuid` not null FK → `teacher_profiles.id` | อาจารย์ผู้เปิดรอบ |
| `teacher_location` | `geography(Point,4326)` not null | พิกัด GPS ของอาจารย์ |
| `max_distance_meters` | `numeric(5,2)` not null default `10.00` | ระยะสูงสุดที่อนุญาต |
| `qr_rotation_seconds` | `smallint` not null default `30` | ระยะเวลาหมุน QR Code |
| `starts_at` | `timestamptz` not null default `now()` | เวลาเริ่มรอบ |
| `ends_at` | `timestamptz` nullable | เวลาปิดรอบ |
| `status` | `text` not null check (`status in ('open','closed','cancelled')`) | สถานะรอบ |
| `created_at` | `timestamptz` not null default `now()` | วันที่สร้าง |

### ตัวอย่างข้อมูล 5 แถว

| id | course_id | teacher_id | teacher_location | max_distance_meters | starts_at | status |
|---|---|---|---|---:|---|---|
| `50000000-...-0001` | `...0001` CS101 | `...0001` | `POINT(100.5018 13.7563)` | 10.00 | 2569-06-23 10:00+07 | closed |
| `50000000-...-0002` | `...0002` CS205 | `...0001` | `POINT(100.5019 13.7564)` | 10.00 | 2569-06-23 13:00+07 | open |
| `50000000-...-0003` | `...0003` CS310 | `...0001` | `POINT(100.5020 13.7561)` | 10.00 | 2569-06-20 09:00+07 | closed |
| `50000000-...-0004` | `...0004` IT201 | `...0002` | `POINT(100.5101 13.7452)` | 10.00 | 2569-06-21 09:30+07 | closed |
| `50000000-...-0005` | `...0005` CS450 | `...0003` | `POINT(100.4950 13.7600)` | 10.00 | 2569-06-22 13:30+07 | cancelled |

> รูปแบบ WKT ของ PostGIS คือ `POINT(longitude latitude)` ไม่ใช่ `POINT(latitude longitude)`

---

## 6) `qr_tokens`

เก็บประวัติ QR Code ที่หมุนเวียนในแต่ละรอบ เพื่อใช้ตรวจสอบ token ที่สแกนได้

| Column | Type | รายละเอียด |
|---|---|---|
| `id` | `uuid` PK default `gen_random_uuid()` | รหัสรายการ QR |
| `session_id` | `uuid` not null FK → `attendance_sessions.id` | รอบเช็คชื่อ |
| `token_hash` | `text` unique, not null | SHA-256 hash ของ token จริง |
| `sequence_no` | `integer` not null | ลำดับ QR ที่หมุนเวียน |
| `issued_at` | `timestamptz` not null default `now()` | เวลาเริ่มใช้ |
| `expires_at` | `timestamptz` not null | เวลาหมดอายุ |
| `is_revoked` | `boolean` not null default `false` | ถูกยกเลิกหรือไม่ |

แนะนำ unique constraint: `unique (session_id, sequence_no)`

### ตัวอย่างข้อมูล 5 แถว

| id | session_id | sequence_no | token_hash | issued_at | expires_at |
|---|---|---:|---|---|---|
| `60000000-...-0001` | `...0002` | 1 | `a4f1...89c2` | 2569-06-23 13:00:00+07 | 13:00:30+07 |
| `60000000-...-0002` | `...0002` | 2 | `b9d0...123e` | 2569-06-23 13:00:30+07 | 13:01:00+07 |
| `60000000-...-0003` | `...0002` | 3 | `c372...ee10` | 2569-06-23 13:01:00+07 | 13:01:30+07 |
| `60000000-...-0004` | `...0001` | 1 | `d8a2...65f1` | 2569-06-23 10:00:00+07 | 10:00:30+07 |
| `60000000-...-0005` | `...0003` | 1 | `e5bc...3a77` | 2569-06-20 09:00:00+07 | 09:00:30+07 |

---

## 7) `attendance_records`

ผลการเช็คชื่อของนักศึกษา หนึ่งนักศึกษามีได้ไม่เกินหนึ่งแถวต่อหนึ่งรอบเช็คชื่อ

| Column | Type | รายละเอียด |
|---|---|---|
| `id` | `uuid` PK default `gen_random_uuid()` | รหัสผลเช็คชื่อ |
| `session_id` | `uuid` not null FK → `attendance_sessions.id` | รอบเช็คชื่อ |
| `student_id` | `uuid` not null FK → `students.id` | นักศึกษา |
| `status` | `text` not null check (`status in ('present','late','absent','leave')`) | สถานะเข้าเรียน |
| `recorded_by` | `text` not null check (`recorded_by in ('qr','teacher_manual')`) | QR Code หรืออาจารย์บันทึกเอง |
| `student_location` | `geography(Point,4326)` nullable | พิกัดนักศึกษา เมื่อใช้ QR |
| `distance_meters` | `numeric(8,2)` nullable | ระยะจากอาจารย์ หน่วยเมตร |
| `location_verified` | `boolean` not null default `false` | ผ่านเงื่อนไข GPS หรือไม่ |
| `checked_in_at` | `timestamptz` not null default `now()` | เวลาเช็คชื่อ/บันทึก |
| `note` | `text` nullable | หมายเหตุจากอาจารย์ |
| `updated_by` | `uuid` nullable FK → `teacher_profiles.id` | อาจารย์ที่แก้ไขล่าสุด |
| `updated_at` | `timestamptz` not null default `now()` | เวลาแก้ไขล่าสุด |

แนะนำ unique constraint: `unique (session_id, student_id)`

### ตัวอย่างข้อมูล 5 แถว

| id | session_id | student_id | status | recorded_by | distance_meters | location_verified | checked_in_at |
|---|---|---|---|---|---:|---|---|
| `70000000-...-0001` | `...0001` | `...0001` 65123456 | present | qr | 4.20 | true | 2569-06-23 10:05+07 |
| `70000000-...-0002` | `...0001` | `...0002` 65123457 | present | qr | 6.80 | true | 2569-06-23 10:06+07 |
| `70000000-...-0003` | `...0001` | `...0003` 65123458 | present | teacher_manual | null | false | 2569-06-23 10:15+07 |
| `70000000-...-0004` | `...0002` | `...0001` 65123456 | present | qr | 3.10 | true | 2569-06-23 13:04+07 |
| `70000000-...-0005` | `...0003` | `...0004` 65123459 | absent | teacher_manual | null | false | 2569-06-20 10:30+07 |

---

# คำแนะนำ: หน้าเว็บควรอ่าน/เขียนข้อมูลอย่างไร

## 1. เข้าสู่ระบบด้วย Google

- **อ่าน:** `teacher_profiles` ด้วย `auth.uid()` เพื่อตรวจ `is_active`
- **เขียน:** ให้ trigger สร้าง/อัปเดต `teacher_profiles` หลัง Google Sign-In ครั้งแรก ไม่ควรให้ Frontend เขียน `teacher_id` เอง
- **ผลลัพธ์:** หาก `is_active = false` ให้แสดงหน้า “ยังไม่ได้รับสิทธิ์ใช้งาน”

## 2. หน้า Dashboard

- **อ่าน:** รายวิชาใน `courses` ที่ `teacher_id = auth.uid()` และ `is_archived = false`
- **อ่าน:** นับจำนวนนักศึกษาผ่าน `enrollments`, จำนวนรอบวันนี้จาก `attendance_sessions`, และสถิติจาก `attendance_records`
- **อ่านแบบ realtime:** subscribe ตาราง `attendance_records` เฉพาะ session ที่กำลังเปิด เพื่อแสดงจำนวนเช็คชื่อสดและแจ้งเตือน
- **ไม่ควรเขียน:** หน้านี้เป็นหน้าสรุปข้อมูลเป็นหลัก

## 3. จัดการรายวิชาและนักศึกษา

- **อ่าน:** `courses`, `enrollments` และ `students` เฉพาะรายวิชาของอาจารย์คนปัจจุบัน
- **เขียน:** เพิ่ม/แก้ไข `courses`; เพิ่มข้อมูล `students` และเพิ่มความสัมพันธ์ใน `enrollments`; เมื่อลบนักศึกษาจากรายวิชาให้ตั้ง `enrollments.is_active = false` แทนการลบประวัติ
- **ข้อควรระวัง:** ป้องกันการเพิ่ม `student_code` ซ้ำ และไม่ควรลบนักศึกษาจริงหากมีประวัติเช็คชื่อแล้ว

## 4. เริ่มรอบเช็คชื่อ

- **อ่าน:** ตรวจว่าอาจารย์เป็นเจ้าของ `course_id` และไม่มี `attendance_sessions.status = 'open'` ในรายวิชาเดียวกัน
- **เขียน:** Frontend ขอ GPS จาก browser แล้วส่งพิกัดไปสร้าง `attendance_sessions` ด้วย `teacher_location`, `max_distance_meters = 10`, `qr_rotation_seconds`
- **เขียน:** สร้าง QR token แรกใน `qr_tokens` โดยส่งเฉพาะ token จริงให้หน้าจอ QR และบันทึกเฉพาะ SHA-256 hash ในฐานข้อมูล
- **ควรทำผ่าน Edge Function:** ไม่ควร insert session และ QR token จาก Browser โดยตรง เพราะต้องตรวจสิทธิ์และสร้าง token ที่ปลอดภัย

## 5. หน้าจอแสดง QR Code ของอาจารย์

- **อ่าน:** session ที่มีสถานะ `open`, QR token ล่าสุด และ `attendance_records` ของ session นั้น
- **เขียน:** Edge Function สร้าง QR token ใหม่เมื่อ token เดิมหมดอายุ ทุก 30 วินาที หรือเรียกจาก server scheduler
- **อ่านแบบ realtime:** subscribe `attendance_records` เพื่อเพิ่มรายชื่อผู้เช็คชื่อทันที
- **เขียน:** เมื่อกดปิดรอบ อัปเดต `attendance_sessions.status = 'closed'` และ `ends_at = now()`

## 6. หน้ายืนยันเช็คชื่อของนักศึกษา

- **อ่าน:** ไม่ควรเปิดสิทธิ์อ่านตารางทั้งหมดให้ public
- **เขียน:** ส่ง `raw_qr_token`, `student_code`, latitude และ longitude ไปยัง Edge Function `check-in`
- **Edge Function ต้องตรวจสอบ:**
  1. hash ของ token ตรงกับ `qr_tokens` และยังไม่หมดอายุ
  2. session ยังมีสถานะ `open`
  3. นักศึกษามีอยู่จริงและลงทะเบียนในรายวิชานั้น
  4. นักศึกษายังไม่มี `attendance_records` ใน session นั้น
  5. ใช้ `ST_DWithin(teacher_location, student_location, max_distance_meters)` ตรวจระยะไม่เกิน 10 เมตร
  6. หากผ่าน ให้ insert `attendance_records` ด้วย `status = 'present'`, `recorded_by = 'qr'`, `location_verified = true`
- **ผลตอบกลับ:** ให้ Edge Function ส่งเพียงผลสำเร็จ ชื่อรายวิชา สถานะ และระยะห่าง ไม่ส่งข้อมูลนักศึกษาคนอื่น

## 7. สรุปรอบเช็คชื่อและบันทึกด้วยตนเอง

- **อ่าน:** รายชื่อนักศึกษาจาก `enrollments` join `students` พร้อม left join `attendance_records`
- **เขียน:** insert หรือ update `attendance_records` ด้วย `recorded_by = 'teacher_manual'`, `updated_by = auth.uid()`
- **ข้อควรระวัง:** อาจารย์แก้ไขได้เฉพาะ session ของรายวิชาที่ตนเป็นเจ้าของ

## 8. รายงานและแดชบอร์ดสถิติ

- **อ่าน:** join `courses`, `enrollments`, `students`, `attendance_sessions`, `attendance_records`
- **แนะนำ:** สร้าง SQL View เช่น `course_attendance_summary` เพื่อคืนค่า `present_count`, `absence_count`, `total_sessions`, `attendance_percentage` ต่อคนต่อรายวิชา
- **ส่งออก:** Frontend อ่านเฉพาะข้อมูลที่คัดกรองแล้วและสร้าง Excel/PDF ฝั่ง client หรือเรียก Edge Function สำหรับรายงานขนาดใหญ่

---

# Row Level Security (RLS) ที่แนะนำ

1. เปิด RLS ทุกตารางใน `public`
2. อาจารย์อ่านและเขียนได้เฉพาะ `courses` ที่ `teacher_id = auth.uid()`
3. ตาราง `enrollments`, `attendance_sessions`, `qr_tokens`, `attendance_records` ต้องตรวจความเป็นเจ้าของผ่าน `courses.teacher_id = auth.uid()`
4. นักศึกษาแบบไม่ล็อกอิน **ห้าม** ได้รับสิทธิ์ `select` หรือ `insert` บนตารางโดยตรง
5. ให้ Edge Function ใช้ service role ในการตรวจ QR Code, ตรวจ GPS และบันทึกเช็คชื่อแทน
6. จำกัด response ของ Edge Function เพื่อไม่ให้เปิดเผยข้อมูลส่วนบุคคลหรือพิกัดของอาจารย์

> ข้อมูล GPS เป็นข้อมูลส่วนบุคคล ควรกำหนด retention policy เช่น เก็บพิกัดละเอียดไว้ 90 วัน แล้วลบหรือทำให้เป็นข้อมูลสรุปตามนโยบายของสถาบัน
