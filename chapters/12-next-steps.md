# บท 12 — ไปต่อทางไหน

จบ core แล้ว บทนี้คือแผนที่ระดับถัดไป — จัดลำดับตามความคุ้มค่า

## เช็คก่อน: คุณพร้อมไปต่อไหม

ก่อนขึ้น topic ยาก ควรทำสิ่งเหล่านี้ได้โดยไม่เปิด writeup:

- ☐ ret2win ทั้งแบบมี/ไม่มี argument
- ☐ ret2libc พร้อม leak libc เอง
- ☐ เขียน ROP chain เรียก execve ได้
- ☐ format string leak + GOT overwrite
- ☐ มี exploit template ในหัว ทำ workflow 5 ขั้นได้คล่อง

ถ้ายังไม่ครบ — กลับไปทำโจทย์เพิ่มก่อน อย่ารีบ ระดับ core ต้องแน่นจริง

---

## ระดับถัดไป: Heap Exploitation

นี่คือ tier ที่ควรไปต่อ (และตรงกับที่คุณกำลังทำ picoCTF heap series อยู่)

### ลำดับการเรียน heap

```
1. glibc malloc internals    → chunk, bins ทำงานยังไง
2. Use-After-Free (UAF)      → ใช้ pointer ที่ free ไปแล้ว
3. Double Free               → free ซ้ำ
4. tcache poisoning          → เขียน fd ของ tcache chunk (โจทย์ยุคใหม่เจอเยอะ)
5. fastbin dup               → หลอก allocator ให้คืน chunk ซ้ำ
6. tcache/fastbin → arbitrary write
7. House of ... (Spirit/Force/Orange ฯลฯ) → เทคนิคขั้นสูง
```

### แหล่งเรียน heap ที่ดีที่สุด

- **how2heap** (shellphish/how2heap) — โค้ดตัวอย่างทุกเทคนิค รันดูได้จริง
- **pwn.college** module "Heap Exploitation"
- **Nightmare** (guyinatuxedo) หมวด heap — โจทย์จริงพร้อมเฉลย
- picoCTF heap series (heap0/1, tcache) ที่คุณทำอยู่ — ดีสำหรับปูพื้น

> heap ต้องเข้าใจ glibc malloc ให้ลึกกว่า stack มาก อย่าข้าม internals

---

## topic อื่นๆ ที่ควรรู้

### Canary bypass ขั้นสูง
- leak canary ผ่าน format string / partial overflow
- brute-force canary (กรณี fork server — canary ไม่เปลี่ยน)

### ret2csu
เมื่อหา gadget `pop rdx`/`pop rsi` ไม่เจอ ใช้ `__libc_csu_init` ควบคุม
register แทน — เจอบ่อยในโจทย์ static/medium

### SROP (Sigreturn-Oriented Programming)
ใช้ signal frame ควบคุม register ทุกตัวพร้อมกัน ทรงพลังเมื่อ gadget น้อย

### ret2dlresolve
เรียก dynamic linker resolve ฟังก์ชันเอง เมื่อไม่มี leak และ Full RELRO

### Shellcoding
เขียน shellcode เอง (เมื่อ NX ปิด) — ฝึกเขียน assembly ให้สั้นและเลี่ยง
bad characters

### Kernel pwn
ระดับสูงสุด — exploit ช่องโหว่ใน kernel module / driver

---

## แพลตฟอร์มฝึกต่อ

| แพลตฟอร์ม | เหมาะกับ |
|-----------|---------|
| **pwn.college** | มีโครงสร้าง เรียนเป็นระบบ (แนะนำสุด) |
| **ROP Emporium** | ฝึก ROP โดยเฉพาะ ครบทุกเทคนิค |
| **picoCTF / picoGym** | ไล่ระดับ มี writeup เยอะ |
| **HTB (pwn challenges)** | โจทย์เหมือน CTF จริง |
| **pwnable.kr / pwnable.tw** | โจทย์คลาสสิกคุณภาพสูง (tw ยากขึ้น) |
| **CTFtime** | หา CTF สดๆ มาลงแข่ง |

---

## คำแนะนำสุดท้าย

**ลงแข่ง CTF จริง** — การอ่านหนังสือกับทำโจทย์ฝึกช่วยได้ระดับหนึ่ง แต่
การแข่งจริงภายใต้เวลาจำกัด กับโจทย์ที่ไม่มีเฉลย จะเร่งการเรียนรู้เร็วสุด
แพ้ก็ไม่เป็นไร อ่าน writeup หลังแข่งแล้วทำตามใหม่

**เขียน writeup เอง** — หลังแก้โจทย์ได้ เขียนอธิบายว่าทำยังไง จะทำให้
เข้าใจลึกขึ้นและเป็น portfolio ไปในตัว

**อ่านโค้ดคนอื่น** — ดู exploit ของคนเก่งๆ ใน GitHub เรียนสไตล์การเขียน
และเทคนิคใหม่ๆ

---

## ปิดท้าย

pwn เป็นทักษะที่สร้างจากการสะสมชั่วโมงบิน ไม่มีทางลัดข้ามการลงมือทำ แต่
ถ้าทำ core ในเล่มนี้ให้แน่น + ฝึกสม่ำเสมอ คุณจะไปได้ไกลกว่าที่คิด

ขอให้สนุกกับการ pwn 🚩

---

*จบเล่ม — กลับไป [สารบัญ](../README.md)*
