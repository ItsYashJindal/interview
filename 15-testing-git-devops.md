[Home](README.md) | [← 14 HR & Behavioural](14-hr-behavioural.md) | Next: [16 Revision Sheet →](16-revision-cheatsheet.md)

# 15 – Testing, Git, DevOps & Linux (added)

## Contents
- [Jest and testing](#jest-and-testing)
- [Git](#git)
- [PM2 and Nginx](#pm2-and-nginx)
- [CI/CD](#cicd)
- [Linux commands](#linux-commands)
- [Clean code and design principles](#clean-code-and-design-principles)

---

## Jest and testing

**Q. Types of tests**

> 🎤 **Interview answer:** "There are three main types:
> - **Unit tests** – test **one function** alone, with everything else replaced by fakes. Very fast. Most tests should be unit tests. Example: testing a discount calculation.
> - **Integration tests** – test **several parts together**, like an API endpoint with a real test database, using supertest.
> - **End-to-end (E2E) tests** – test the **whole system like a real user**, clicking through the app with Cypress or Playwright. Slowest, so only a few, for the most important flows.
>
> This is the 'testing pyramid': many unit tests, fewer integration tests, very few E2E. In my projects I wrote Jest unit tests for services, with fake DB models and gateways, plus supertest and Postman for API flows."

| Type | Tests | Speed | Example |
|---|---|---|---|
| Unit | One function alone, others faked | Fast | `calculateDiscount()` |
| Integration | Several parts together (API + DB) | Medium | `POST /orders` with a test DB |
| E2E | Whole system like a user | Slow | Cypress / Playwright |

**Basic Jest test:**
```js
// discount.js
export const applyDiscount = (price, pct) => {
  if (pct < 0 || pct > 100) throw new Error("Invalid percent");
  return price - (price * pct) / 100;
};

// discount.test.js
describe("applyDiscount", () => {
  it("applies percentage", () => {
    expect(applyDiscount(1000, 10)).toBe(900);
  });

  it("throws for invalid percent", () => {
    expect(() => applyDiscount(100, 120)).toThrow("Invalid percent");
  });
});
```

**Mocking:**

> 🎤 **Interview answer:** "Mocking means **replacing real things with fakes** in tests, like the database, payment gateway or email service. So tests are fast, always give the same result, and only test my logic.
> - `jest.fn()` – a fake function. I can check if it was called, and with what.
> - `jest.mock()` – replace a whole module, like the User model, and decide what it returns.
> - `jest.spyOn()` – watch or change one method on a real object. For example, make the payment client fail, to test the error path.
>
> I reset fakes before each test, and use fake timers to test debounce or retry logic without actually waiting."

```js
// Replace the whole User model
jest.mock("../models/User");
User.findById.mockResolvedValue({ _id: "1", name: "Yash" });

// Fake function – check how it was called
const sendEmail = jest.fn();
expect(sendEmail).toHaveBeenCalledWith(expect.objectContaining({ to: "a@x.com" }));

// Make one method fail
jest.spyOn(paymentClient, "createOrder").mockRejectedValue(new Error("Gateway down"));
```
- `beforeEach` / `afterEach` – setup and reset (`jest.clearAllMocks()`).
- Async: `await expect(fn()).rejects.toThrow()`.
- Timers: `jest.useFakeTimers()` to test debounce/retry without waiting.

**API integration test with supertest:**

> 🎤 **Interview answer:** "Supertest sends real HTTP requests to my Express app (or `app.getHttpServer()` in NestJS) without starting a real server. So it tests routing, middleware, validation, controller and service together. I run these on a **separate test database** and clean it between tests, so tests don't affect each other."

```js
const request = require("supertest");
const app = require("../app");

it("GET /health returns 200", async () => {
  const res = await request(app).get("/health");
  expect(res.status).toBe(200);
});
```

**Q. What is code coverage?**

> 🎤 **Interview answer:** "Code coverage is the **percentage of code that runs during tests** (`jest --coverage`). It helps find parts with no tests. But 100% coverage doesn't mean no bugs, because code can run without the test checking the right thing. So I focus on good tests for important logic and edge cases, not on the number. A target of about 70–80% on core modules works well."

**Q. TDD?**

> 🎤 **Interview answer:** "TDD (test-driven development) means **writing the test first**:
> 1. Write a test that fails (because the code doesn't exist yet).
> 2. Write just enough code to make it pass.
> 3. Clean up the code, with the test as a safety net.
>
> I don't use it strictly everywhere, but it's very useful for business rules like reward or price calculation, and for bugs: first write a test that shows the bug, then fix it."

---

## Git

> 🎤 **Interview answer:** "Day to day I use feature branches, small focused commits, and pull requests with code review. Useful commands beyond the basics:
> - `git stash` – put unfinished work aside for a moment.
> - `git cherry-pick` – copy one commit to another branch, like a hotfix.
> - `git revert` – safely undo a commit that's already pushed.
> - `git reset --soft` – undo my last local commit but keep the changes.
> - `git reflog` – find 'lost' commits.
>
> And `.gitignore` keeps `node_modules`, `.env` and build files out of the repo."

| Command | Use |
|---|---|
| `git clone`, `git pull`, `git push` | Basics |
| `git checkout -b feature/x` / `git switch -c` | New branch |
| `git add -p` | Stage only parts of a file |
| `git commit -m "..."` | Commit |
| `git stash` / `git stash pop` | Put work aside / bring it back |
| `git log --oneline --graph` | History |
| `git diff` | See changes |
| `git cherry-pick <hash>` | Copy one commit to current branch |
| `git reset --soft HEAD~1` | Undo last commit, keep changes |
| `git reset --hard <hash>` | Throw away changes (dangerous) |
| `git revert <hash>` | New commit that undoes a commit (safe after push) |
| `git reflog` | Find "lost" commits |
| `git tag v1.2.0` | Mark a release |

**Q. merge vs rebase?**

> 🎤 **Interview answer:**
> - "**Merge** – joins two branches with a 'merge commit'. Keeps the real history. Safe for shared branches.
> - **Rebase** – moves my commits on top of the latest `main`, so history is a clean straight line. But it **rewrites** commits, so **never rebase a branch others are using**.
>
> My usual flow: rebase my own feature branch on `main` to stay up to date, then merge it through a pull request."

**Q. reset vs revert?**

> 🎤 **Interview answer:**
> - "`git reset` **moves the branch back** to an older commit, removing the later ones from history. Only for local work that isn't pushed yet.
> - `git revert` **adds a new commit** that undoes an old one. History is not changed, so it's the safe way to undo something already pushed, like a bad change on `main`."

**Q. How do you resolve a merge conflict?**

> 🎤 **Interview answer:** "I pull the latest code, open each conflicted file, and look at both versions between the `<<<<<<<` and `>>>>>>>` markers. I don't just pick mine or theirs blindly. I understand both changes and combine them, and ask the other developer if it's unclear. Then I remove the markers, run the app and tests, `git add`, and continue. To have fewer conflicts, I keep branches short and sync with `main` often."

**Q. Branching strategy?**

> 🎤 **Interview answer:** "Each feature or bug fix gets its own branch. It goes through a pull request with code review and CI checks before merging. Releases are tagged. Urgent production fixes use a hotfix branch from `main`. I prefer short-lived branches: fewer conflicts and faster feedback."

**Q. fetch vs pull?**

> 🎤 **Interview answer:** "`git fetch` **downloads** the latest changes but doesn't change my branch, so I can look first. `git pull` = fetch **+ merge**, so it updates my branch immediately."

**Q. .gitignore?**

> 🎤 **Interview answer:** "`.gitignore` lists files Git should **not** track: `node_modules`, `.env` (secrets), `dist`, logs. Important: if a secret was already committed by mistake, adding it to `.gitignore` is not enough. It's still in the history, so the key must be changed."

---

## PM2 and Nginx

> 🎤 **Interview answer:** "**PM2** is the tool I used to run Node in production on EC2. It:
> - runs the app on **all CPU cores** (cluster mode),
> - **restarts it automatically** if it crashes,
> - manages logs,
> - can restart if memory gets too high,
> - `pm2 reload` restarts copies **one by one**, so deploys have no downtime,
> - `pm2 startup` + `pm2 save` start the apps again after a server reboot.
>
> **Nginx** sits in front of PM2. It handles HTTPS, compresses responses, serves static files, spreads requests across Node processes, limits request size, sets timeouts, does basic rate limiting, and passes the special headers Socket.io needs."

```bash
pm2 start dist/main.js -i max --name api   # run on all CPU cores
pm2 list / pm2 monit / pm2 logs api        # see status and logs
pm2 reload api                             # restart one by one – no downtime
pm2 restart api                            # hard restart
pm2 save && pm2 startup                    # start again after server reboot
```

Nginx config example in [07 → Reverse proxy](07-apis-backend.md#q-reverse-proxy--nginx).

**Q. Why not expose Node directly on port 80/443?**

> 🎤 **Interview answer:** "Nginx is better at this front-door work. It handles HTTPS and static files more efficiently, protects Node from slow clients, spreads load across many Node processes, and adds rate limits and size limits. Also, using ports 80 and 443 normally needs root access, and I don't want Node running as root. So Node listens on an internal port like 3000, and only Nginx is open to the internet."

---

## CI/CD

> 🎤 **Interview answer:**
> - "**CI (Continuous Integration)** – on every push or pull request, the pipeline automatically runs: install → lint → type-check → tests → build. Broken code is caught **before** it's merged.
> - **CD (Continuous Deployment)** – after merging, the pipeline builds the app (often a Docker image), deploys it to staging and then production, runs a health check, and rolls back if it fails. In simpler setups, it logs into the server, pulls the code, builds and runs `pm2 reload`.
>
> I've used GitHub Actions for this *[adjust to your tools]*. Secrets are kept in the CI secret store, never in the code."

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm ci
      - run: npm run lint
      - run: npm test
      - run: npm run build
```

**Environments:** dev → staging (same settings as production) → production.

---

## Linux commands

> 🎤 **Interview answer:** "On servers, I mostly use:
> - `tail -f` and `grep` to follow and search logs, `pm2 logs` for app logs,
> - `top` / `htop` and `ps aux` to check CPU and processes,
> - `df -h`, `du -sh`, `free -m` for disk and memory,
> - `lsof -i :3000` to see what's using a port,
> - `curl` to test an endpoint,
> - `systemctl` and `nginx -t` to manage and check Nginx,
> - `kill -15` to stop a process nicely, and `kill -9` only as the last option."

| Command | Use |
|---|---|
| `ls -la`, `cd`, `pwd`, `mkdir -p`, `rm -rf`, `cp`, `mv` | Files and folders |
| `cat`, `less`, `head -n 50`, `tail -f app.log` | View files / follow logs live |
| `grep -rn "ERROR" logs/` | Search text in files |
| `find . -name "*.log" -mtime +7` | Find files (here: logs older than 7 days) |
| `chmod 755 file`, `chown user:group file` | Permissions |
| `ps aux \| grep node`, `top`, `htop` | Processes and CPU |
| `kill -15 <pid>` (nice stop), `kill -9 <pid>` (force) | Stop a process |
| `df -h`, `du -sh *`, `free -m` | Disk and memory |
| `ss -tulpn`, `lsof -i :3000` | What is using a port |
| `curl -I https://api.example.com/health` | Test an endpoint |
| `ssh -i key.pem ubuntu@ip`, `scp` | Connect / copy files to a server |
| `systemctl status nginx`, `sudo nginx -t`, `sudo systemctl reload nginx` | Manage services |
| `journalctl -u nginx -f` | Service logs |
| `crontab -e` | Scheduled jobs |
| `export VAR=value`, `env` | Environment variables |

---

## Clean code and design principles

**SOLID (simple):**

> 🎤 **Interview answer:** "SOLID is five rules for clean code:
> - **S – Single responsibility:** each class has one job. That's why I separate controllers, services and repositories.
> - **O – Open/closed:** add new features without changing old code. Example: to add a new payment gateway, I add a new class, I don't edit `if/else` everywhere.
> - **L – Liskov substitution:** any child class must work wherever the parent is used. Every gateway class must follow the same contract.
> - **I – Interface segregation:** many small interfaces are better than one huge one.
> - **D – Dependency inversion:** depend on an interface, not a specific class. NestJS DI with a `PaymentGateway` interface does exactly this.
>
> Plus DRY (don't repeat yourself), KISS (keep it simple) and YAGNI (don't build what you don't need yet)."

**Design patterns you have actually used:**

> 🎤 **Interview answer:** "Patterns I've used in real projects:
> - **Singleton** – one shared DB / Redis connection.
> - **Factory** – pick the right payment gateway by country or currency.
> - **Adapter / Strategy** – one class per payment gateway, all with the same interface.
> - **Observer / Pub-Sub** – EventEmitter, Redis pub/sub, Kafka.
> - **Chain of responsibility** – Express middleware, NestJS guards and pipes.
> - **Repository** – TypeORM repositories for database access.
> - **Decorator** – everywhere in NestJS.
>
> I use a pattern only when it solves a real problem."

| Pattern | Where |
|---|---|
| Singleton | DB/Redis connection module |
| Factory | Choose payment gateway by country/currency |
| Adapter / Strategy | One class per payment gateway behind a common interface |
| Observer / Pub-Sub | EventEmitter, Redis pub/sub, Kafka |
| Chain of responsibility | Express middleware, NestJS guards/pipes |
| Repository | TypeORM repositories |
| Decorator | NestJS `@Controller`, `@Injectable` |

---

[Home](README.md) | [← 14 HR & Behavioural](14-hr-behavioural.md) | Next: [16 Revision Sheet →](16-revision-cheatsheet.md)
