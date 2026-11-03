# Tutorial 6 worksheet: serving patterns

**Name / ชื่อ:** ชยุต อาชามงคล

**Student ID / รหัสนิสิต:** 6631307621

This is **individual work**: each student completes and submits their own copy, in their own fork.

*งานนี้เป็น**งานเดี่ยว** นิสิตแต่ละคนทำและส่งงานของตัวเอง โดยใช้ forked repository ของตัวเอง*

Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in Thai or English.

ตอนที่ตอบ ให้นิสิตคัดลอกผลจาก `results/runs.csv` (หรือจาก output ของ load test) ลงในตาราง ตอบสั้นๆ ข้อละ 1–3 ประโยค จะตอบเป็นภาษาไทยหรือภาษาอังกฤษก็ได้

## Task 0: Baseline (copy the baseline results from `results/runs.csv` here / copy ผลลัพธ์ของ baseline จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|
| t0-large | 487 | 582 | 93.5% | 100% |
| t0-small | 17 | 41 | 72.5% | 0% |

**Q0.** What does the large model buy, and what does it cost?

*large model ให้อะไรเพิ่ม และต้องแลกด้วยอะไร*

> ได้ accuracy เพิ่มประมาณ 21 จุด (93.5% vs 72.5%) แต่ช้ากว่าเยอะ (p50 487 vs 17 ms) และทุก review คือ 1 paid call (large calls 100%) ซึ่งยังมี cap ที่ 20 calls/s อีก

## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 1 จาก `results/runs.csv` มาวางที่นี่)

| Run | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
|---|---|---|---|---|---|---|
| t1-before | 32.8 | 230 | 414 | 22 | 0 | 92.7% |
| t1-after | 9.6 | 409 | 1008 | 0 | 22 | 95.5% |

**Q1.1** In `t1-before`, p50 can look *better* than the baseline. Why is p50 misleading here? Which column tells the real story?

*ใน `t1-before` ค่า p50 อาจดู "ดีกว่า" baseline ทำไม p50 จึงทำให้เข้าใจผิดในกรณีนี้ และ column ไหนบอกความจริง*

> p50 นับแค่ request ที่สำเร็จ ส่วน 22 ตัวที่ค้างกลายเป็น error แล้วหลุดจาก latency ไปเลย และ call ที่ค้างไม่ได้ใช้ slot ของ large model ตัวอื่นเลยไม่ต้องต่อคิวนาน (230 vs 487 ms) column ที่บอกความจริงคือ `errors` (22/200 = 11%) กับ run time (32.8 s vs 9.9 s)

**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

*การค้างเกิดกับ calls ไปยัง large model แค่ 10% ทำไมจึงส่งผลต่อระบบแย่กว่าการที่รีวิว 10% ช้าลงเฉยๆ ลองคิดว่า request ที่รออยู่ถือทรัพยากรอะไรไว้ใน review service และ gateway*

> request ที่ค้างไม่ fail แต่ถือ connection กับ task ค้างไว้ทั้งใน review service (client ตั้ง timeout=None เลยรอเต็ม 20 s) และ gateway ที่ต้องรอ review service ต่อ ทำให้ concurrent slot เต็มไปด้วยตัวที่ค้าง ตัวที่ปกติก็ต้องรอ slot ไปด้วย run เลยกินเวลา 32.8 s ทั้งที่ค้างแค่ 10%

**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

*รีวิวที่ pending ยังไม่มี label ควรจัดการกับรีวิวเหล่านี้อย่างไร และ topology ใดจาก lecture ที่ทำแบบนั้น*

> เก็บ review ไว้ก่อน แล้วให้ batch job มาดึงจาก `/reviews/pending` ไป score ทีหลังตอน inference service กลับมาปกติ topology นี้คือ Asynchronous / batch (เก็บหรือเข้าคิวไว้ แล้วติด label ทีหลังเป็นชุด)

**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

*เราเลือก timeout 1 วินาที จะเกิดปัญหาอะไรถ้าตั้งเป็น 0.1 วินาที และถ้าตั้งเป็น 10 วินาที*

> 0.1 s สั้นกว่าเวลาปกติของ large model (~200 ms) เลย call ปกติก็ timeout เกือบหมด --> ส่วนใหญ่กลายเป็น pending ทั้งที่ยังจ่ายเงินค่า call อยู่ ส่วน 10 s จะรอ call ที่ค้างนานเกือบเท่า `t1-before` และเกิน QR2 ที่รอได้ไม่เกิน 1 s

## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 2 จาก `results/runs.csv` มาวางที่นี่)

| Run | threshold | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|---|
| t2-0.6 | 0.6 | 5 | 537 | 91.5% | 38% |
| t2-0.7 | 0.7 | 367 | 573 | 93.0% | 53% |
| t2-0.8 | 0.8 | 501 | 675 | 93.5% | 64% |
| t2-0.9 | 0.9 | 474 | 620 | 93.5% | 77% |

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

*จะเลือก threshold ใดสำหรับ deploy ใน CampusEats เพราะอะไร และต้องยอมเสีย quality ใดไป*

> เลือก 0.7 เพราะ accuracy 93.0% ใกล้ large (93.5%) และสูงกว่า QR3 (90%) แต่ส่งไป large แค่ 53% ส่วน 0.8 กับ 0.9 ได้ accuracy เพิ่มแค่ 0.5 จุด แต่ cost เพิ่มเป็น 64% และ 77% ส่วน 0.6 ถูกกว่า (38%) แต่ 91.5% เหลือ margin เหนือ 90% แค่ 1.5 จุด (ยอมเสีย accuracy ไปประมาณ 0.5 จุดเทียบกับ large)

**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

*ดูค่า p50 และ p95 ของ threshold ที่เลือก ทำไม cascade จึงช่วย p50 ได้มากกว่า p95 มาก*

> ที่ 0.7 p50 ลดจาก 487 เหลือ 367 ms แต่ p95 573 ms แทบเท่า large (582 ms) เพราะ request ที่ช้าสุด 5% คือตัวที่ escalate ไป large ซึ่งยังเสีย ~200 ms + ต่อคิวเหมือนเดิม ปล. p50 จะลดเยอะจริง ๆ ตอน escalate น้อยกว่าครึ่ง (0.6: 38% --> p50 5 ms)

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

*cascade เชื่อ confidence ของ small model ยกตัวอย่างหนึ่งสถานการณ์ที่ไม่ควรเชื่อ และผลที่เกิดกับผู้ใช้*

> small model ดูแค่คำเดี่ยว ๆ ประโยคปฏิเสธอย่าง "not cold" หรือประโยคที่มี "but" เลยอาจ confident แต่ตอบผิด --> cascade ไม่ escalate --> label ผิดไปถึงเจ้าของร้าน เช่น review ที่ไม่พอใจถูกติดเป็น positive

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 3 จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls | cache hits |
|---|---|---|---|---|---|
| t3-cache | 6 | 557 | 93.0% | 38% | 35% |

**Q3.1** Which reviews benefit from the cache, and which never will?

*รีวิวแบบไหนได้ประโยชน์จาก cache และแบบไหนไม่มีวันได้ประโยชน์*

> ได้ประโยชน์คือ review ที่ข้อความซ้ำกัน เช่นประโยคสั้น ๆ ที่คนเขียนเหมือนกันเยอะ (หลัง normalise ตัวพิมพ์กับช่องว่าง) hit 35% ทำให้ large calls ลดจาก 53% (`t2-0.7`) เหลือ 38% ส่วน review ที่ข้อความไม่ซ้ำใคร และครั้งแรกของทุกข้อความ จะ miss เสมอ เลยไม่ช่วย p95 (573 --> 557 ms)

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

*สัปดาห์หน้าจะ deploy large model ตัวใหม่ที่ดีกว่าเดิม ต้องทำอะไรกับ cache และจะออกแบบรองรับเรื่องนี้อย่างไร*

> ต้องล้าง cache เพราะข้างในยังเป็นคำตอบของ model ตัวเก่า ถ้าไม่ล้างก็จะ serve คำตอบเก่าต่อ ออกแบบให้ใส่ model version เข้าไปใน cache key (version + normalised text) ตัวใหม่เริ่มจาก miss เอง ส่วน entry เก่าก็ค่อยหลุดไปตาม LRU

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

*ยกตัวอย่างหนึ่งสิ่งที่ได้เรียนรู้จากการวัดผลครั้งนี้ ที่ไม่มีพูดถึงใน lecture*

> p50 ดูดีได้ทั้งที่ระบบพัง: `t1-before` p50 230 ms ดีกว่า `t0-large` (487 ms) แต่มี error 11% เพราะ request ที่ fail ไม่ถูกนับใน latency อีกอย่างคือ cascade ช่วย p50 เยอะก็ต่อเมื่อ escalate น้อยกว่าครึ่ง (0.6 ได้ 5 ms, 0.7 ได้ 367 ms)
