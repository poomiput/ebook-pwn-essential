# 🕳️ Pwn Essential — คู่มือ Binary Exploitation ฉบับเข้าใจง่าย

> คู่มือปูพื้นฐาน pwn สำหรับ CTF ระดับเริ่มต้นถึงกลาง
> เขียนแบบสอนให้ "เข้าใจว่าทำไม" ไม่ใช่แค่ท่องสูตร

เป้าหมายของ ebook เล่มนี้คือ ทำให้คุณหยิบโจทย์ pwn ระดับ intro–medium
บน picoCTF / HTB / pwn.college ขึ้นมาแล้ว **รู้ว่าจะเริ่มตรงไหน ทำอะไรต่อ
และเขียน exploit ออกมาได้จริง** โดยไม่ต้องเปิด writeup ทุกครั้ง

---

## 📖 สารบัญ

| บท | เรื่อง | สิ่งที่จะได้ |
|----|--------|-------------|
| [00](chapters/00-how-to-use.md) | วิธีใช้หนังสือเล่มนี้ | เรียนยังไงให้ได้ผลจริง |
| [01](chapters/01-mental-model.md) | ภาพรวม: pwn คืออะไรกันแน่ | mental model ก่อนลงลึก |
| [02](chapters/02-stack-and-calling-convention.md) | Stack & Calling Convention | รู้ว่า RBP/RSP/return address อยู่ตรงไหน |
| [03](chapters/03-buffer-overflow.md) | Buffer Overflow → ควบคุม RIP | ช่องโหว่พื้นฐานที่สุด |
| [04](chapters/04-pwntools.md) | pwntools ให้คล่อง | เครื่องมือคู่ใจ |
| [05](chapters/05-tools.md) | GDB / Ghidra / [checksec](chapters/05-tools.md#1-checksec--ด่านแรกเสมอ) | เครื่องมือที่ต้องใช้เป็น |
| [06](chapters/06-ret2win.md) | ret2win | เทคนิคแรกที่ต้องทำได้ |
| [07](chapters/07-ret2libc.md) | ret2libc | leak libc → system("/bin/sh") |
| [08](chapters/08-rop.md) | ROP Chain | ต่อ gadget เอง |
| [09](chapters/09-format-string.md) | Format String | leak/เขียน memory ผ่าน %p %n |
| [10](chapters/10-got-overwrite.md) | GOT Overwrite | เปลี่ยน flow ของโปรแกรม |
| [11](chapters/11-workflow.md) | Workflow มาตรฐาน | ขั้นตอนที่ใช้ซ้ำได้ทุกโจทย์ |
| [12](chapters/12-next-steps.md) | ไปต่อทางไหน | heap และอื่นๆ |

---

## 🎯 อ่านเล่มนี้จบแล้วจะทำอะไรได้

- อ่าน decompile หา vulnerability เป็น
- หา offset ควบคุม RIP ได้
- ทำ ret2win / ret2libc / ROP ได้คล่อง
- ใช้ format string leak และเขียน memory ได้
- มี exploit template ที่หยิบมาใช้ได้ทันที
- มี workflow ในหัวที่ทำให้ไม่ "ตัน" หน้าโจทย์

## 🛠️ เตรียมเครื่องมือ

```bash
# pwntools
pip install pwntools

# pwndbg (GDB plugin)
git clone https://github.com/pwndbg/pwndbg
cd pwndbg && ./setup.sh

# ROPgadget
pip install ROPgadget

# Ghidra — โหลดจาก https://ghidra-sre.org/
```

---

*เขียนขึ้นเพื่อการศึกษา — ใช้ทักษะเหล่านี้เฉพาะกับระบบที่คุณได้รับอนุญาตเท่านั้น*
