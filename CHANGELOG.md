# Changelog - FearFree Animals Bug Fixes & Security Hardening

## สรุปการแก้ไขทั้งหมด

---

## CRITICAL FIXES

### 1. [BE] Doctor routes ไม่มี role middleware
- **ก่อนแก้:** User คนไหนก็ได้ที่ login แล้ว (patient, admin) สามารถเรียก API `/doctor/*` ได้ทั้งหมด — ดูข้อมูลผู้ป่วย สร้าง/ลบผู้ป่วยได้
- **หลังแก้:** เพิ่ม `IsDoctor` middleware ใน `middleware/auth.middleware.go` และใช้กับ doctor route group — เฉพาะ role=doctor เท่านั้นที่เข้าถึงได้
- **ถ้าไม่แก้:** Patient สามารถยิง `POST /doctor/patients` สร้างผู้ป่วยปลอม หรือ `DELETE /doctor/patients/:id` ลบผู้ป่วยคนอื่นได้ → ข้อมูลทางการแพทย์ถูกเข้าถึงโดยไม่ได้รับอนุญาต
- **ไฟล์:** `middleware/auth.middleware.go`, `routes/routes.go`

### 2. [BE] Race condition ใน RedeemReward — balance ติดลบได้
- **ก่อนแก้:** อ่าน balance ข้างนอก transaction, ไม่ lock row → กด redeem พร้อมกัน 2 ครั้ง balance ติดลบได้
- **หลังแก้:** ย้าย patient query เข้า transaction + `FOR UPDATE` lock + ใช้ `gorm.Expr("balance - ?", ...)` atomic update + re-read หลัง commit
- **ถ้าไม่แก้:** ผู้ป่วยที่มี 30 coins กดแลกรางวัล 30 coins 2 ครั้งพร้อมกัน → ทั้ง 2 ผ่าน → balance เป็น -30 → ได้รางวัลฟรี
- **ไฟล์:** `controllers/reward.controller.go`

### 3. [BE] สร้างผู้ป่วยจาก Doctor ใช้ FearLevel ภาษาไทย "ปานกลาง"
- **ก่อนแก้:** ใส่ `"ปานกลาง"` ลง PostgreSQL enum ที่รับแค่ `low/medium/high` → **DB error ทุกครั้ง** ที่หมอสร้างผู้ป่วย
- **หลังแก้:** เปลี่ยนเป็น `"medium"`
- **ถ้าไม่แก้:** หมอกดปุ่ม "สร้างผู้ป่วยใหม่" แล้ว error 500 ทุกครั้ง ใช้ฟีเจอร์นี้ไม่ได้เลย
- **ไฟล์:** `controllers/doctor.controller.go`

### 4. [BE] generateCodePatient race condition — code ซ้ำได้
- **ก่อนแก้:** COUNT query อยู่นอก transaction → สร้างผู้ป่วยพร้อมกัน 2 คนได้ code เดียวกัน
- **หลังแก้:** ย้าย count เข้า transaction + retry logic 3 ครั้ง
- **ถ้าไม่แก้:** หมอ 2 คนกดสร้างผู้ป่วยพร้อมกัน → ได้รหัสเดียวกัน เช่น CHBCD0005 ทั้งคู่ → patient login ด้วยรหัสนี้จะเข้าผิดคน
- **ไฟล์:** `controllers/doctor.controller.go`

### 5. [BE] bcrypt / JWT error ไม่ถูก check
- **ก่อนแก้:** `bcrypt.GenerateFromPassword` และ `jwt.SignedString` error ถูก discard → สร้าง user ด้วย password hash ว่าง หรือส่ง token ว่างให้ client ได้
- **หลังแก้:** Check error ทั้งหมด return 500 ถ้า fail
- **ถ้าไม่แก้:** ถ้า password ยาวเกิน 72 bytes (limit ของ bcrypt) → user ถูกสร้างด้วย hash ว่าง → ใครก็ login เป็น user นั้นได้ด้วย password ว่าง
- **ไฟล์:** `controllers/auth.controller.go`, `controllers/doctor.controller.go`

### 6. [BE] Middleware claims panic — server crash ได้
- **ก่อนแก้:** `claims["user_id"].(float64)` ไม่ใช้ comma-ok pattern → ถ้า JWT ไม่มี field user_id server **panic crash**
- **หลังแก้:** ใช้ comma-ok pattern, return 401 ถ้า claim หายหรือผิด type
- **ถ้าไม่แก้:** Hacker สร้าง JWT ที่ valid signature แต่ไม่มี user_id → ยิง request เข้ามา → server crash ทั้งเครื่อง (Denial of Service)
- **ไฟล์:** `middleware/auth.middleware.go`

### 7. [FE] Patient login ใช้ mock hardcode — ไม่ยิง API จริง
- **ก่อนแก้:** กรอกอะไรก็ login ผ่าน สร้าง mock user id=99 กับ mock token → ระบบ auth ไม่ทำงานจริง
- **หลังแก้:** เรียก `authService.patientLogin()` จริง พร้อม error handling
- **ถ้าไม่แก้:** ใครก็กรอก code อะไรก็ได้ เข้าหน้าผู้ป่วยได้ทั้งหมด → ข้อมูลสุขภาพไม่ปลอดภัย, เล่นเกมได้ฟรีไม่ต้องมี code จากหมอ
- **ไฟล์:** `src/app/login/patient/page.tsx`

### 8. [FE] Game store functions body ว่างเปล่า
- **ก่อนแก้:** `fetchCategories`, `fetchAnimalsByCategory`, `fetchAnimalAndStages`, `fetchGameRules`, `fetchStageDetail` ทั้ง 5 ตัว body เป็น `/*...*/` → **เกมไม่โหลดข้อมูล** เลย
- **หลังแก้:** Implement ครบทั้ง 5 functions เรียก API จริง + loading/error state
- **ถ้าไม่แก้:** เข้าหน้า "เล่นเกม" → เห็นหน้าว่าง ไม่มี category ให้เลือก → ฟีเจอร์หลักของแอปใช้ไม่ได้เลย
- **ไฟล์:** `src/stores/game.store.ts`

### 9. [FE] Route param [animalID] vs animalId — หน้า Stage ไม่ทำงาน
- **ก่อนแก้:** Folder ชื่อ `[animalID]` แต่ code อ่าน `params.animalId` → ได้ `undefined` → `NaN` → ไม่โหลด stage
- **หลังแก้:** แก้เป็น `params.animalID` ให้ตรงกับชื่อ folder
- **ถ้าไม่แก้:** คลิกเลือกสัตว์แล้ว → เข้าหน้า stage → เห็นหน้าว่าง ไม่มี stage ให้เลือก → เล่นเกมต่อไม่ได้
- **ไฟล์:** `src/app/game/play/stage/[animalID]/page.tsx`

### 10. [FE] Doctor dashboard role check ถูก comment out
- **ก่อนแก้:** Patient/admin เข้าหน้า doctor dashboard ได้ ดูข้อมูลผู้ป่วยทั้งหมด
- **หลังแก้:** Uncomment role check — redirect ไป /login ถ้าไม่ใช่ doctor
- **ถ้าไม่แก้:** ผู้ป่วยพิมพ์ URL `/doctor/dashboard` เข้าได้ → เห็นรายชื่อผู้ป่วยทั้งหมด สร้าง/ลบผู้ป่วยได้ → ละเมิดความเป็นส่วนตัว
- **ไฟล์:** `src/app/doctor/dashboard/page.tsx`

---

## MEDIUM FIXES

### 11. [BE] Assessment score ไม่ validate range
- **ก่อนแก้:** Client ส่ง score 9999 หรือ -100 ได้ → % ผิด, fear level ผิด
- **หลังแก้:** Validate score 0-10 ต่อคำถาม return 400 ถ้าเกิน
- **ถ้าไม่แก้:** ใช้ Postman ส่ง score 999 → ผล fear_level เป็น "high" 9900% → ข้อมูลเสียในฐานข้อมูล หมอเห็นผลที่ไม่ถูกต้อง
- **ไฟล์:** `controllers/assessment.controller.go`

### 12. [BE] Signup ไม่ validate email / FearLevel enum
- **ก่อนแก้:** สมัครด้วย email ว่างหรือ FearLevel="xxx" ได้ → DB error หรือข้อมูลเสีย
- **หลังแก้:** เพิ่ม email regex validation + FearLevel enum whitelist
- **ถ้าไม่แก้:** สมัครด้วย email "abc" ได้ → ส่ง reset password ไม่ได้ / FearLevel ผิด → PostgreSQL enum error 500 leak ข้อมูล DB
- **ไฟล์:** `controllers/auth.controller.go`

### 13. [BE] DeletePatient ไม่ลบข้อมูลเกี่ยวข้อง
- **ก่อนแก้:** ลบ Patient/User แต่ PatientProgress, Assessment, RedemptionHistory ยังอยู่ → orphan data
- **หลังแก้:** ลบทุก related record ใน transaction
- **ถ้าไม่แก้:** ลบผู้ป่วย → มี progress/assessment ค้างใน DB → FK constraint error หรือ orphan rows สะสม → DB ใหญ่ขึ้นเรื่อยๆ
- **ไฟล์:** `controllers/doctor.controller.go`

### 14. [BE] CORS hardcode localhost:3000
- **ก่อนแก้:** Deploy production แล้ว frontend domain ถูก block
- **หลังแก้:** อ่านจาก env `CORS_ORIGIN` fallback localhost:3000
- **ถ้าไม่แก้:** Deploy ขึ้น production domain → frontend เรียก API ไม่ได้เลย → "CORS error" ทุก request
- **ไฟล์:** `main.go`

### 15. [BE] Admin delete ไม่เช็คว่ามีอยู่จริง
- **ก่อนแก้:** ลบ ID ที่ไม่มี → return success
- **หลังแก้:** เช็คก่อน return 404 ถ้าไม่พบ
- **ถ้าไม่แก้:** Admin ลบ category ID 999 (ไม่มี) → API บอก success → Admin คิดว่าลบแล้ว แต่จริงๆ ไม่มีอะไรเกิดขึ้น → สับสน
- **ไฟล์:** `controllers/admin_game.controller.go`

### 16. [BE] API return null แทน [] เมื่อไม่มีข้อมูล
- **ก่อนแก้:** Frontend ต้อง handle null กับ [] ต่างกัน → crash ได้ถ้า `.map()` บน null
- **หลังแก้:** Initialize slice เป็น `[]Type{}` ทุกที่
- **ถ้าไม่แก้:** ผู้ป่วยใหม่ที่ยังไม่มีประวัติเล่น → API return `"data": null` → frontend ทำ `data.map(...)` → **white screen crash**
- **ไฟล์:** `controllers/profile.controller.go`, `controllers/doctor.controller.go`, `controllers/game.controller.go`, `controllers/reward.controller.go`

### 17. [BE] ParamsInt error ไม่ถูก check + tx.Commit() error ไม่ถูก check
- **ก่อนแก้:** URL param ไม่ใช่ตัวเลข → ได้ 0 → query ผิด record / commit fail เงียบ
- **หลังแก้:** Check ทุกที่ return 400/500 ตามสถานการณ์
- **ถ้าไม่แก้:** เรียก `/api/v1/game/categories/abc/animals` → param เป็น 0 → query หา category ID=0 → return ข้อมูลผิดหรือว่างเปล่า
- **ไฟล์:** controllers ทุกไฟล์

### 18. [BE] AutoMigrate ไม่ check error + ขาด Hospital model
- **ก่อนแก้:** Migration fail แต่ app start ด้วย schema เสีย / Hospital table ไม่ถูกสร้าง
- **หลังแก้:** log.Fatal ถ้า migrate fail + เพิ่ม Hospital, UserHospital
- **ถ้าไม่แก้:** เปิด server ใหม่บน DB เปล่า → ไม่มีตาราง hospitals → เรียก API hospital → error 500
- **ไฟล์:** `main.go`

### 19. [FE] Stores fallback mock data เงียบเมื่อ API fail
- **ก่อนแก้:** API ล่มแต่ user เห็นข้อมูล mock ปลอม ไม่รู้ว่ามีปัญหา
- **หลังแก้:** เพิ่ม `toast.error()` แจ้งเตือน + console.warn
- **ถ้าไม่แก้:** Backend ล่ม → user เห็นข้อมูลปลอม (เช่น ชื่อ "Mock User", balance 999) → คิดว่าระบบทำงานปกติ → ตัดสินใจผิดจากข้อมูลเท็จ
- **ไฟล์:** `src/stores/user.store.ts`, `src/stores/doctor.store.ts`, `src/stores/reward.store.ts`

### 20. [FE] redeemReward catch return true — แสดง "สำเร็จ" แม้ fail
- **ก่อนแก้:** API error แต่ UI บอกว่า redeem สำเร็จ
- **หลังแก้:** return false + toast.error
- **ถ้าไม่แก้:** กดแลกรางวัล → backend error → UI แสดง "แลกสำเร็จ!" → ผู้ป่วยไปขอรับรางวัล → ไม่มีรายการ → เสียความเชื่อมั่น
- **ไฟล์:** `src/stores/reward.store.ts`

### 21. [FE] deletePatient ลบจาก UI แม้ API fail
- **ก่อนแก้:** กด delete → API fail → patient หายจากหน้าจอแต่ยังอยู่ใน DB
- **หลังแก้:** ลบจาก UI เฉพาะเมื่อ API success เท่านั้น
- **ถ้าไม่แก้:** หมอกดลบผู้ป่วย → หายจากหน้าจอ → refresh → ผู้ป่วยกลับมา → หมอสับสน กดลบซ้ำ
- **ไฟล์:** `src/stores/doctor.store.ts`

### 22. [FE] Reward store mock field name ผิด
- **ก่อนแก้:** Mock ใช้ `cost` แต่ interface คือ `cost_coins` → แสดง undefined
- **หลังแก้:** แก้เป็น `cost_coins`
- **ถ้าไม่แก้:** หน้า Reward แสดงราคาเป็น "undefined คอยน์" → ผู้ป่วยไม่รู้ว่าต้องใช้กี่ coins
- **ไฟล์:** `src/stores/reward.store.ts`

### 23. [FE] Admin service ส่ง "Bearer null"
- **ก่อนแก้:** ไม่มี token → ส่ง header `Authorization: Bearer null`
- **หลังแก้:** Check token ก่อน set header
- **ถ้าไม่แก้:** Admin ยังไม่ login → เข้าหน้า admin → API ส่ง "Bearer null" → backend ถือว่า invalid token → error 401 ซ้ำๆ
- **ไฟล์:** `src/services/admin.service.ts`

### 24. [FE] ไม่มี 401 interceptor — token หมดอายุไม่ redirect
- **ก่อนแก้:** Token expire แต่ user ยังอยู่ในระบบ เห็น error ทุก API call
- **หลังแก้:** เพิ่ม response interceptor → 401 = clear auth + redirect /login
- **ถ้าไม่แก้:** Token หมดอายุหลัง 72 ชม. → user ยังเห็น UI ปกติ → กดอะไรก็ error → ต้อง clear cache/cookie เอง → UX แย่มาก
- **ไฟล์:** `src/services/apiClient.ts`

### 25. [FE] Forgot password แสดง "ส่งลิงก์แล้ว" โดยไม่มี API
- **ก่อนแก้:** User คิดว่า reset email ถูกส่งแล้ว แต่ไม่มีอะไรเกิดขึ้นจริง
- **หลังแก้:** แสดง toast "ฟีเจอร์นี้ยังไม่พร้อมใช้งาน" + TODO comment
- **ถ้าไม่แก้:** User ลืม password → กด reset → เห็น "ส่งลิงก์แล้ว" → รอ email ไม่มาตลอด → ติดต่อ support เสียเวลา
- **ไฟล์:** `src/app/forgot-password/page.tsx`

### 26. [FE] Assessment result หายเมื่อ refresh → redirect loop
- **ก่อนแก้:** Refresh หน้า → result เป็น null → redirect ไป /assessment วนลูป
- **หลังแก้:** แสดง "ไม่พบผลการประเมิน" + ปุ่มกลับ แทน auto-redirect
- **ถ้าไม่แก้:** ทำแบบประเมินเสร็จ → เห็นผล → กด F5 → วนกลับไปหน้าเริ่มต้น → ทำแบบประเมินใหม่ทั้งหมด → เสียเวลา + ผลเก่าหาย
- **ไฟล์:** `src/app/assessment/result/page.tsx`

### 27. [FE] Register ไม่ validate ชื่อ-สกุลว่าง
- **ก่อนแก้:** ส่ง fullName เปล่าได้
- **หลังแก้:** เพิ่ม `!formData.fullName` ใน validation
- **ถ้าไม่แก้:** สมัครสมาชิกโดยไม่กรอกชื่อ → profile แสดงชื่อว่าง → หมอเห็นผู้ป่วยไม่มีชื่อ → ไม่รู้ว่าเป็นใคร
- **ไฟล์:** `src/app/register/page.tsx`

---

## UI/UX FIXES (จาก Figma comparison)

### 28. [FE] Profile page CSS bug — w-[1px ขาด ]
- **ก่อนแก้:** เส้นแบ่งระหว่าง % ความกลัว กับ ระดับความกลัว ไม่แสดง
- **หลังแก้:** แก้เป็น `w-[1px]`
- **ถ้าไม่แก้:** Tailwind parse class ไม่ได้ → เส้นแบ่งหายไป → ข้อมูล % กับ ระดับ ติดกันอ่านยาก
- **ไฟล์:** `src/app/profile/page.tsx`

### 29. [FE] Login placeholder มี hint "(ขั้นต่ำ X ตัวอักษร)"
- **ก่อนแก้:** Figma แสดงแค่ "ชื่อผู้ใช้งาน" และ "รหัสผ่าน" ไม่มี hint
- **หลังแก้:** ลบ hint ออก ให้ตรง Figma
- **ถ้าไม่แก้:** UI ไม่ตรง design spec → ไม่ผ่าน review จาก designer
- **ไฟล์:** `src/app/login/page.tsx`, `src/app/register/page.tsx`

### 30. [FE] Patient login placeholder ไม่ตรง Figma
- **ก่อนแก้:** "Ex. CHBCD0001"
- **หลังแก้:** "กรอกโค้ดที่ได้รับจากคุณหมอ" (ตาม Figma)
- **ถ้าไม่แก้:** ผู้ป่วยสับสนว่าต้องกรอกอะไร "Ex. CHBCD0001" ดูเหมือนตัวอย่างจริง → พิมพ์ตาม → login ไม่ผ่าน
- **ไฟล์:** `src/app/login/patient/page.tsx`

### 31. [FE] Register มี 5 fields แต่ Figma มี 4
- **ก่อนแก้:** มีช่อง "ยืนยันรหัสผ่าน" ซึ่งไม่มีใน Figma
- **หลังแก้:** ลบช่อง ยืนยันรหัสผ่าน ออก
- **ถ้าไม่แก้:** UI ไม่ตรง Figma → ฟอร์มยาวเกินไป → UX ไม่ดี
- **ไฟล์:** `src/app/register/page.tsx`

### 32. [FE] Doctor redemption status แสดง "สำเร็จ" ตลอด
- **ก่อนแก้:** ternary ทั้ง 2 ฝั่งเป็น "สำเร็จ" เหมือนกัน → `status === 'success' ? 'สำเร็จ' : 'สำเร็จ'`
- **หลังแก้:** แสดงตาม status จริง + สีต่างกัน (เขียว/เหลือง/เทา)
- **ถ้าไม่แก้:** มี redemption ที่ pending/failed → หมอเห็น "สำเร็จ" ทั้งหมด → ไม่รู้ว่ามีรายการที่ยังไม่สำเร็จ
- **ไฟล์:** `src/app/doctor/redemption-history/[patientId]/page.tsx`

### 33. [FE] Profile โรงพยาบาล แสดงทุก role
- **ก่อนแก้:** User ทั่วไปเห็นแถว โรงพยาบาล ด้วย (Figma ไม่มี)
- **หลังแก้:** แสดงเฉพาะ patient เท่านั้น
- **ถ้าไม่แก้:** User ทั่วไปเห็นแถว "โรงพยาบาล: -" → สับสนว่าทำไมมีข้อมูลนี้ → ไม่ตรง Figma
- **ไฟล์:** `src/app/profile/page.tsx`

### 34. [FE] Assessment slider ตัวเลขซ้อน thumb
- **ก่อนแก้:** ตัวเลข 5, 7 ซ้อนทับกับหัว slider อ่านไม่ออก
- **หลังแก้:** เพิ่ม `pt-8` + bg-white/80 + shadow ให้ตัวเลขเด่นขึ้น ไม่ซ้อน
- **ถ้าไม่แก้:** ผู้ป่วยเลื่อน slider → ไม่เห็นตัวเลขชัด → เลือกค่าผิด → ผลประเมินไม่แม่นยำ
- **ไฟล์:** `src/app/assessment/questions/page.tsx`

---

## OTHER FIXES

### 35. [BE] Token stripping ใช้ strings.Replace แทน TrimPrefix
- **หลังแก้:** ใช้ `strings.HasPrefix` + `strings.TrimPrefix` ที่ถูกต้อง
- **ถ้าไม่แก้:** ถ้า header เป็น "bearer " (ตัวเล็ก) หรือรูปแบบอื่น → token ผิด → auth fail โดยไม่มี error message ที่ชัดเจน
- **ไฟล์:** `middleware/auth.middleware.go`

### 36. [BE] Duplicate main() — seed.go, tmp_seed.go, tmp_fix_role.go
- **ก่อนแก้:** Go compiler error — main redeclared → `go build` ไม่ผ่าน
- **หลังแก้:** ย้ายไปอยู่ `cmd/seed/`, `cmd/tmp_seed/`, `cmd/tmp_fix_role/` แยก package
- **ถ้าไม่แก้:** `go build ./...` fail → CI/CD pipeline ไม่ผ่าน → deploy ไม่ได้
- **ไฟล์:** ย้าย 3 files → `cmd/*/main.go`

---

## ROUND 2 — Deep Review & Security Hardening

### 37. [FE] Services ทั้งหมดย้ายมาใช้ apiClient ตัวเดียว
- **ก่อนแก้:** `auth.service.ts`, `user.service.ts`, `assessment.service.ts`, `game.service.ts`, `reward.service.ts`, `doctor.service.ts`, `admin.service.ts` — แต่ละตัวสร้าง axios instance เอง หรือใช้ raw axios → ไม่มี 401 interceptor, ไม่ auto-attach token บางตัว
- **หลังแก้:** ทุก service ใช้ `apiClient` จาก `src/services/apiClient.ts` ตัวเดียว — request interceptor auto-attach Bearer token, response interceptor catch 401 → clear auth + redirect /login
- **ถ้าไม่แก้:** token หมดอายุ → บาง service redirect, บางตัวแสดง error แบบสุ่ม → UX ไม่สม่ำเสมอ, duplicate code กระจายทุกไฟล์
- **ไฟล์:** `src/services/apiClient.ts`, `auth.service.ts`, `user.service.ts`, `assessment.service.ts`, `game.service.ts`, `reward.service.ts`, `doctor.service.ts`, `admin.service.ts`

### 38. [FE] Assessment store unsafe `as any` cast
- **ก่อนแก้:** `submitAnswers` cast response `as any` แล้วอ่าน `data.fear_level` ตรง ๆ → ถ้า backend เปลี่ยน format → crash เงียบ
- **หลังแก้:** Cast เป็น `Record<string, unknown>` + ตรวจ type ทุก field ก่อนใช้
- **ถ้าไม่แก้:** Backend เปลี่ยน response format → frontend crash โดยไม่มี error message → white screen
- **ไฟล์:** `src/stores/assessment.store.ts`

### 39. [FE] Home page hydration mismatch
- **ก่อนแก้:** `page.tsx` ไม่มี `"use client"` แต่ import Navbar ที่ใช้ client-side state → React hydration warning + flickering
- **หลังแก้:** เพิ่ม `"use client"` ที่บรรทัดแรก
- **ถ้าไม่แก้:** Server render ไม่ตรงกับ client → UI กระพริบ, console warning เยอะ → ใน Next.js 16 อาจ break rendering
- **ไฟล์:** `src/app/page.tsx`

### 40. [FE] Game timer นับติดลบ + ไม่ auto-submit
- **ก่อนแก้:** Timer หมดเวลาแล้วยังนับต่อเป็นลบ (-1, -2, ...) → ผู้ป่วยเล่นได้ไม่จำกัดเวลา
- **หลังแก้:** Timer หยุดที่ 0 + auto-submit คำตอบเมื่อหมดเวลา
- **ถ้าไม่แก้:** เกมที่จำกัดเวลา 30 วินาที → ผู้ป่วยใช้เวลา 5 นาทีก็ได้ → ระบบให้คะแนนเหมือนตอบทันเวลา → ผลเกมไม่แม่นยำ
- **ไฟล์:** `src/app/game/play/run/[animalId]/[stageNo]/page.tsx`

### 41. [FE] Game stage ไม่ fetch stages on mount
- **ก่อนแก้:** เข้าหน้า stage แล้วไม่โหลดข้อมูล stage → หน้าว่างเปล่า
- **หลังแก้:** เพิ่ม `useEffect` fetch `fetchAnimalAndStages` เมื่อ component mount
- **ถ้าไม่แก้:** เลือกสัตว์แล้ว → เข้าหน้า stage → เห็นหน้าว่าง ไม่มีด่านให้เลือก
- **ไฟล์:** `src/app/game/play/stage/[animalID]/page.tsx`

### 42. [FE] Doctor store deletePatient ไม่ลบจาก recentCreated
- **ก่อนแก้:** ลบผู้ป่วยสำเร็จ → หายจากรายชื่อหลัก แต่ยังเห็นใน "ผู้ป่วยที่เพิ่งสร้าง"
- **หลังแก้:** ลบจากทั้ง `patients` และ `recentCreated` arrays
- **ถ้าไม่แก้:** หมอลบผู้ป่วย → ยังเห็นชื่อใน section "เพิ่งสร้าง" → กดเข้าไป → 404 error → สับสน
- **ไฟล์:** `src/stores/doctor.store.ts`

### 43. [FE] useEffect infinite loop ใน categories + play page
- **ก่อนแก้:** `fetchCategories()` อยู่ใน useEffect โดยมี store function เป็น dependency → re-render ทุก render → **infinite API call loop** → tab แฮงค์
- **หลังแก้:** ใช้ `useRef` เก็บ flag `hasFetched` ป้องกันเรียกซ้ำ
- **ถ้าไม่แก้:** เข้าหน้าเลือกหมวดหมู่เกม → ยิง API ไม่หยุด → browser แฮงค์ + backend โดน DDoS จาก client เดียว
- **ไฟล์:** `src/app/game/categories/page.tsx`, `src/app/game/play/[categoryId]/page.tsx`

### 44. [FE] Assessment age ไม่เช็ค isNaN + whitespace
- **ก่อนแก้:** กรอกอายุ "abc" → parseInt ได้ NaN → ส่งไป backend เป็น NaN
- **หลังแก้:** เพิ่ม `isNaN()` check + `trim()` input
- **ถ้าไม่แก้:** ผู้ป่วยกรอกอายุผิด → ผลประเมินผิดพลาด → หมอเห็นข้อมูลอายุที่ไม่ถูกต้อง
- **ไฟล์:** `src/app/assessment/page.tsx`

### 45. [FE] Doctor dashboard hydration + alert→toast
- **ก่อนแก้:** Doctor dashboard ใช้ `window.confirm()` → block UI + ไม่มี hydration safety
- **หลังแก้:** เปลี่ยนเป็น toast notification + เพิ่ม hydration guard
- **ถ้าไม่แก้:** กดลบผู้ป่วย → popup แบบ native ดูไม่ professional → ไม่ตรง UX ของแอป
- **ไฟล์:** `src/app/doctor/dashboard/page.tsx`

### 46. [BE] Doctor ownership model — หมอเห็นเฉพาะผู้ป่วยตัวเอง
- **ก่อนแก้:** หมอทุกคนเห็นผู้ป่วยทุกคนในระบบ ลบผู้ป่วยของหมอคนอื่นได้
- **หลังแก้:** เพิ่ม `CreatedByDoctorID` ใน Patient model + filter query ทุก endpoint ด้วย doctor ID จาก JWT
- **ถ้าไม่แก้:** หมอ A เห็นผู้ป่วยของหมอ B → ดูประวัติ/ลบได้ → ละเมิด privacy ทางการแพทย์
- **ไฟล์:** `models/user.model.go`, `controllers/doctor.controller.go`

### 47. [BE] Game progress race condition — duplicate records
- **ก่อนแก้:** PatientProgress query ไม่ lock row → 2 requests พร้อมกัน → สร้าง record ซ้ำ → คะแนนถูกนับ 2 เท่า
- **หลังแก้:** เพิ่ม `FOR UPDATE` lock + unique index `idx_patient_stage` บน PatientProgress
- **ถ้าไม่แก้:** ผู้ป่วยกดส่งคำตอบ 2 ครั้งเร็วๆ → ได้ coins 2 เท่า → balance เพิ่มเกินจริง
- **ไฟล์:** `controllers/game.controller.go`, `models/game.model.go`

### 48. [BE] JWT_SECRET ค่า default ใช้ได้ใน production
- **ก่อนแก้:** ถ้าไม่ตั้ง env `JWT_SECRET` → ใช้ค่า default "secret" → ใครก็ปลอม JWT ได้
- **หลังแก้:** `log.Fatal` ถ้า JWT_SECRET ไม่ได้ตั้งค่า หรือเป็นค่า default
- **ถ้าไม่แก้:** Deploy production ลืมตั้ง env → JWT secret = "secret" → hacker สร้าง JWT เองเข้าระบบเป็นใครก็ได้
- **ไฟล์:** `config/config.go`

### 49. [BE] Admin cascade delete ไม่ลบ child records
- **ก่อนแก้:** ลบ Category → Animal ยังอยู่ orphan, ลบ Animal → Stage/PatientProgress ยังอยู่ orphan → FK constraint error หรือ data leak
- **หลังแก้:** ลบใน transaction: Category → ลบ Animals + Stages + Progress ทั้งหมด
- **ถ้าไม่แก้:** Admin ลบหมวดหมู่ "สุนัข" → สัตว์ทั้งหมดในหมวดยังอยู่ → DB ข้อมูลเสีย → query ผิดพลาด
- **ไฟล์:** `controllers/admin_game.controller.go`

### 50. [BE] Admin input validation ไม่ครบ
- **ก่อนแก้:** สร้าง stage ด้วย `stage_no: -1`, `reward_coins: -100`, `display_time: 0`, `media_type: "xxx"` ได้ → ข้อมูลเสีย
- **หลังแก้:** Validate ทุก field: stage_no > 0, reward_coins >= 0, display_time > 0, media_type ∈ {image, video, gif}, เช็ค category/animal มีอยู่จริง
- **ถ้าไม่แก้:** Admin สร้าง stage ผิด → ผู้ป่วยเล่นเกม → display_time = 0 → เกมจบทันที / reward_coins = -100 → หักเงินผู้ป่วย
- **ไฟล์:** `controllers/admin_game.controller.go`, `controllers/admin_reward.controller.go`

### 51. [BE] Unchecked Save/Delete/Create errors ทั่วทั้ง backend
- **ก่อนแก้:** `db.Save()`, `db.Delete()`, `db.Create()` หลายจุดไม่ check `.Error` → fail เงียบ
- **หลังแก้:** Check error ทุกจุด return 500 พร้อม generic message (ไม่ leak SQL)
- **ถ้าไม่แก้:** DB connection drop → Save fail → API return 200 success → client คิดว่าบันทึกแล้ว → ข้อมูลหาย
- **ไฟล์:** `controllers/profile.controller.go`, `controllers/hospital.controller.go`, `controllers/game.controller.go`, `controllers/admin_game.controller.go`, `controllers/admin_reward.controller.go`

### 52. [BE] UserHospital unique index + model migration
- **ก่อนแก้:** ไม่มี unique index บน `UserHospital.UserID` → user ผูก hospital ซ้ำได้หลาย record → query join ได้ผลซ้ำ
- **หลังแก้:** เพิ่ม `gorm:"uniqueIndex"` + เพิ่ม Hospital/UserHospital ใน AutoMigrate
- **ถ้าไม่แก้:** ผู้ป่วยผูกโรงพยาบาลซ้ำ 5 ครั้ง → profile แสดงโรงพยาบาลซ้ำ 5 แถว → DB ข้อมูลซ้ำ
- **ไฟล์:** `models/hospital.model.go`, `main.go`

### 53. [BE] Rate limiting บน auth routes
- **ก่อนแก้:** Login/Register ไม่มี rate limit → brute force password ได้ไม่จำกัด
- **หลังแก้:** เพิ่ม Fiber rate limiter middleware (20 req/min) บน auth route group
- **ถ้าไม่แก้:** Hacker ยิง login request 1000 ครั้ง/วินาที → brute force password → เข้าถึงบัญชีผู้ป่วย/หมอ
- **ไฟล์:** `routes/routes.go`

### 54. [BE] Error middleware — 500 error leak SQL/stack trace
- **ก่อนแก้:** Internal server error ส่ง error message ดิบกลับ client → อาจ leak ชื่อตาราง, SQL query, stack trace
- **หลังแก้:** Error middleware แทนที่ 500 message ด้วย generic "Internal Server Error"
- **ถ้าไม่แก้:** Frontend แสดง error ที่มี SQL query → hacker เห็นโครงสร้าง DB → ใช้ทำ SQL injection
- **ไฟล์:** `middleware/error.middleware.go`
