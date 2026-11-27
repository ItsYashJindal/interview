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

> 🎤 **Interview answer:** "Hi, I'm Yash. I'm a backend engineer with about 4.5 years of experience, mainly in **Node.js, TypeScript and NestJS**, with PostgreSQL, MongoDB and Redis, on AWS.
>
> Right now I'm at **Notus Services**, working for **Air India**. I built 8 backend interfaces that keep data in sync between their SAP system and their warehouse system, and a Next.js dashboard for entering goods receipts.
>
> Before that, at **Revinfotech**, I built two NestJS microservices, Referral and Staking, for KindSwap, a trading platform with over 100K users. One problem I solved there: users were getting duplicate price updates from 4–5 servers. I fixed it with leader election in Redis.
>
> I spent three years at **Appsinvo**. There I mentored 6 junior developers, introduced Kafka with a dead-letter queue for notifications, integrated 5 payment gateways with protection against double payments, and made a slow API go from 4.5 seconds to 550 ms.
>
> I enjoy solving reliability and performance problems, and I'm looking for a role where I can work on systems that scale, and grow toward a senior or lead backend role."

**Tips:** Don't read your resume line by line. Go: now → past → what you want next. End with something they'll want to ask about (leader election, Kafka, 4.5 s → 550 ms).

---

## Why did you change jobs / short tenure?

Your timeline: Appsinvo (3 years) → Revinfotech (May–Aug 2026, ~4 months) → Notus (Sep 2026 – present). Interviewers **will** ask.

**Rules:**
- Be honest and short. Never speak badly about a previous company.
- Show that the long stay (3 years at Appsinvo) is your normal pattern.
- Talk about what you are looking for **now**.

> 🎤 **Interview answer:** "(Pick the version that's true for you.) My longest stay was 3 years at Appsinvo, where I grew from developer to mentoring 6 juniors. That's the kind of long-term growth I want.
>
> After Appsinvo, I wanted to work on bigger distributed systems, and I got that at Revinfotech on KindSwap. That was a project-based role, and once the Referral and Staking services were delivered, the scope reduced *[adjust]*. So I moved to Notus for the Air India project, which gave me valuable enterprise integration experience.
>
> Now I'm looking for a stable, long-term product team where I can own systems end to end and grow for several years. That's why this role interests me."

**Never say:** "only for salary", "my boss was bad", "the work was boring".

---

## Why do you want to join us?

Research the company first (product, tech stack, recent news).

> 🎤 **Interview answer:** "(Fill in from your research.) I've read about *[their product]* and saw that you use *[their stack, e.g. Node.js, NestJS, Kafka, AWS]*. That matches what I've done for the last 4.5 years: microservices, event-driven systems with Kafka, payment integrations, and making busy APIs fast. What excites me is *[their challenge – scale, reliability, a product area]*, because those are exactly the problems I enjoy. I'm also looking for a team where I can stay long-term, take ownership and grow toward a senior or lead role, and this looks like that kind of place."

---

## Strengths and weaknesses

> 🎤 **Interview answer – strengths (always give proof):**
> 1. "**Debugging and performance** – I made a user-listing API go from 4.5 seconds to 550 ms, and I found and fixed duplicate price updates across 4–5 servers with leader election.
> 2. **Thinking about failures** – I naturally design for things going wrong: idempotency keys for payments, dead-letter queues for Kafka, and validation, logging and retries in the SAP integration so no data is lost.
> 3. **Mentoring and communication** – I mentored 6 junior developers and joined client calls for requirements and demos."

> 🎤 **Interview answer – weakness (real, not harmful, + what you do about it):**
> "Earlier in my career, I took too much work on myself and fixed things directly instead of delegating. When I started mentoring juniors, I saw this slowed the team down. So I learned to break work into clear tasks, assign them, and guide through reviews instead of rewriting their code. Now the team delivers more, and I can focus on design."
>
> *(Alternative: "Frontend is not my strongest area. I can build React and Next.js screens, like the GRN dashboard, but I'm much deeper in backend. I'm improving by building small features end to end.")*

---

## Behavioural stories (STAR)

Prepare one short story for each. Reuse your technical stories.

**Disagreement with a teammate**

> 🎤 **Interview answer:** "At Appsinvo, a teammate wanted to call the email and SMS services directly inside the order API. I showed, with response-time numbers, how much slower it made the API, and the risk that an order fails or a message is lost when the provider is down. I suggested a queue. We agreed to try Kafka for notifications. The order API got faster, and messages became reliable with retries. I kept it about the data, not about being right *[adjust]*."

**A mistake you made**

> 🎤 **Interview answer:** "Early at Appsinvo, I shipped a feature with a new query filter without checking the indexes. It was fine in testing, but slow in production with real data. I added the right index quickly. Since then, I always run `explain()` on new queries with realistic data, and I added an index check to our code review checklist *[adjust to a real incident]*."

**Tight deadline**

> 🎤 **Interview answer:** "When a client needed a feature quickly, I split it into must-haves and nice-to-haves, agreed with the client what would ship first, delivered in phases, and kept them updated, so there were no surprises at the end *[adjust]*."

**Mentoring**

> 🎤 **Interview answer:** "One junior's code often had missing validation and error handling. Instead of just fixing it myself, I sat with them once on a full feature, shared a small checklist, and explained my review comments. Within a few weeks, review comments dropped a lot and they delivered faster *[adjust]*."

**Learning something new fast**

> 🎤 **Interview answer:** "At KindSwap I had to get productive in NestJS and the crypto domain quickly, and at Notus I had to learn SAP and warehouse ideas like POs and GRNs. I read the docs, built small test projects, asked domain experts specific questions, and delivered real features in the first few weeks *[adjust]*."

**Unclear requirement**

> 🎤 **Interview answer:** "I ask clarifying questions, write down my assumptions and share them, show a quick demo early, and adjust based on feedback. So we find misunderstandings in days, not at the end."

**Difficult client**

> 🎤 **Interview answer:** "I stay calm, confirm the scope in writing, and when new requests come, I give options with the effort and timeline impact. So the client makes an informed choice, and we don't silently take on extra work."

| Question | Story to use |
|---|---|
| Hardest technical problem | Redis leader election / 4.5 s API ([13](13-resume-questions.md)) |
| A production issue you handled | Duplicate price updates found in CloudWatch logs; or double-payment risk fixed with idempotency *[adjust]* |
| Disagreement | Email/SMS inside the order API → Kafka (above) |
| A mistake | Missed index → learned to always run `explain()` (above) |
| Tight deadline | Must-haves first, phased delivery |
| Mentoring / leadership | Junior's code quality improved through pairing and reviews |
| Learning fast | NestJS / crypto / SAP concepts |
| Unclear requirement | Questions, written assumptions, early demo |
| Difficult client | Calm, scope in writing, options with effort |

---

## Salary, notice period, relocation

> 🎤 **Interview answer:** "**Expected salary:** based on my experience, the market for 4–5 years in Node.js backend, and this role's responsibilities, I'm looking for X to Y LPA *[fill your researched range]*, but I'm open to discussing the full package.
> **Current CTC:** I give the true number.
> **Notice period:** I'm an immediate joiner *[confirm your exact status and joining date]*.
> **Relocation:** yes, I'm open to relocating *[mention preferred cities if asked]*."

- Research the market range for your city before the interview.
- Be truthful about current CTC – it may be checked.

---

## Questions to ask the interviewer

> 🎤 **Interview answer:** "At the end I'd ask: 'What's the biggest technical challenge the team is working on right now?' and 'What would you expect me to achieve in my first 3 months?' This shows I'm thinking about contributing, not just getting the offer."

Always ask 1–2. Good ones:
- "What does a typical day / sprint look like for this team?"
- "What is the tech stack, and the biggest technical challenge right now?"
- "How do you handle deployments and on-call?"
- "What would you expect me to achieve in the first 3 months?"
- "How are code reviews and mentoring done in the team?"

Don't ask about leave or salary in technical rounds.

---

## Interview-day tips

- If you don't know something: *"I haven't used that directly, but here's how I would approach it…"* – then think it through out loud. Never bluff.
- In coding rounds, think out loud. Start with a simple solution, then improve it.
- For scenario / design questions, ask a clarifying question first.
- Bring answers back to your own experience.
- Keep water nearby, test mic/camera, keep your resume open.
- Read the [16 Revision Sheet](16-revision-cheatsheet.md) 1 hour before.

---

[Home](README.md) | [← 13 Resume Deep-Dive](13-resume-questions.md) | Next: [15 Testing, Git & DevOps →](15-testing-git-devops.md)
