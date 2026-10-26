# BARO, the Simple Version

A practice guide for `BARO Research Presentation.odp`. It follows the 12 slides in order and uses the same words the slides use. The deep version is `BARO Paper Walkthrough.md`; you don't need it to present.

---

## The whole talk is five sentences

Practice these out loud with no slides. Every slide is just one of these with evidence attached.

1. When a big app breaks, you need to know **when** it broke and **what** broke it. *(slides 1 to 2)*
2. Most tools assume they know the "when" exactly. In real life it's usually a little off. *(slide 3)*
3. BARO compares each gauge to its **middle value** instead of its **average**, so a wrong "when" doesn't throw it off. *(slides 4 to 6)*
4. On the big test app, BARO found the culprit far more often than the map-based tool: **0.81 vs 0.28**. *(slides 7 to 9)*
5. The honest catch: give the old simple tool the exact time and it does just as well. BARO wins because in real life you never have the exact time. *(slides 10 to 12)*

## The analogy that runs through the talk: a restaurant kitchen

| On the slides | Kitchen version |
|---|---|
| Services (64 in the big app) | Cooking stations: grill, fryer, salad, dessert |
| Gauges | Each station's readings: how hot, how busy, how many orders are late |
| A failure | Food coming out slow or wrong |
| Step 1: find when it broke | The head chef noticing "something's off, it started around 7:15" |
| Step 2: rank the gauges | Working out **which station** caused it |
| A late start time | The chef says 7:20 when it really started at 7:15 |

---

## One analogy per slide, at a glance

| Slide | Analogy |
|---|---|
| 1. Title | A restaurant kitchen: find which station is slowing down the orders |
| 2. Why I chose it | A detective who trusts a wrong time of death |
| 3. The problem | A car alarm that goes off for every cat |
| 4. How it works | The hallway smoke alarm goes off first, but the fire is in the kitchen |
| 5. Step 1 | Black Friday: busy is not broken |
| 6. Step 2 | Bill Gates walks into a bar |
| 7. The test | A teacher who writes the test has the answer key |
| 8. Results | "Who broke the vase?": BARO gets it right about 8 times out of 10 |
| 9. Robustness | "Who broke the vase?" with the wrong time: BARO still picks the right kid |
| 10. Caveats | The fine print on the vase game |
| 11. Reflection | Instant triage, slow lab results |
| 12. The claim | Bill Gates again: use the middle, not the average |

---

## Slide by slide

Each slide has the analogy (in the box), then the script to say. *[Italic brackets]* tell you what to point at on the slide. The same script is in the deck's speaker notes.

### 1. Title (15 sec)
> **Analogy:** Picture a restaurant kitchen. Each service is a cooking station. When orders start coming out slow, BARO helps the head chef find which station is causing it.

**Script:**

Hi, I'm Aaron. My paper is BARO, from FSE 2024, by Luan Pham, Huong Ha, and Hongyu Zhang.

*[Gesture at the title]* Picture a restaurant kitchen where orders start coming out slow. BARO helps you figure out when it started and which station is causing it.

### 2. Why I chose this paper (50 sec)
> **Analogy:** A detective show assumes the time of death is exact. In real life it is often off, and a case built on that exact time falls apart.

**Script:**

*[Card 01]* I chose this paper because of one problem: alarms are often late. Most tools assume they know exactly when a failure started. It's like a detective who completely trusts the time of death. If that time is wrong, the whole case falls apart.

*[Card 02]* It's also cheap to try. It's simple math on monitoring data most teams already collect. No AI training.

*[Card 03]* It was tested on a big app with 64 services, not just a small demo.

*[Card 04]* And anyone can check it. The code and data are public, and the results were independently verified.

*[Your own line]* (Add one sentence from your own experience here, like an alert that fired late.)

### 3. The problem (50 sec)
> **Analogy:** Simple alarms are a car alarm that goes off for every cat. Map-based tools are like drawing a family tree first: easy for 12 people, a mess for 64.

**Script:**

So why do today's tools struggle?

*[Left box]* Simple alarms cry wolf. They're like a car alarm that goes off every time a cat walks by. They catch real problems, but they also go off when nothing is wrong.

*[Middle box]* Map-based tools first draw a map of which services talk to which. That's like drawing a family tree: easy with 12 people, a mess with 64. On the big app, the map often never gets finished.

*[Right box]* And here's the real problem. They all trust the clock. If the alarm time is a little late, some broken data gets counted as normal, and the answer comes out wrong. That's what BARO is built to fix.

### 4. How it works (40 sec)
> **Analogy:** The hallway smoke alarm goes off first, but the fire is in the kitchen. The first change tells you when, not who.

**Script:**

BARO works in two steps: first when, then what.

*[Box 1]* It reads the gauges: speed, errors, CPU, and memory for every service. *[Box 2]* It finds the moment things broke. *[Box 3]* It scores every gauge by how far it moved. *[Box 4]* And it hands over a list, biggest movers on top, so the engineer knows where to look first.

*[Bottom line]* One important thing: the first change only tells you when it broke, not who broke it. The hallway smoke alarm goes off first, but the fire is in the kitchen.

### 5. Step 1: spot the moment things changed (50 sec)
> **Analogy:** On Black Friday the kitchen is slammed, but food still comes out on time. That is busy, not broken.

**Script:**

Step one: spot the moment things changed.

*[Big text]* The key rule is: busy is not broken. *[Black Friday card]* Think Black Friday. The kitchen is slammed, but food still comes out on time. That's fine. BARO only cares about two things: is it slow, and is it throwing errors?

*[Paragraph on the left]* It also watches those signals together, not one at a time. Like a teacher who doesn't react to one loud kid, but notices when the whole room's mood changes.

*[First change card]* The first moment things shift becomes the start time. *[Nothing to tune card]* And there's nothing to set by hand. The paper calls this multivariate Bayesian online change point detection. That's all you need to know about the name.

### 6. Step 2: rank the gauges (75 sec)
> **Analogy:** Ten people in a bar earn about $50k. Bill Gates walks in. The average jumps to billions. The middle person still earns about $50k.

**Script:**

*(The takeaway: even when the alarm goes off late, BARO still points at the right culprit.)*

Once BARO knows when things broke, it has to figure out what broke.

*[The three boxes]* For every gauge, it does three things. It learns the gauge's normal level. It checks how far the gauge jumped after the break. And it puts the biggest jumps at the top of the list.

For example, say the payment service's memory normally sits around 40 percent and jumps to 95. That's a huge jump, so payment goes to the top of the list.

*[Graph]* Here's the catch. This blue line is one gauge over time. The black line is when it really broke. The red dashed line is when the alarm went off, a little late. BARO treats everything before the red line as normal, so a few broken readings get mixed into normal.

*[Dark box]* If you use the average, those broken readings pull normal up. Then the real jump looks smaller, and the real culprit can slide down the list. It's like Bill Gates walking into a bar: the average salary shoots up, but the middle salary doesn't move. BARO uses the middle, so the real culprit stays on top.

So the point of this slide: a late alarm can trick the old tools into blaming the wrong service. BARO uses the middle value, so it still blames the right one.

*(Do not read the symbols in the graph's corners.)*

**If the takeaway still feels fuzzy, this example shows it (made-up numbers, for you, not for the slide):**

| | Normal | After the break | Jump |
|---|---|---|---|
| Payment memory (real culprit) | 40 | 95 | **55** |
| Cart CPU (innocent) | 30 | 50 | 20 |

Payment jumped more, so it is #1. Correct. Now the alarm goes off late, and some of payment's 95s get counted as "normal":

- **Old tools (average):** payment's "normal" gets dragged up to about 80, so its jump looks like 15. Cart's 20 beats it. **The list blames cart. Wrong.**
- **BARO (middle value):** a few 95s barely move the middle, so normal stays about 40 and the jump stays 55. **Payment stays #1. Right.**

### 7. The test (40 sec)
> **Analogy:** A teacher who writes the test also has the answer key. Breaking things on purpose gave the authors the answer key.

**Script:**

To test this, the authors built three apps. *[First two cards]* Two small online stores, *[Train Ticket card]* and one big train booking app with 64 services.

*[Numbers row]* Then they broke them on purpose, 100 times per app, in four ways: overloading the CPU, leaking memory, slowing the network, and dropping data.

Because they broke things themselves, they always knew the right answer. That gave them an answer key to grade BARO and the other tools.

*[Bottom line]* One detail: they restarted each app fresh after every test. I'll come back to that.

### 8. Results (65 sec)
> **Analogy:** A game of "Who broke the vase?" in a big house full of kids. BARO points at the right kid about 8 times out of 10. The map-based tool, about 3 out of 10.

**Script:**

Here's the report card. Think of it as a game of "Who broke the vase?" Something breaks, and each tool points at who it thinks did it.

*[BARO culprit score row, Train Ticket column]* In the big app, BARO points at the right culprit about 8 times out of 10. That's the 0.81.

*[Point at the orange 0.28]* The map-based tool gets it right only about 3 times out of 10.

*[Old simple tool row]* The old simple tool does almost as well as BARO here, about 7 or 8 out of 10. Its weakness shows up on the next slide, when the alarm is late.

*[Boutique and Sock Shop columns]* In the small apps, everyone does fine. The big app is where BARO pulls ahead.

*[BARO alarm score row]* The top row is a different question: did it notice something broke at all? BARO caught every failure, with fewer false alarms than the others.

*(Say "about" 8 out of 10. The scores give partial credit for being near the top of the list, so they are not an exact count.)*

### 9. Move the clock and BARO holds still (50 sec)
> **Analogy:** Play "Who broke the vase?" again, but give the detectives the wrong time it broke. BARO still picks the right kid. The others get confused.

**Script:**

This slide answers one question: what if the alarm time is wrong?

*[Graph]* The authors gave every tool the wrong time on purpose, sometimes early, sometimes late. It's like playing "Who broke the vase?" but telling the detectives the wrong time it broke.

*[Point at the pink line]* BARO is the pink line. It stays flat. A wrong time doesn't fool it.

*[Point at the blue and black dashed lines on the right side]* These two tools use the average. When the time is late, their lines drop. They start blaming the wrong kid.

*[Third card]* And here's the proof that the middle value is the secret: when BARO used the average instead, it dropped from about 8 out of 10 to about 3 out of 10.

*(Only look at the pink line and the two dashed lines. The three charts tell the same story, and the green and yellow lines are other tools you can ignore.)*

### 10. What I would not overclaim (45 sec)
> **Analogy:** BARO won the vase game, but every win has fine print. This slide is the fine print.

**Script:**

BARO did well, but here's the fine print.

*[Perfect clock card]* One: if you tell the detectives the exact right time, the old simple tool does just as well as BARO. BARO only wins when the time is wrong. In real life, it usually is.

*[Close race card]* Two: BARO is great at pointing to the right station, like "it's the grill." Pointing to the exact dial on that station, like "the grill's temperature," is a closer race with the other tools.

*[15 of 100 card]* Three: the map-based detective needs a map of the house first. In the big app, it couldn't draw one 85 times out of 100. So that wasn't a totally fair fight.

*[Lab conditions card]* Four: this was a practice game. They broke things on purpose, one at a time, and reset everything after each round. Real life is messier.

### 11. A to-do list, not a self-healing system (60 sec)
> **Analogy:** The triage is instant, but the lab results take a while. Ranking takes 0.01 seconds; spotting the failure takes up to 7 minutes.

**Script:**

So would I use it? Yes, but as a to-do list for a person, not a system that fixes itself.

*[Ready now box]* What's ready now: the ranking takes about a hundredth of a second. There's nothing to train and no map to maintain. It still works when the clock is off. And the code is public.

*[Not yet box]* What's not ready: spotting the failure takes thirty seconds to seven minutes. It's like instant triage but slow lab results. And it was only tested in a clean lab. Restarting after every test is like crash-testing a car on a fresh track every time. It never had to survive a muddy week-long road trip.

*[Bottom line]* What I'd add: logs and traces, which the authors plan too; a flag when several services break together; and a test on a system that isn't reset after every failure.

### 12. The claim I would defend (30 sec)
> **Analogy:** Back to Bill Gates: use the middle, not the average, and the list stays right when the clock is wrong.

**Script:**

*[Read the slide]* Watching the signals together means fewer false alarms. Using the middle value keeps the list right when the clock is off. That is what still works when the system gets big.

*[Pause]* Thank you. I'm happy to take questions.

---

## If you blank, remember these four things

1. **The problem:** tools assume they know exactly *when* it broke. They usually don't.
2. **The fix:** use the middle, not the average (Bill Gates in the bar).
3. **The number:** 0.81 vs 0.28 on the big app.
4. **The honest catch:** with a perfect clock, the old simple tool ties it.

---

## Technical names, only if someone asks

The slides avoid these on purpose. You don't need to say them unprompted.

| On the slides | The paper calls it |
|---|---|
| Alarm score | F1 |
| Culprit score | Avg@5 |
| Map-based tool | CausalRCA |
| Old simple tool | N-Sigma (a z-score: distance from the average) |
| Middle value / how spread out normal is | Median / IQR (interquartile range) |
| Gauges | Metrics |
| Root cause | The service (or gauge) that actually broke |
| The three test apps | Online Boutique, Sock Shop, Train Ticket |

## Likely questions, one-line answers

| Question | Answer |
|---|---|
| Isn't this just a z-score with the median? | Basically, for step 2. That's the point: a small change makes it hold up when the time is off. |
| Why not just use the old simple tool, since it ties with the exact time? | You never have the exact time in real life. With a guessed time it drops: 0.30 vs 0.81 on the big app. |
| Does it fix the problem? | No. It hands a human a ranked list of where to look. |
| What if a failure never makes things slow or throw errors? | BARO won't catch it. The paper says so. |
| Why does the big app matter so much? | It's where the map-based tools break down and BARO doesn't. |
| How fast is it? | Ranking: about 0.01 seconds. Spotting the failure: 30 seconds to about 7 minutes. |
| Can I trust the results? | The code and data are public, and ACM gave it a "Results Validated" badge. |
