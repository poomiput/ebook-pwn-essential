# บท 4 — pwntools ให้คล่อง

pwntools คือ library Python ที่ทำให้เขียน exploit เร็วขึ้นมาก บทนี้
รวมทุกอย่างที่ต้องใช้จนคล่อง

## Template เริ่มต้น (จำให้ขึ้นใจ)

```python
from pwn import *

context.binary = elf = ELF('./vuln')   # ตั้ง arch/endian อัตโนมัติ
context.log_level = 'debug'            # เปิดดู byte ที่รับ-ส่ง (ตอน debug)

# เลือกโหมด: local หรือ remote
p = process('./vuln')
# p = remote('mercury.picoctf.net', 12345)

# ===== exploit ตรงนี้ =====

p.interactive()
```

> **เคล็ดลับ:** ทำ template นี้เป็นไฟล์ `template.py` แล้วก็อปทุกครั้งที่
> เริ่มโจทย์ใหม่ ไม่ต้องพิมพ์ใหม่

## การรับ-ส่งข้อมูล (I/O)

นี่คือส่วนที่ใช้บ่อยที่สุด แยกให้ชัดระหว่าง "รับ" กับ "ส่ง":

### ส่งข้อมูล (send)

```python
p.send(b'hello')              # ส่งเฉยๆ ไม่มี newline
p.sendline(b'hello')          # ส่ง + newline (เหมือนกด Enter)
p.sendafter(b'name: ', data)  # รอเจอ 'name: ' ก่อนแล้วค่อยส่ง
p.sendlineafter(b'> ', data)  # รอเจอ '> ' แล้วส่ง + newline (ใช้บ่อยสุด)
```

### รับข้อมูล (recv)

```python
p.recv(1024)              # รับสูงสุด 1024 ไบต์
p.recvline()              # รับจนเจอ newline หนึ่งบรรทัด
p.recvuntil(b'flag: ')    # รับจนกว่าจะเจอ 'flag: ' (ทิ้งที่รับมา)
p.recvuntil(b'0x')        # นิยมใช้ก่อน leak address
p.recvn(6)                # รับพอดี 6 ไบต์ (ใช้ตอนอ่าน leaked address)
```

## แปลงตัวเลข ↔ ไบต์ (สำคัญมากสำหรับ pwn)

address และตัวเลขต้องแปลงเป็นไบต์แบบ little-endian ก่อนส่ง:

```python
# pack: ตัวเลข → ไบต์
p64(0x401156)     # 64-bit → b'\x56\x11\x40\x00\x00\x00\x00\x00'
p32(0x8048456)    # 32-bit

# unpack: ไบต์ → ตัวเลข (ใช้ตอนอ่าน leaked address)
u64(data)         # ต้องได้ 8 ไบต์พอดี
u32(data)         # ต้องได้ 4 ไบต์พอดี
```

### แพทเทิร์นอ่าน leaked address (เจอบ่อยมาก)

leak มักได้ไม่ครบ 8 ไบต์ ต้อง pad ให้ครบก่อน unpack:

```python
# วิธีที่ 1: รับ 6 ไบต์ แล้ว pad ด้วย \x00 ให้ครบ 8
leak = u64(p.recv(6).ljust(8, b'\x00'))

# วิธีที่ 2: รับทั้งบรรทัดแล้วตัด
p.recvuntil(b'0x')
leak = int(p.recvline().strip(), 16)

log.info(f'leaked address: {hex(leak)}')   # print เลขฐาน 16 สวยๆ
```

## ทำงานกับ ELF (หา address ง่ายๆ)

```python
elf = ELF('./vuln')

elf.symbols['win']        # address ของฟังก์ชัน win
elf.symbols['system']     # address ของ system (ถ้ามีใน binary)
elf.plt['puts']           # address ของ puts ใน PLT
elf.got['puts']           # address ของ puts ใน GOT
elf.address               # base address (ถ้า PIE จะเป็น 0 จนกว่าจะ set)
```

## ทำงานกับ libc

```python
libc = ELF('./libc.so.6')

# หลัง leak address ของฟังก์ชันหนึ่งได้ ตั้ง base:
libc.address = leaked_puts - libc.symbols['puts']

# แล้วหา address อื่นได้เลย:
system = libc.symbols['system']
binsh  = next(libc.search(b'/bin/sh'))
```

## หา gadget อัตโนมัติ (ใช้ในบท ROP)

```python
rop = ROP(elf)
rop.raw(rop.find_gadget(['ret'])[0])   # หา ret gadget
pop_rdi = rop.find_gadget(['pop rdi', 'ret'])[0]

# หรือสร้าง chain อัตโนมัติ
rop.call('system', [binsh])
payload = flat(b'A'*offset, rop.chain())
```

## ประกอบ payload ด้วย flat()

```python
# แทนที่จะต่อ string ยาวๆ ใช้ flat() อ่านง่ายกว่า
payload = flat(
    b'A' * offset,
    p64(pop_rdi),
    p64(binsh),
    p64(system),
)
```

## debug ด้วย GDB จาก pwntools

```python
p = gdb.debug('./vuln', '''
    break vuln
    continue
''')
# เปิด GDB พร้อม breakpoint อัตโนมัติ สะดวกมากตอน debug exploit
```

## คำสั่ง command-line ที่มากับ pwntools

```bash
pwn checksec ./vuln       # เช็ค mitigation
pwn cyclic 200            # สร้าง pattern
pwn cyclic -l 0x6161...   # หา offset
pwn template ./vuln       # สร้าง exploit template อัตโนมัติ!
pwn disasm '4831c0'       # แปลง hex → assembly
```

> `pwn template ./vuln` สร้าง skeleton exploit ให้เลย ลองใช้ดู

---

## Cheat Sheet สรุป

| ต้องการ | ใช้ |
|---------|-----|
| รอ prompt แล้วส่ง | `sendlineafter(b'> ', payload)` |
| รับจนเจอคำ | `recvuntil(b'flag')` |
| ตัวเลข → ไบต์ | `p64(x)` |
| ไบต์ → ตัวเลข | `u64(data.ljust(8, b'\x00'))` |
| หา address ฟังก์ชัน | `elf.symbols['win']` |
| หา /bin/sh ใน libc | `next(libc.search(b'/bin/sh'))` |
| เข้า shell | `p.interactive()` |

---

## สรุปบทนี้

- มี template ในหัว: `context.binary` → I/O → `interactive()`
- `sendlineafter` + `recvuntil` คือคู่หูที่ใช้บ่อยสุด
- `p64/u64` ต้องคล่องเป็นอัตโนมัติ
- `ELF` และ `libc` object ช่วยหา address ให้หมด

บทต่อไป: เครื่องมือ GDB / Ghidra / checksec →
