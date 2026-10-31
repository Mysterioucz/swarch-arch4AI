# Tutorial 6 worksheet: serving patterns

**Name / ชื่อ:** Chatrin Yoonchalard

**Student ID / รหัสนิสิต:** 6631304721

This is **individual work**: each student completes and submits their own copy, in their own fork.

_งานนี้เป็น**งานเดี่ยว** นิสิตแต่ละคนทำและส่งงานของตัวเอง โดยใช้ forked repository ของตัวเอง_

Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in Thai or English.

ตอนที่ตอบ ให้นิสิตคัดลอกผลจาก `results/runs.csv` (หรือจาก output ของ load test) ลงในตาราง ตอบสั้นๆ ข้อละ 1–3 ประโยค จะตอบเป็นภาษาไทยหรือภาษาอังกฤษก็ได้

## Task 0: Baseline (copy the baseline results from `results/runs.csv` here / copy ผลลัพธ์ของ baseline จาก `results/runs.csv` มาวางที่นี่)

| Run      | p50 ms | p95 ms | accuracy | large calls |
| -------- | ------ | ------ | -------- | ----------- |
| t0-large | 491    | 587    | 93.5%    | 100%        |
| t0-small | 14     | 31     | 72.5%    | 0%          |

**Q0.** What does the large model buy, and what does it cost?

_large model ให้อะไรเพิ่ม และต้องแลกด้วยอะไร_

> The large model gives higher accuracy (93.5% vs 72.5%), but costs much higher latency (p50 491 ms vs 14 ms), money on every review (100% large calls), and limited capacity (max 20 calls/s).

## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 1 จาก `results/runs.csv` มาวางที่นี่)

| Run       | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
| --------- | ------------ | ------ | ------ | ------ | ------- | -------- |
| t1-before | 10           | 243    | 441    | 20     | 0       | 92.8%    |
| t1-after  | 8.1          | 397    | 1009   | 0      | 28      | 93.0%    |

**Q1.1** In `t1-before`, p50 can look _better_ than the baseline. Why is p50 misleading here? Which column tells the real story?

_ใน `t1-before` ค่า p50 อาจดู "ดีกว่า" baseline ทำไม p50 จึงทำให้เข้าใจผิดในกรณีนี้ และ column ไหนบอกความจริง_

> Because the percentiles only include requests that finished; the 20 hung requests that timed out are left out, so p50 looks fast. The column that tells the real story is `errors` (20 of 200 failed).

**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

_การค้างเกิดกับ calls ไปยัง large model แค่ 10% ทำไมจึงส่งผลต่อระบบแย่กว่าการที่รีวิว 10% ช้าลงเฉยๆ ลองคิดว่า request ที่รออยู่ถือทรัพยากรอะไรไว้ใน review service และ gateway_

> a waiting request keeps holding resources, such as a connection, a worker or a socket, in both the review service and the gateway, for the full 20 s. These pile up until resources run out and healthy requests queue behind them. A timeout releases the held resources so other requests in the queue can use them.

**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

_รีวิวที่ pending ยังไม่มี label ควรจัดการกับรีวิวเหล่านี้อย่างไร และ topology ใดจาก lecture ที่ทำแบบนั้น_

> Pending reviews should be queued and labelled later using batch processing, which scores them in bulk once the large model is available again.

**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

_เราเลือก timeout 1 วินาที จะเกิดปัญหาอะไรถ้าตั้งเป็น 0.1 วินาที และถ้าตั้งเป็น 10 วินาที_

> With 0.1 s, almost every large-model call times out, because a normal call already takes ~491 ms (t0-large p50), so most reviews become pending even when nothing is wrong. With 10 s, hung calls hold connections and workers for too long, so other requests queue up behind them. 1 s sits between the two: it is above normal large-model latency but still short enough to free resources quickly, and it matches the 1-second requirement.

## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 2 จาก `results/runs.csv` มาวางที่นี่)

| Run    | threshold | p50 ms | p95 ms | accuracy | large calls |
| ------ | --------- | ------ | ------ | -------- | ----------- |
| t2-0.6 | 0.6       | 7      | 552    | 91.5%    | 38%         |
| t2-0.7 | 0.7       | 374    | 587    | 93.0%    | 53%         |
| t2-0.8 | 0.8       | 500    | 706    | 93.5%    | 64%         |
| t2-0.9 | 0.9       | 481    | 660    | 93.5%    | 77%         |

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

_จะเลือก threshold ใดสำหรับ deploy ใน CampusEats เพราะอะไร และต้องยอมเสีย quality ใดไป_

> I would deploy 0.6. It has the lowest p50 (7 ms), p95 (552 ms) and share of large calls (38%). The quality I give up is accuracy: 91.5%, only 1.5–2 points below the higher thresholds, which is acceptable for a food-review app in exchange for much faster responses and lower large-model cost.

**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

_ดูค่า p50 และ p95 ของ threshold ที่เลือก ทำไม cascade จึงช่วย p50 ได้มากกว่า p95 มาก_

> With threshold 0.6, 62% of reviews are answered by the small model in a few ms, so the median request (p50 = 7 ms) is a small-model answer. The slowest 5% are always escalated reviews so p95 (552 ms) stays close to all-large (587 ms is t0-large).

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

_cascade เชื่อ confidence ของ small model ยกตัวอย่างหนึ่งสถานการณ์ที่ไม่ควรเชื่อ และผลที่เกิดกับผู้ใช้_

> Negation: for "The soup was not cold at all" the bag-of-words small model sees "cold" and can be confidently negative, so the review never reaches the large model. The owner sees a happy customer counted as unhappy (or the reverse), and nothing in the system flags it, because the error comes with high confidence.

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 3 จาก `results/runs.csv` มาวางที่นี่)

| Run      | p50 ms | p95 ms | accuracy | large calls | cache hits |
| -------- | ------ | ------ | -------- | ----------- | ---------- |
| t3-cache | 9      | 563    | 93.0%    | 38%         | 35%        |

**Q3.1** Which reviews benefit from the cache, and which never will?

_รีวิวแบบไหนได้ประโยชน์จาก cache และแบบไหนไม่มีวันได้ประโยชน์_

> Short common reviews that repeat word for word like "Delicious!" or "Too salty", benefit: in t3-cache 35% of requests were hits and large calls fell from 53% (t2-0.7) to 38%. Long, unique reviews never benefit, and neither does the first occurrence of any text; changing one word or punctuation mark makes a new key.

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

_สัปดาห์หน้าจะ deploy large model ตัวใหม่ที่ดีกว่าเดิม ต้องทำอะไรกับ cache และจะออกแบบรองรับเรื่องนี้อย่างไร_

> The cache must be invalidated, otherwise it keeps serving the old model's labels and hides the new model's improvement. Design for it by putting the model version in the cache key (e.g. `(model_version, text)`) so a deploy starts with a fresh cache automatically, plus a TTL so stale entries expire.

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

_ยกตัวอย่างหนึ่งสิ่งที่ได้เรียนรู้จากการวัดผลครั้งนี้ ที่ไม่มีพูดถึงใน lecture_

> The timeout value becomes the tail latency: after adding the 1 s fallback, p95 in t1-after jumped to 1009 ms, just above the timeout, because every hung call now ends exactly at 1 s.
