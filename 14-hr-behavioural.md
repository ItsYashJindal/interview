[Home](README.md) | [← 13 Resume Deep-Dive](13-resume-questions.md) | Next: [15 Testing, Git & DevOps →](15-testing-git-devops.md)

# 14 – HR & Behavioural Questions

Keep answers short (1–2 minutes), honest and positive. Use **STAR** for stories: **S**ituation → **T**ask → **A**ction → **R**esult.

## Contents
- [Tell me about yourself](#tell-me-about-yourself)
- [Why did you change jobs / short tenure?](#why-did-you-change-jobs--short-tenure)
- [Why do you want to join us?](#why-do-you-want-to-join-us)
- [Strengths and weaknesses](#strengths-and-weaknesses)
- [Behavioural stories (STAR)](#behavioural-stories-star)
- [Salary, notice period, relocation](#salary-notice-period-relocation)
- [Questions to ask the interviewer](#questions-to-ask-the-interviewer)
- [Interview-day tips](#interview-day-tips)

---

## Tell me about yourself

**Sample (about 90 seconds) – adjust to your style:**

"Hi, I'm Yash. I'm a backend engineer with about 4.5 years of experience, mainly in **Node.js, TypeScript and NestJS**, with PostgreSQL, MongoDB and Redis, deployed on AWS.

Currently I'm at **Notus Services**, working for **Air India**, where I built 8 backend interfaces that sync data between their SAP system and the warehouse management system, and a Next.js dashboard for goods-receipt entry.

Before that, at **Revinfotech**, I built two NestJS microservices – Referral and Staking – for KindSwap, a trading platform with over 100K users. One problem I solved there was duplicate price updates across 4–5 instances, which I fixed with Redis-based leader election.

I spent three years at **Appsinvo**, where I mentored 6 junior developers, introduced Kafka with a dead-letter queue for notifications, integrated 5 payment gateways with idempotency to prevent double payments, and reduced a heavy API from 4.5 seconds to 550 milliseconds.

I enjoy solving reliability and performance problems in backend systems, and I'm looking for a role where I can work on scalable systems and grow toward a senior / lead backend role."

**Tips:** Don't read your resume line by line. Present → past → what you want next. End with something they can ask about (leader election, Kafka, 4.5 s → 550 ms).

---

## Why did you change jobs / short tenure?

> 🎤 **Interview answer:** "(Pick the version that's true for you.) My longest stay was 3 years at Appsinvo, where I grew from developer to mentoring 6 juniors, and that's the kind of long-term growth I'm looking for. After Appsinvo, I wanted to work on larger distributed systems, which I got at Revinfotech on KindSwap, building the Referral and Staking microservices and the leader election for the price service. That was a project-based engagement, and once those services were delivered, the scope reduced *[adjust]*, so I moved to Notus for the Air India SAP–WMS integration, which gave me valuable enterprise integration experience. Now I'm looking for a stable, long-term product engineering team where I can own systems end to end and grow for several years, and that's why this role interests me."

Your timeline: Appsinvo (3 years) → Revinfotech (May–Aug 2026, ~4 months) → Notus (Sep 2026 – present). Interviewers **will** ask. Rules:
- Be honest and brief. Never speak badly about a previous employer.
- Show that the long stay (3 years at Appsinvo) is your normal pattern.
- Focus on what you are looking for now.

**Templates (pick the one that is true for you):**
- *Project-based:* "At Revinfotech I was hired for the KindSwap project. After I delivered the Referral and Staking services, the engagement ended / scope reduced *[adjust]*, so I moved to Notus for the Air India project."
- *Better opportunity:* "After 3 years at Appsinvo I wanted to work on larger distributed systems, which I got at Revinfotech. Notus then offered work on a large enterprise integration for Air India, which was a different and valuable experience."
- *Why leave Notus now (only ~3 months):* "It's a client-project role *[adjust]*. I'm looking for a stable, long-term product/engineering team where I can own systems end to end and grow for several years." Keep it calm and future-focused.

**Never say:** "salary only", "boss was bad", "work was boring".

---

## Why do you want to join us?

> 🎤 **Interview answer:** "(Fill in from your research.) I've read about *[their product]* and saw that you're building it with *[their stack, for example Node.js, NestJS, Kafka, AWS]*. That matches what I've done for the last 4.5 years: building microservices, event-driven systems with Kafka, payment integrations with idempotency, and optimising high-traffic APIs. What excites me is *[their challenge, like scale, reliability or a specific product area]*, because those are exactly the problems I enjoy solving. I'm also looking for a team where I can stay long-term, take ownership and grow toward a senior or lead backend role, and from what I've seen, this is that kind of place."

Research the company first (product, tech stack, recent news). Formula:
"I saw that you're building *[their product]* using *[their stack]*. My experience with *[matching thing – Node/NestJS microservices, Kafka, payments, high-traffic APIs]* fits well, and I want to work on *[their challenge – scale, reliability]*. I'm also looking for a team where I can stay long-term and grow."

---

## Strengths and weaknesses

> 🎤 **Interview answer:** "For strengths, I'd say three things, each with proof. First, debugging and performance: I brought a user-listing API from 4.5 seconds to 550 milliseconds, and I traced and fixed duplicate price updates across 4 to 5 instances with leader election. Second, reliability thinking: I naturally design for failure, with idempotency keys for payments, DLQs for Kafka, and validation, logging and retries in the SAP integration so no data is lost. Third, mentoring and communication: I mentored 6 junior developers and joined client calls for requirements and demos.
>
> For weakness: earlier in my career I tended to take too much on myself and fix things directly instead of delegating. When I started mentoring juniors, I realised that slowed the team's growth, so I learned to break work into clear tasks, assign them, and guide through reviews instead of rewriting their code. Now the team delivers more and I can focus on design. *(Alternative: frontend isn't my strongest area. I can build React and Next.js screens like the GRN dashboard, but I'm much deeper in backend, and I'm improving by building small features end to end.)*"

**Strengths (give proof):**
- Debugging and performance – "4.5 s → 550 ms API; duplicate price updates fixed with leader election."
- Reliability thinking – idempotency, DLQ, validation and logging in integrations.
- Mentoring and communication – mentored 6 juniors, joined client calls.

**Weakness (real but not harmful + what you do about it):**
- "I used to take on too much myself instead of delegating. While mentoring juniors I learned to break tasks down and assign them, and review instead of rewriting."
- or "Frontend is not my strongest area. I can build React/Next.js screens (like the GRN dashboard) but I'm much deeper in backend, and I'm improving by building small features end to end."

---

## Behavioural stories (STAR)

> 🎤 **Interview answer:** "**Disagreement:** At Appsinvo, a teammate wanted to call the email and SMS services directly inside the order API. I explained, with response-time numbers, the latency it added and the risk that an order fails or a message is lost when the provider is down, and I suggested a queue. We agreed to try Kafka for notifications. The order API got faster and messages became reliable with retries and a DLQ. I made it about the data, not about being right *[adjust]*.
>
> **Mistake:** Early at Appsinvo, I shipped a feature with a new query filter without checking indexes. It was fine in testing, but slow in production with real data. I added the right index quickly, and since then I always run `explain()` on new queries with realistic data. I also added an index check to our code review checklist *[adjust to a real incident]*.
>
> **Tight deadline:** When a client needed a feature on a short timeline, I split it into must-haves and nice-to-haves, agreed with the client on what would ship first, delivered in phases, and kept them updated, so there were no surprises at the end *[adjust]*.
>
> **Mentoring:** One junior's pull requests regularly had missing validation and error handling. Instead of just fixing them, I paired with them once on a full feature, shared a small checklist, and reviewed with explanations. Within a few weeks, review comments dropped noticeably and they delivered faster *[adjust]*.
>
> **Learning fast:** When I joined KindSwap, I had to get productive in NestJS and the crypto domain quickly, and at Notus I had to learn SAP and WMS concepts like POs and GRNs. I read the docs, built small prototypes, asked domain experts targeted questions, and delivered real features within the first few weeks *[adjust]*.
>
> **Unclear requirement:** I ask clarifying questions, write down my assumptions and share them, show a quick demo early, and adjust based on feedback, so we find misunderstandings in days, not at the end.
>
> **Difficult client:** I stay calm, confirm the scope in writing, and when new requests come in, I give options with effort and timeline impact, so the client makes an informed choice instead of us silently absorbing scope creep."

Prepare one short story for each. Reuse your technical stories.

| Question | Story to use |
|---|---|
| Hardest technical problem | Redis leader election / 4.5 s API ([13](13-resume-questions.md)) |
| A production issue you handled | Duplicate price updates found via CloudWatch logs; or a payment double-charge risk fixed with idempotency *[adjust]* |
| A time you disagreed with someone | "A teammate wanted to call the email service directly in the order API. I explained the latency and failure risk with numbers and suggested a queue. We agreed to try Kafka for notifications – order API got faster." *[adjust]* |
| A mistake you made | Something real and small: e.g. deployed without a migration / missed an index → what you learned → process change (checklist, review). |
| Tight deadline | How you prioritised the must-haves, communicated with the client, delivered in phases. |
| Mentoring / leadership | Helping a junior improve code quality through reviews and pairing; result (fewer review comments, faster delivery). |
| Learning something new fast | Learning Kafka / NestJS / SAP integration concepts on the job. |
| Handling an unclear requirement | Asked questions, wrote down assumptions, showed a quick demo, adjusted. |
| Working with a difficult client | Stayed calm, clarified scope in writing, gave options with effort estimates. |

**Example – mistake:**
*"Early at Appsinvo I added a feature with a new query filter and didn't check indexes. It was fine in testing but slow in production with real data. I added the right index and since then I always run `explain()` on new queries with realistic data and I added an index check to our review checklist."* *[adjust to a real incident]*

---

## Salary, notice period, relocation

> 🎤 **Interview answer:** "On expected salary: based on my experience, the market for 4 to 5 years in Node.js backend, and the responsibilities of this role, I'm looking for X to Y LPA *[fill your researched range]*, but I'm open to discussing the complete package. On current CTC, I give the true number. On notice period: I'm an immediate joiner *[confirm your exact status and joining date]*. On relocation: yes, I'm open to relocating *[mention preferred cities if asked]*."

- **Expected CTC:** Research market range for 4–5 yrs Node.js backend in your city. Give a range based on that and your current CTC: "Based on my experience and the role, I'm looking for X–Y LPA, but I'm open to discussing the full package."
- **Current CTC:** Be truthful – it may be verified.
- **Notice period:** Your resume says **Immediate Joiner** – be ready to confirm your exact status and joining date.
- **Relocation:** Resume says "Open to relocate" – say yes confidently and mention any preferred cities if asked.

---

## Questions to ask the interviewer

> 🎤 **Interview answer:** "At the end, I'd ask something like: 'What's the biggest technical challenge the team is working on right now?' and 'What would you expect me to achieve in my first 3 months?' Optionally also: 'How do deployments, on-call and code reviews work in the team?' That shows I'm thinking about contributing, not just getting the offer."

Always ask 1–2. Good ones:
- "What does a typical day / sprint look like for this team?"
- "What is the current tech stack and the biggest technical challenge right now?"
- "How do you handle deployments and on-call?"
- "What would you expect me to achieve in the first 3 months?"
- "How is code review and mentoring done in the team?"

Avoid asking about leave/salary in technical rounds.

---

## Interview-day tips

- If you don't know something: *"I haven't used that directly, but here's how I would approach it…"* – then reason it out. Never bluff.
- Think aloud in coding rounds; start with a simple solution, then improve.
- Clarify the question before answering scenario/design questions.
- Bring answers back to your experience.
- Keep a glass of water, test mic/camera, keep the resume open.
- Read the [16 Revision Sheet](16-revision-cheatsheet.md) 1 hour before.

---

[Home](README.md) | [← 13 Resume Deep-Dive](13-resume-questions.md) | Next: [15 Testing, Git & DevOps →](15-testing-git-devops.md)
