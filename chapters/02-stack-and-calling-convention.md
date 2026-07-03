# บท 2 — Stack & Calling Convention (x86-64)

นี่คือบทที่สำคัญที่สุด ถ้าเข้าใจบทนี้แน่น บทที่เหลือจะง่ายไปหมด

## Stack คืออะไร

Stack เป็นโครงสร้างข้อมูลแบบ **LIFO** (Last In First Out) — เข้าทีหลัง
ออกก่อน เหมือนกองจานซ้อนกัน

จุดสำคัญที่คนมักงง: **stack โตจากที่อยู่สูงไปต่ำ**
เวลา push ของเข้า stack ตัวชี้ (RSP) จะ **ลด** ค่าลง

```
push  → RSP ลดลง (stack โตลงล่าง)
pop   → RSP เพิ่มขึ้น (stack หดขึ้นบน)
```

## Register ที่ต้องรู้จัก

| Register | หน้าที่ |
|----------|---------|
| **RSP** | Stack Pointer — ชี้ยอดของ stack (ตำแหน่งบนสุด) |
| **RBP** | Base Pointer — ชี้ฐานของ stack frame ปัจจุบัน |
| **RIP** | Instruction Pointer — ชี้คำสั่งถัดไปที่จะรัน |
| **RAX** | มักเก็บค่า return ของฟังก์ชัน |
| **RDI, RSI, RDX, RCX, R8, R9** | ส่ง argument (ตามลำดับ) |

## Calling Convention (System V AMD64)

บน Linux x86-64 เวลาเรียกฟังก์ชัน argument ส่งผ่าน **register** ก่อน
ไม่ใช่ stack (ต่างจาก x86 32-bit ที่ส่งผ่าน stack)

ลำดับ argument:

```
argument ที่ 1  →  RDI
argument ที่ 2  →  RSI
argument ที่ 3  →  RDX
argument ที่ 4  →  RCX
argument ที่ 5  →  R8
argument ที่ 6  →  R9
argument ที่ 7+ →  stack
```

> **จำให้ขึ้นใจ:** `system("/bin/sh")` ต้องเอา address ของ `"/bin/sh"`
> ใส่ใน **RDI** ก่อนเรียก — นี่คือหัวใจของ ret2libc (บทที่ 7)

ตัวช่วยจำ: **Di**ane's **Si**lk **D**ress **C**osts **8**9 (RDI RSI RDX RCX R8 R9)

## Stack Frame — ชีวิตของหนึ่งฟังก์ชัน

เมื่อเรียกฟังก์ชัน จะเกิดสิ่งเหล่านี้บน stack:

```
        ที่อยู่สูง
┌──────────────────────┐
│  argument (ถ้าเกิน 6) │
├──────────────────────┤
│   Return Address     │  ← address ที่จะกลับไปหลังฟังก์ชันจบ ★ เป้าหมายเรา
├──────────────────────┤
│   Saved RBP          │  ← ค่า RBP เดิมของ caller
├──────────────────────┤  ← RBP ชี้ตรงนี้
│                      │
│   ตัวแปร local        │  ← buffer ต่างๆ อยู่โซนนี้
│   (เช่น char buf[64]) │
│                      │  ← RSP ชี้ยอดบนสุด
└──────────────────────┘
        ที่อยู่ต่ำ
```

### ขั้นตอนตอนเรียกฟังก์ชัน (function prologue)

```asm
call function      ; push return address ลง stack แล้วกระโดดไป function
; --- เข้ามาใน function ---
push rbp           ; เก็บ RBP เดิมไว้
mov  rbp, rsp      ; ตั้ง RBP ใหม่เป็นฐานของ frame นี้
sub  rsp, 0x40     ; จอง 64 ไบต์ให้ตัวแปร local
```

### ขั้นตอนตอนจบฟังก์ชัน (function epilogue)

```asm
leave              ; = mov rsp, rbp ; pop rbp  (คืน stack frame)
ret                ; = pop rip      (ดึง return address ใส่ RIP แล้วกระโดด)
```

## จุดที่ทำให้ pwn เป็นไปได้

สังเกตว่า **buffer (ตัวแปร local) อยู่ที่ address ต่ำกว่า return address**

เมื่อเราเขียนข้อมูลลง buffer มันเขียน**จากล่างขึ้นบน** (address ต่ำ → สูง)
ดังนั้นถ้าเราเขียนเกินขนาด buffer มันจะไหลขึ้นไปทับ:

```
Saved RBP  →  Return Address
```

พอทับ Return Address ได้ = ตอนฟังก์ชัน `ret` มันจะ pop ค่าที่เราเขียน
เข้า RIP = **เราควบคุม RIP ได้!**

```
buffer[64]           [AAAA...AAAA]  ← เราเขียนตรงนี้
saved RBP  (8 ไบต์)  [AAAAAAAA]     ← ล้นมาทับ
return addr (8 ไบต์) [ที่อยู่ที่เราต้องการ]  ← ★ ควบคุมได้แล้ว
```

## ลองดูของจริงใน GDB

```bash
gdb ./vuln
```
```
pwndbg> break main
pwndbg> run
pwndbg> stack 20        # ดู stack 20 ช่อง
pwndbg> info frame      # ดูข้อมูล stack frame
pwndbg> x/20gx $rsp     # ดู 20 ช่องแบบ 8 ไบต์ (g=giant) เลขฐาน 16
```

ลองทำตามแล้วสังเกตว่า return address อยู่ห่างจาก buffer กี่ไบต์ —
ตัวเลขนี้แหละคือ **offset** ที่เราต้องหาในบทถัดไป

---

## สรุปบทนี้

- stack โตลงล่าง (push = RSP ลด)
- argument ส่งผ่าน register: **RDI RSI RDX RCX R8 R9**
- `ret` = `pop rip` → ถ้าคุมค่าบน stack ตรงนั้นได้ = คุม RIP
- buffer อยู่**ต่ำกว่า** return address → overflow ไหลขึ้นไปทับได้

บทต่อไป: ลงมือ overflow จริง →
