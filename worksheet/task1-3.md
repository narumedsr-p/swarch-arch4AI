# Tutorial 6 worksheet: serving patterns

**Name / ชื่อ:** Narumedsr Pitayachamrat

**Student ID / รหัสนิสิต:** 6731328921

This is **individual work**: each student completes and submits their own copy, in their own fork.

*งานนี้เป็น**งานเดี่ยว** นิสิตแต่ละคนทำและส่งงานของตัวเอง โดยใช้ forked repository ของตัวเอง*

Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in Thai or English.

ตอนที่ตอบ ให้นิสิตคัดลอกผลจาก `results/runs.csv` (หรือจาก output ของ load test) ลงในตาราง ตอบสั้นๆ ข้อละ 1–3 ประโยค จะตอบเป็นภาษาไทยหรือภาษาอังกฤษก็ได้

## Task 0: Baseline (copy the baseline results from `results/runs.csv` here / copy ผลลัพธ์ของ baseline จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|
| t0-large | 503 | 617 | 93.5% | 100% |
| t0-small | 18 | 41 | 72.5% | 0% |

**Q0.** What does the large model buy, and what does it cost?

*large model ให้อะไรเพิ่ม และต้องแลกด้วยอะไร*

The large model buys accuracy: 93.5% vs 72.5% for the small model, a ~21 point jump. The cost is latency: p50 jumps from 18ms to 503ms and p95 from 41ms to 617ms, roughly 15-25x slower per request.

## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 1 จาก `results/runs.csv` มาวางที่นี่)

| Run | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
|---|---|---|---|---|---|---|
| t1-before | 26.8 | 234 | 418 | 18 | 0 | 93.4% |
| t1-after | 9.7 | 429 | 1009 | 0 | 17 | 92.9% |

**Q1.1** In `t1-before`, p50 can look *better* than the baseline. Why is p50 misleading here? Which column tells the real story?

*ใน `t1-before` ค่า p50 อาจดู "ดีกว่า" baseline ทำไม p50 จึงทำให้เข้าใจผิดในกรณีนี้ และ column ไหนบอกความจริง*

p50/p95 are only computed over requests that actually completed. The 18 hung calls never return, so they never enter the latency distribution at all: they show up only in the `errors` column. Dropping the worst 18 requests pulls the percentiles down, making the system look fast, while `errors` is the column that tells the real story.

**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

*การค้างเกิดกับ calls ไปยัง large model แค่ 10% ทำไมจึงส่งผลต่อระบบแย่กว่าการที่รีวิว 10% ช้าลงเฉยๆ ลองคิดว่า request ที่รออยู่ถือทรัพยากรอะไรไว้ใน review service และ gateway*

A hung call ties up a connection, a worker thread, and a gateway slot for the entire time it's stuck waiting, and those resources are finite. A steady 10% hang rate keeps eating workers and connections until the pool runs out, and then even requests that never touch the large model start queuing or failing. That's how a small, steady hang rate turns into a system-wide outage.

**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

*รีวิวที่ pending ยังไม่มี label ควรจัดการกับรีวิวเหล่านี้อย่างไร และ topology ใดจาก lecture ที่ทำแบบนั้น*

Pending reviews should go to an offline batch job that retries classification later, without a tight timeout, instead of blocking the user. That's the T4 Asynchronous/batch topology from the lecture: reviews get queued and scored later in bulk rather than inline with the request.

**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

*เราเลือก timeout 1 วินาที จะเกิดปัญหาอะไรถ้าตั้งเป็น 0.1 วินาที และถ้าตั้งเป็น 10 วินาที*

At 0.1s, calls that would have succeeded get cut off before the large model even replies (its own p50 is 503ms), so reviews fall back to pending unnecessarily and accuracy drops. At 10s, the 10% that actually hang still block a worker and connection for the full 10 seconds each, which just brings back the resource-exhaustion problem the timeout was meant to fix, only slower.

## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 2 จาก `results/runs.csv` มาวางที่นี่)

| Run | threshold | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|---|
| t2-0.6 | 0.6 | 8 | 539 | 91.5% | 38% |
| t2-0.7 | 0.7 | 400 | 583 | 93.0% | 53% |
| t2-0.8 | 0.8 | 430 | 589 | 93.5% | 64% |
| t2-0.9 | 0.9 | 475 | 607 | 93.5% | 77% |

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

*จะเลือก threshold ใดสำหรับ deploy ใน CampusEats เพราะอะไร และต้องยอมเสีย quality ใดไป*

t2-0.6. It sends the fewest reviews to the large model (38%) while still clearing QR3's 90% accuracy floor (91.5%), which is exactly what QR4 asks for: minimize large-model calls without breaking QR3. The quality given up is accuracy headroom, about 2 points (91.5% vs. the large model's own 93.5%), in exchange for cutting large-model traffic by 62 points.

**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

*ดูค่า p50 และ p95 ของ threshold ที่เลือก ทำไม cascade จึงช่วย p50 ได้มากกว่า p95 มาก*

At threshold 0.6, p50 drops from 503ms (always-large) to 8ms, but p95 only drops from 617ms to 539ms. The 62% of requests the small model answers confidently finish in under a millisecond, which drags the median way down. The other 38% still have to wait for the large model, which is capped at 4 concurrent calls, so the slow tail of the distribution is still dominated by those escalated requests. Cascade changes how often you pay the large model's latency, not the latency of that path itself.

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

*cascade เชื่อ confidence ของ small model ยกตัวอย่างหนึ่งสถานการณ์ที่ไม่ควรเชื่อ และผลที่เกิดกับผู้ใช้*

The small model is a bag-of-words model, so it can be confidently wrong on negation. For "not bad," it may latch onto "bad" and report high confidence negative, missing that "not" flips the meaning. Because the cascade trusts any confidence above the threshold, this confidently wrong case never reaches the large model for correction. The restaurant owner ends up seeing a happy review mislabeled negative, with no sign it was ever uncertain.

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 3 จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls | cache hits |
|---|---|---|---|---|---|
| t3-cache | 6 | 541 | 93.0% | 38% | 35% |

**Q3.1** Which reviews benefit from the cache, and which never will?

*รีวิวแบบไหนได้ประโยชน์จาก cache และแบบไหนไม่มีวันได้ประโยชน์*

Repeated or common text benefits: the workload's 25% "frequent" short reviews (e.g. "Delicious!", "Too salty") hit the cache on every later occurrence, giving a 35% cache hit rate here and skipping the model entirely. The other 75%, unique free text, never repeats verbatim beyond whitespace or case, so it always misses and always pays the cascade's cost.

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

*สัปดาห์หน้าจะ deploy large model ตัวใหม่ที่ดีกว่าเดิม ต้องทำอะไรกับ cache และจะออกแบบรองรับเรื่องนี้อย่างไร*

The cache must be invalidated (cleared, or at least not trusted) since its stored predictions came from the old model and may now be wrong or stale. I'd version the cache key by model version (e.g. include a model id/version in the key, or just flush `_data` on deploy), so old and new predictions never collide and a rollback can't accidentally serve the wrong model's answers.

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

*ยกตัวอย่างหนึ่งสิ่งที่ได้เรียนรู้จากการวัดผลครั้งนี้ ที่ไม่มีพูดถึงใน lecture*

Across fallback, cascade, and cache, p50 improved a lot but p95 barely moved (e.g. t2-0.6: p50 8ms vs baseline 503ms, but p95 only 539ms vs 617ms). The lecture presented each pattern as a way to cut latency, but the measurements show every pattern still falls back to the same slow, concurrency-limited large model for its "hard" fraction of requests. That makes the tail latency a shared floor none of these patterns alone can lower; only adding capacity to the large model itself would move p95.
