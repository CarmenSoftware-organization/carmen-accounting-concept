---
title: "อ้างอิงเอกสาร FRD และ mockup ด้วยลิงก์โฟลเดอร์ Google Drive ไม่ใช่ path ในเครื่องหรือลิงก์ไฟล์"
author: Thammanoon
date: 2026-11-09
branch: docs/ar-appendix-trail-setup
cost: { minutes: 1, tokens: 2304 }
id: l-31056c48
scope: repo
status: active
src: agent
touches:
  - docs/PRD-accounting-system-overview.md
---

## Decided
- เอกสาร New Concept (FRD และ mockup ของ AP, AR, GL, Master Data) ย้ายไปอยู่บน shared drive แล้ว ทุก ref ใน repo ต้องลิงก์ไปที่ Drive ห้ามใช้ ../Accounting-docs/ ในเครื่องอีก because the local folder exists only on one machine
- ให้ลิงก์ไปที่โฟลเดอร์ของแต่ละ module ไม่ลิงก์ไปที่ไฟล์ เพราะ (because) FRD ออกเวอร์ชันใหม่บ่อย และเวอร์ชันใหม่เป็นไฟล์ใหม่ ลิงก์ไฟล์จึงค้างอยู่ที่เวอร์ชันเก่า ชื่อไฟล์และเวอร์ชันให้เขียนเป็นข้อความข้างลิงก์
- ไฟล์ review และ spec ต้องเก็บชื่อไฟล์ที่ review ไว้เป็นข้อความ เพราะ (because) เป็นบันทึกย้อนหลังว่า review เวอร์ชันไหน

## Watch out
- URL ต้องไม่มี /u/<n>/ because ส่วนนี้เป็นลำดับ account ในเบราว์เซอร์ของคนที่ copy คนอื่นเปิดแล้วจะเข้า account ผิด
- folder ID หาได้ผ่าน Google Drive search ด้วย parentId; Master Data มีมากกว่า 5 โฟลเดอร์ ต้องดึงหน้าถัดไป (pagination) otherwise จะได้ ID ไม่ครบ

In the code:
- root: https://drive.google.com/drive/folders/1nSKSvjF37mh8EPJTOA3AionmgnsyuczU?usp=drive_link
- ตาราง folder ID: docs/PRD-accounting-system-overview.md Appendix A
