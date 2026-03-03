# HRMS Architecture Overview (สรุปภาพรวมโครงสร้าง)

เอกสารนี้เป็นภาพรวมโครงสร้างระบบ HRMS สำหรับ 4 บริษัทในแพลตฟอร์มเดียว (multi-company / multi-tenant) ที่รองรับ 3 บทบาทหลัก:

- Employee (User)
- Team Lead (หัวหน้าทีม)
- HR Admin (ผู้ดูแลเต็มระบบ)

---

## 1) เป้าหมายหลักของระบบ

1. ระบบลงเวลา (Check-in / Check-out) ที่รองรับพิกัดหลายจุดและรัศมีตามบุคคล
2. ระบบคำขอ (ลา/ขาด/OT/แก้เวลา) พร้อม workflow อนุมัติหลายชั้น
3. ระบบโควต้าการลาแบบรายบริษัท รายนโยบาย และรายบุคคล
4. ระบบเงินเดือนที่คำนวณได้ยืดหยุ่นตามกฎ (สาย/ขาด/ประกันสังคม/ภาษี)
5. ระบบรายงานและดาวน์โหลดข้อมูลระดับองค์กร
6. ความเสถียรสูง พร้อมตรวจสอบย้อนหลัง (audit trail)

---

## 2) ขอบเขตโมดูลหลัก (High-level Modules)

### A. Identity & Access
- Login ด้วย Google OAuth + Email/Password
- RBAC + Scope ตามบริษัท/ทีม/บุคคล
- รองรับการย้ายบทบาท Employee ↔ Team Lead ↔ HR

### B. Workforce Core
- ประวัติพนักงาน (Master profile + Employment history)
- โครงสร้างองค์กร (บริษัท/แผนก/ทีม/หัวหน้า)
- ตำแหน่งงานและสายบังคับบัญชา

### C. Time & Attendance
- Check-in/Check-out ด้วย geofence (lat/lng + radius)
- รองรับหลายจุดสำหรับแต่ละคน
- รองรับกะปกติ, Flex time, และกะข้ามวัน
- คำนวณ work_date ให้ถูกต้องตามกะ

### D. Leave & Request Workflow
- คำขอลา (ลาป่วย/ลากิจ/พักร้อน/อื่นๆ)
- คำขอแก้เวลาเมื่อลืมเช็คอิน/เช็คเอาท์
- อนุมัติ/ปฏิเสธโดยหัวหน้า และ override โดย HR
- แจ้งเตือนสถานะทุกขั้นตอน

### E. Payroll Engine
- กฎคำนวณเงินเดือนแบบกำหนดได้
- หักสายรายนาที, ขาดงาน, ประกันสังคม, ภาษี
- รองรับพนักงานคนละ policy/contract

### F. Reporting & Export
- รายงานการเข้างานรายวัน/รายเดือน
- รายงานการลาและโควต้าคงเหลือ
- Export CSV/Excel สำหรับ HR/บัญชี

### G. Admin Control Tower (HR)
- ตั้งนโยบายบริษัท (เวลาเข้างาน, ประเภทลา, กฎเงินเดือน)
- จัดทีม/ย้ายทีม/เปลี่ยนหัวหน้า
- จัดสิทธิ์การเข้าถึงระดับรายบุคคล

---

## 3) โมเดลสิทธิ์ (Role Model)

- **Employee**
  - ลงเวลา, ส่งคำขอ, ดูประวัติของตน, แก้โปรไฟล์ตน
- **Team Lead**
  - เห็นข้อมูลทีม, อนุมัติ/ปฏิเสธคำขอทีม, ดู attendance ทีม
- **HR Admin**
  - เห็นทั้งองค์กร, ตั้ง policy, แก้ข้อมูลบุคคล, ปิดงวดเงินเดือน, export ทั้งหมด

> แนะนำใช้ RBAC + Policy Condition (เช่น `company_id`, `team_id`, `user_id`) เพื่อกันข้อมูลรั่วข้ามบริษัท

---

## 4) โครงสร้างข้อมูลที่ควรมี (Conceptual Data Model)

### Core entities
- `companies`
- `users`
- `employee_profiles`
- `employment_history`
- `teams`
- `team_members`
- `shift_definitions`
- `employee_shift_assignments`
- `geofence_locations`
- `employee_geofence_permissions`
- `attendance_logs` (raw check-in/out)
- `attendance_daily_summary` (สรุปรายวัน)
- `leave_types`
- `leave_quotas`
- `leave_requests`
- `time_adjustment_requests`
- `approval_workflows`
- `notifications`
- `payroll_policies`
- `payroll_runs`
- `payroll_items`
- `audit_logs`

### หลักสำคัญด้านเวลา
- เก็บเวลาเป็น UTC ทั้งหมด
- เก็บ timezone ของบริษัท/พนักงานเพิ่ม
- คำนวณ `work_date` จาก shift assignment ไม่ใช่จาก calendar date ตรงๆ

---

## 5) หลักคิดเรื่องกะข้ามวันและ Flex Time

### กะข้ามวัน (เช่น 20:00 - 05:00)
- ใช้ `shift_start_at`, `shift_end_at`, `grace_period`
- บันทึกคู่กับ `work_date` ของวันที่เริ่มกะ
- การเช็คเอาท์หลังเที่ยงคืนยังผูกกับ `work_date` เดิม

### Flex Time
- กำหนดกรอบเวลาเข้าได้ เช่น 07:00-10:00
- ระบบคำนวณ expected end time ตามชั่วโมงที่ต้องทำ
- สาย/ขาดคำนวณตาม policy ของแต่ละบริษัท

---

## 6) สถาปัตยกรรมเชิงเทคนิค (แนะนำแบบคุมค่าใช้จ่าย)

### ชั้นระบบ
1. **Frontend**: Next.js (Light mode, โทนฟ้าขาว)
2. **Transactional DB + Auth**: Supabase (Postgres + Auth + RLS)
3. **Async Jobs/Workers**: Queue สำหรับคำนวณสรุปรายวัน/แจ้งเตือน/ปิดงวดเงินเดือน
4. **Analytics**: BigQuery (ดึงเฉพาะข้อมูลสรุป/รายงานหนัก)

### แนวทางผสม Supabase + BigQuery
- งานธุรกรรมรายวันทั้งหมดอยู่ Supabase (ต้นทุนต่ำและ latency ต่ำ)
- ส่งข้อมูลสรุปเป็นรอบ (batch/stream) ไป BigQuery เพื่อทำ dashboard/report หนัก
- ตั้ง retention/raw log และ partition ใน BigQuery เพื่อลดค่า query

---

## 7) Workflow หลักที่ต้องทำก่อน (MVP → Phase 2)

### MVP (ต้องมี)
1. Auth + RBAC + Multi-company isolation
2. Employee profile + Team structure
3. Check-in/out + geofence + work_date logic
4. Leave request + approval + notification
5. Attendance dashboard รายเดือนของพนักงาน
6. HR admin policy พื้นฐาน + export CSV

### Phase 2
1. Payroll engine เต็มรูปแบบ + rule builder
2. Advanced analytics บน BigQuery
3. Auto anomaly detection (มาสายผิดปกติ, ลืมเช็คบ่อย)
4. Mobile PWA และ push notification

---

## 8) เสถียรภาพและความปลอดภัย (ต้องล็อกตั้งแต่ต้น)

- RLS ทุกตารางที่มีข้อมูลบุคคล
- Audit log สำหรับการแก้ข้อมูลสำคัญ
- Idempotency key ในการลงเวลา ป้องกันบันทึกซ้ำ
- Background retry + dead-letter queue สำหรับงาน async
- Monitoring: error rate, queue lag, payroll job duration
- Backup/restore drill รายเดือน

---

## 9) UX/UI แนวทางสั้นๆ

- โทนสี: Light mode ฟ้า-ขาว
- หน้า employee ต้องเห็น 3 อย่างชัดเจนทันที:
  1) ปุ่มเช็คอิน/เอาท์
  2) สถานะคำขอล่าสุด
  3) โควต้าลาคงเหลือ
- หน้า team lead เน้น inbox งานอนุมัติ
- หน้า HR เน้น control tower + report + policy center

---

## 10) สรุปโครงสร้างโดยย่อ

แนะนำเริ่มจากแกน **Supabase + Next.js + RBAC + Attendance/Leave** ให้เสถียรก่อน แล้วค่อยขยาย **Payroll + BigQuery Analytics** ในเฟสถัดไป เพื่อคุมความเสี่ยงและต้นทุน พร้อมรองรับการเติบโตหลายบริษัทบนระบบเดียว.
