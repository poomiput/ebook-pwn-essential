# บท 5 — เครื่องมือที่ต้องใช้เป็น

pwn ที่ดีมาจากการอ่าน binary เป็นและ debug เป็น บทนี้รวมเครื่องมือหลัก 3 ตัว

---

## 1. checksec — ด่านแรกเสมอ

ก่อนทำอะไร ดูก่อนว่า binary เปิด mitigation อะไรไว้:

```bash
$ pwn checksec ./vuln
[*] '/home/user/vuln'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found      ← ไม่มี canary → overflow ตรงๆ ได้
    NX:       NX enabled           ← รัน shellcode บน stack ไม่ได้ → ใช้ ROP
    PIE:      No PIE (0x400000)    ← address คงที่ → ไม่ต้อง leak โค้ด
```

### อ่านผลยังไง → เลือกเทคนิคยังไง

| ผล checksec | แปลว่า | ทำยังไง |
|-------------|--------|---------|
| No canary | overflow ทับ return ได้เลย | ลุยตรงๆ |
| Canary found | มีค่าเฝ้า | ต้อง leak canary ก่อน |
| NX enabled | รันโค้ดบน stack ไม่ได้ | ใช้ ret2libc / ROP |
| NX disabled | รัน shellcode บน stack ได้ | inject shellcode ได้ |
| No PIE | โค้ดอยู่ที่ address คงที่ | ใช้ address ตรงๆ |
| PIE enabled | โค้ดสุ่มตำแหน่ง | ต้อง leak address โค้ดก่อน |
| Partial RELRO | เขียน GOT ได้ | ทำ GOT overwrite ได้ |
| Full RELRO | เขียน GOT ไม่ได้ | หาทางอื่น |

> **นี่คือขั้นตอนแรกของ workflow ทุกโจทย์** — checksec บอกเราว่าเกมนี้
> เล่นยังไง

---

## 2. Ghidra / objdump — อ่าน binary

### objdump (เร็ว ใช้ดูเร็วๆ)

```bash
objdump -d ./vuln              # disassemble ทั้งหมด
objdump -d ./vuln | grep win   # หาฟังก์ชัน win
objdump -M intel -d ./vuln     # ใช้ Intel syntax (อ่านง่ายกว่า AT&T)
objdump -R ./vuln              # ดู GOT entries
```

### Ghidra (decompile เป็น C อ่านง่าย)

Ghidra แปลง assembly กลับเป็นโค้ดคล้าย C ทำให้หา vulnerability ง่ายขึ้นมาก

ขั้นตอนใช้งาน:
1. เปิด Ghidra → สร้าง project ใหม่
2. ลาก binary เข้าไป → กด Analyze (เอา default ได้)
3. ดับเบิลคลิกฟังก์ชัน `main` ในหน้าต่าง Symbol Tree
4. อ่านโค้ด decompile ในหน้าต่างขวา

สิ่งที่มองหาในโค้ด decompile:

```c
// ช่องโหว่ที่พบบ่อย — มองหาพวกนี้
gets(buf);                    // ← ไม่จำกัดขนาด = overflow
read(0, buf, 0x100);          // ← ถ้า 0x100 > ขนาด buf = overflow
scanf("%s", buf);             // ← %s ไม่จำกัด = overflow
strcpy(dst, src);             // ← ถ้า src ยาวกว่า dst = overflow
printf(user_input);           // ← format string! (ควรเป็น printf("%s", input))
system(user_controlled);      // ← command injection
```

> **เทคนิค:** ใน Ghidra กด `L` ที่ตัวแปรเพื่อเปลี่ยนชื่อให้สื่อความหมาย
> จะอ่านโค้ดง่ายขึ้นมาก

---

## 3. GDB + pwndbg — debug ตัวจริง

GDB เปล่าๆ ใช้ยาก ให้ลง **pwndbg** (หรือ GEF) ช่วยแสดงผลสวยงาม

### คำสั่งพื้นฐาน

```bash
gdb ./vuln

# --- ตั้ง breakpoint ---
break main            # หยุดที่ main
break vuln            # หยุดที่ฟังก์ชัน vuln
break *0x401156       # หยุดที่ address เจาะจง

# --- ควบคุมการรัน ---
run                   # เริ่มรัน (ย่อ: r)
continue              # รันต่อ (ย่อ: c)
nexti                 # รันคำสั่งถัดไป ข้าม call (ย่อ: ni)
stepi                 # รันคำสั่งถัดไป เข้า call (ย่อ: si)
finish                # รันจนจบฟังก์ชันปัจจุบัน
```

### ดูหน่วยความจำและ register

```bash
# --- register ---
info registers        # ดู register ทั้งหมด
p $rsp                # print ค่า RSP
p $rip

# --- ดู memory (x = examine) ---
x/20gx $rsp           # 20 ช่อง, g=8ไบต์, x=ฐาน16 — ดู stack
x/8gx $rbp            # ดูรอบๆ RBP
x/s 0x402004          # ดูเป็น string
x/20i $rip            # ดู 20 คำสั่งถัดไป (i=instruction)

# --- pwndbg เสริม ---
stack 20              # ดู stack สวยๆ 20 ช่อง
telescope $rsp        # ดู stack แบบไล่ตาม pointer
vmmap                 # ดู memory map (หา base address ต่างๆ)
```

### หา offset ด้วย cyclic (ใน pwndbg)

```bash
pwndbg> cyclic 200                # สร้าง pattern
pwndbg> run                       # paste ตอนโปรแกรมถาม
# ... โปรแกรม crash ...
pwndbg> cyclic -l 0x6161616c      # เอาค่าใน RSP/RIP มาหา offset
72
```

### เทคนิคที่ใช้บ่อย

```bash
# ดูว่า crash ที่ไหน ตอน RIP = ขยะ
pwndbg> run
# หลัง crash:
pwndbg> info registers rip

# ดู GOT
pwndbg> got

# หา string ในหน่วยความจำ
pwndbg> search "/bin/sh"

# ดู disassembly ฟังก์ชัน
pwndbg> disassemble vuln
```

---

## Workflow เครื่องมือ (สรุปการใช้ร่วมกัน)

```
1. checksec ./vuln          → รู้ว่าเปิด mitigation อะไร
        ↓
2. Ghidra / objdump         → หา vulnerability + ฟังก์ชันที่น่าสนใจ
        ↓
3. GDB + cyclic             → หา offset, ดู stack จริง
        ↓
4. pwntools                 → เขียน exploit
```

---

## สรุปบทนี้

- **checksec** = ด่านแรกเสมอ บอกว่าจะเล่นเกมนี้ยังไง
- **Ghidra/objdump** = อ่านหา vulnerability
- **GDB+pwndbg** = ดู memory จริง หา offset
- คำสั่ง GDB ที่ใช้บ่อย: `b`, `r`, `c`, `x/20gx $rsp`, `cyclic`

บทต่อไป: เทคนิคแรก ret2win →
