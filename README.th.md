# pbi-breakdown

[English version](README.md)

[Agent Skill](https://github.com/vercel-labs/skills) สำหรับแตก PBI ของ sprint **หนึ่งใบ** ออกเป็น dev task ที่เชื่อถือได้และอ้างอิง artifact จริง สำหรับทีมที่เลือก (FE / BE / MB) — ทุก task มีประมาณการเวลาและตั้งชื่อตาม pattern เดียวกัน ส่งออกเป็น markdown checklist

## ทำอะไรได้บ้าง

- **เลือกทีมก่อน** — ถามว่าทีมไหนรับงาน (FE / BE / MB) แล้วแตกงานเฉพาะส่วนของทีมนั้น
- **อ้างอิง artifact จริง** — ทุก task ต้องชี้ไปที่ endpoint / table / field / component ที่มีชื่อจริง ไม่มี artifact = ไม่มี task
- **ชุดคำศัพท์ work-type แบบปิด** — รายการ label ต่อทีม (เช่น BE `Migration`/`Upload`/`Integration`, MB `Local store`/`Offline sync`/`Permission`/`Device`) เพื่อให้ชื่อ task สม่ำเสมอทุกครั้งที่รัน
- **ชี้จุดที่ข้อมูลขาด ไม่เดาเอง** — `⚠ spec-gap` สำหรับข้อมูลที่หายไป, `❓ open-decision` สำหรับ business rule ที่ยังไม่ตัดสินใจ
- **ประมาณการตรงไปตรงมา** — effort จริงต่อ task (ไม่มีเพดาน) และเตือนเมื่อ task เกิน 5 ชั่วโมง
- **Dependency + งานข้ามทีม** — `↳ after #n` สำหรับลำดับงานในทีมเดียวกัน, `🔗` สำหรับ handoff ข้ามทีม
- **ถาม-ตอบทีละขั้น** ผ่าน skill `grill-me` แล้วได้ markdown checklist พร้อมส่งต่อเข้า Azure DevOps

## ติดตั้ง

```bash
npx skills@latest add mekkkwiz/pbi-breakdown-skill
```

จากนั้นเรียกใช้ใน Claude Code ด้วย `/pbi-breakdown` หรือแค่วาง PBI แล้วบอกให้ "break it down"

## ไฟล์

- `skills/pbi-breakdown/SKILL.md` — ตัว skill
- `skills/pbi-breakdown/EXAMPLES.md` — ตัวอย่างการรันแบบเต็ม (BE / FE / MB)
