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

> 🎤 **Interview answer:** "There are three main levels. Unit tests check one function or class in isolation, with its dependencies mocked. They're fast and make up most of the suite. An example is testing a discount calculation or a payment state transition. Integration tests check several parts working together, like an API endpoint with a real test database, using supertest. End-to-end tests drive the whole system like a real user, with Cypress or Playwright. They're the slowest, so there are only a few, for critical flows. That's the testing pyramid: many unit tests, fewer integration tests, very few E2E. In my projects I wrote Jest unit tests for services with mocked models and gateways, plus supertest and Postman for API flows."

| Type | Tests | Speed | Example |
|---|---|---|---|
| Unit | One function/class alone, dependencies mocked | Fast | `calculateDiscount()` |
| Integration | Several parts together (API + DB) | Medium | `POST /orders` with test DB using supertest |
| E2E | Whole system like a user | Slow | Cypress/Playwright |

**Testing pyramid:** many unit tests, fewer integration, very few E2E.

**Basic Jest test:**
```js
// discount.js
export const applyDiscount = (price, pct) => {
  if (pct < 0 || pct > 100) throw new Error("Invalid percent");
  return price - (price * pct) / 100;
};

// discount.test.js
describe("applyDiscount", () => {
  it("applies percentage", () => expect(applyDiscount(1000, 10)).toBe(900));
  it("throws for invalid percent", () => expect(() => applyDiscount(100, 120)).toThrow("Invalid percent"));
});
```

**Mocking:**

> 🎤 **Interview answer:** "Mocking replaces real dependencies, like the database, a payment gateway or an email service, with controlled fakes, so tests are fast, deterministic and focused on my logic. In Jest, `jest.fn()` creates a fake function I can assert on, like checking it was called with certain arguments. `jest.mock()` replaces a whole module, like the User model, and I set return values with `mockResolvedValue`. `jest.spyOn()` watches or overrides one method on a real object, for example making the payment client reject, to test the error path. I reset mocks in `beforeEach`, test async code with `resolves` and `rejects`, and use fake timers to test debounce or retry logic without actually waiting."

```js
jest.mock("../models/User");
User.findById.mockResolvedValue({ _id: "1", name: "Yash" });

const sendEmail = jest.fn();
expect(sendEmail).toHaveBeenCalledWith(expect.objectContaining({ to: "a@x.com" }));

jest.spyOn(paymentClient, "createOrder").mockRejectedValue(new Error("Gateway down"));
```
- `jest.fn()` – fake function. `jest.mock()` – fake whole module. `jest.spyOn()` – watch/replace a method on a real object.
- `beforeEach` / `afterEach` – setup and reset (`jest.clearAllMocks()`).
- Async: `await expect(fn()).rejects.toThrow()`; `resolves`.
- Timers: `jest.useFakeTimers()` to test debounce/retry without waiting.

**API integration test with supertest:**

> 🎤 **Interview answer:** "For integration tests I use supertest, which sends real HTTP requests to the Express app, or to `app.getHttpServer()` in NestJS, without starting a network server. So it exercises routing, middleware, validation, the controller and the service together. I run them against a separate test database, mongodb-memory-server or a Postgres in Docker, and clean the data between tests, so they're isolated and repeatable."

```js
const request = require("supertest");
const app = require("../app");

it("GET /health returns 200", async () => {
  const res = await request(app).get("/health");
  expect(res.status).toBe(200);
});
```
Use a separate test DB (e.g. `mongodb-memory-server` or a Docker Postgres) and clean data between tests.

**Q. What is code coverage?** % of lines/branches run by tests (`jest --coverage`). Useful signal, but 100% coverage doesn't mean no bugs – test important logic and edge cases.

> 🎤 **Interview answer:** "Code coverage is the percentage of lines, branches and functions executed by the tests, from `jest --coverage`. It's a useful signal for finding untested areas, but 100% coverage doesn't mean bug-free, because you can execute code without asserting the right things. So I focus on meaningful tests for important business logic and edge cases rather than chasing a number. A reasonable threshold in CI, like 70 to 80% on core modules, works well."

**Q. TDD?** Write a failing test → write code to pass → refactor.

> 🎤 **Interview answer:** "TDD, test-driven development, is a red-green-refactor cycle: first write a failing test that describes the behaviour, then write the minimum code to make it pass, then refactor with the test as a safety net. It forces clear requirements and testable design. I don't apply it strictly everywhere, but I find it very useful for pure business logic, like reward calculations or pricing rules, and for reproducing a bug with a test before fixing it."

---

## Git

> 🎤 **Interview answer:** "In daily work I use feature branches, small focused commits, and pull requests with code review. Beyond the basics, the commands I find most useful are `git stash` for parking unfinished work, `cherry-pick` for applying a hotfix commit to a release branch, `revert` for safely undoing a commit that's already pushed, `reset --soft` for redoing my last local commit, and `reflog` for recovering 'lost' commits. I tag releases, and `.gitignore` keeps node_modules, .env and build output out of the repo."

| Command | Use |
|---|---|
| `git clone`, `git pull`, `git push` | Basics |
| `git checkout -b feature/x` / `git switch -c` | New branch |
| `git add -p` | Stage parts of a file |
| `git commit -m "..."` | Commit |
| `git stash` / `git stash pop` | Save unfinished work temporarily |
| `git log --oneline --graph` | History |
| `git diff` | See changes |
| `git cherry-pick <hash>` | Copy one commit to current branch (hotfix) |
| `git reset --soft HEAD~1` | Undo last commit, keep changes staged |
| `git reset --hard <hash>` | Throw away changes (dangerous) |
| `git revert <hash>` | New commit that undoes a commit (safe for shared branches) |
| `git reflog` | Find "lost" commits |
| `git tag v1.2.0` | Mark a release |

**Q. merge vs rebase?**

> 🎤 **Interview answer:** "Both integrate changes from one branch into another. Merge creates a merge commit and keeps the true history of how branches diverged. It's non-destructive, so it's safe for shared branches. Rebase replays my commits on top of the latest main, giving a clean, linear history, but it rewrites commit hashes. So the golden rule is never rebase a branch that others are working on. My usual workflow: rebase my own feature branch on main to stay up to date and keep history clean, then merge it through a PR, often with squash merge."

- **Merge** – joins branches with a merge commit; keeps true history. Safe for shared branches.
- **Rebase** – moves your commits on top of the latest main; clean straight history. **Don't rebase branches others are using** (rewrites history).

**Q. reset vs revert?** `reset` moves the branch pointer back (rewrites history – local only). `revert` adds a new commit that undoes changes (safe for pushed code).

> 🎤 **Interview answer:** "`git reset` moves the branch pointer back to an earlier commit, effectively removing later commits from history. With `--soft` the changes stay staged, with `--hard` they're discarded. Since it rewrites history, I only use it on local, unpushed work. `git revert` creates a new commit that undoes the changes of a previous commit, without rewriting history, so it's the safe way to undo something already pushed to a shared branch, like rolling back a bad change on main."

**Q. How do you resolve a merge conflict?** Pull latest, open conflicted files, choose/combine changes between `<<<<<<<` and `>>>>>>>`, run tests, `git add`, continue merge/rebase.

> 🎤 **Interview answer:** "First I pull the latest changes. Then I open each conflicted file and look at both versions between the conflict markers. I don't just pick mine or theirs blindly. I understand both changes and combine them correctly, talking to the other developer if the intent isn't clear. Then I remove the markers, run the app and the tests to confirm nothing broke, `git add` the files and continue the merge or rebase. To reduce conflicts in the first place, I keep branches short-lived and sync with main often."

**Q. Branching strategy?** Feature branches → Pull Request → code review → merge to `develop`/`main`; release branches/tags; hotfix branches from main. (Git Flow or trunk-based with short-lived branches.)

> 🎤 **Interview answer:** "We used feature branches off develop or main. Each feature or bug fix gets its own branch and goes through a pull request with code review and CI checks before merging. Releases are tagged, and urgent production fixes go through hotfix branches from main, which are then merged back. Depending on the team, that's Git Flow, with develop and release branches, or trunk-based development with very short-lived branches and feature flags. I prefer short-lived branches, because they mean fewer conflicts and faster feedback."

**Q. fetch vs pull?** `fetch` downloads changes without merging; `pull` = fetch + merge (or rebase).

> 🎤 **Interview answer:** "`git fetch` downloads the latest commits from the remote but doesn't change my working branch, so I can inspect them first. `git pull` is fetch plus merge, or plus rebase if configured, so it updates my branch immediately. I often fetch first to see what's changed, especially before rebasing."

**Q. .gitignore?** Files Git should not track: `node_modules`, `.env`, `dist`, logs.

> 🎤 **Interview answer:** "`.gitignore` lists files and folders Git shouldn't track: `node_modules`, `.env` files with secrets, build output like `dist`, logs, coverage reports and editor files. It's critical for security, because secrets must never be committed. If a secret is committed by mistake, adding it to .gitignore isn't enough. It's still in the history, so the key has to be rotated."

---

## PM2 and Nginx

> 🎤 **Interview answer:** "PM2 is the process manager I used for Node in production on EC2. It runs the app in cluster mode across all CPU cores, restarts it automatically on crashes, manages logs, can restart on a memory limit, and `pm2 reload` restarts workers one at a time for zero-downtime deploys. `pm2 startup` and `pm2 save` bring the apps back after a server reboot, and the ecosystem file holds the config and environment.
>
> In front of PM2 sits Nginx as a reverse proxy. It terminates SSL, compresses responses with gzip, serves static files, load-balances across processes, sets body size limits and timeouts, does basic rate limiting, and passes the WebSocket upgrade headers for Socket.io."

**PM2** – process manager for Node in production.
```bash
pm2 start dist/main.js -i max --name api   # cluster mode on all cores
pm2 list / pm2 monit / pm2 logs api
pm2 reload api        # zero-downtime restart (one worker at a time)
pm2 restart api       # hard restart
pm2 save && pm2 startup  # start apps automatically after server reboot
```
Features: auto-restart on crash, cluster mode, log management, `max_memory_restart`, ecosystem file (`ecosystem.config.js`) for env and settings.

**Nginx** – reverse proxy in front of Node: SSL, gzip, static files, load balancing, websocket upgrade headers, rate limiting (`limit_req`), max body size (`client_max_body_size`), timeouts. Config example in [07 → Reverse proxy](07-apis-backend.md#q-reverse-proxy--nginx).

**Q. Why not expose Node directly on port 80/443?** Nginx handles SSL and static files more efficiently, protects from slow clients, can balance across processes, and Node doesn't need root permissions.

> 🎤 **Interview answer:** "Nginx is better at the edge work. It handles SSL termination and static files more efficiently, buffers slow clients so they don't tie up Node, load-balances across multiple Node processes, and adds rate limits and body size limits. Also, binding to ports 80 and 443 normally needs root, and I don't want the Node app running as root. So Node listens on an internal port like 3000, and only Nginx is exposed."

---

## CI/CD

> 🎤 **Interview answer:** "CI, continuous integration, means every push or pull request automatically runs `npm ci`, lint, type-check, tests and the build, so broken code is caught before merging. CD, continuous delivery or deployment, means after merging, the pipeline builds the artifact, typically a Docker image pushed to ECR, deploys it to staging and then production, runs health checks, and rolls back if they fail. In simpler setups it SSHes to the server, pulls, builds and runs `pm2 reload`. I've used GitHub Actions for this kind of pipeline *[adjust to your tools]*. Secrets live in the CI secret store, and staging mirrors production config so surprises are caught early."

- **CI (Continuous Integration):** on every push/PR → install (`npm ci`) → lint → type-check → test → build.
- **CD (Continuous Delivery/Deployment):** after merge → build Docker image → push to registry (ECR) → deploy to server/ECS (or SSH + `git pull` + `pm2 reload`) → health check → rollback if failing.
- Tools: GitHub Actions, GitLab CI, Jenkins, AWS CodePipeline.

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

**Environments:** dev → staging (same as prod config) → production. Secrets stored in the CI secret store, not in the repo.

---

## Linux commands

> 🎤 **Interview answer:** "On servers, the commands I use most are: `tail -f` and `grep` to follow and search logs, `pm2 logs` for app logs, `top` or `htop` and `ps aux` to check CPU and processes, `df -h`, `du -sh` and `free -m` for disk and memory, `ss -tulpn` or `lsof -i :3000` to see what's using a port, `curl` to test endpoints and health checks, `systemctl` and `nginx -t` to manage and validate Nginx, `journalctl` for service logs, and `kill -15` for a graceful stop, using `kill -9` only as a last resort."

| Command | Use |
|---|---|
| `ls -la`, `cd`, `pwd`, `mkdir -p`, `rm -rf`, `cp`, `mv` | Files |
| `cat`, `less`, `head -n 50`, `tail -f app.log` | View files / follow logs |
| `grep -rn "ERROR" logs/` | Search text |
| `find . -name "*.log" -mtime +7` | Find files |
| `chmod 755 file`, `chown user:group file` | Permissions |
| `ps aux \| grep node`, `top`, `htop` | Processes and CPU |
| `kill -15 <pid>` (graceful), `kill -9 <pid>` (force) | Stop process |
| `df -h`, `du -sh *`, `free -m` | Disk and memory |
| `netstat -tulpn` / `ss -tulpn`, `lsof -i :3000` | What is using a port |
| `curl -I https://api.example.com/health` | Test an endpoint |
| `ssh -i key.pem ubuntu@ip`, `scp` | Connect / copy to server |
| `systemctl status nginx`, `sudo nginx -t`, `sudo systemctl reload nginx` | Services |
| `journalctl -u nginx -f` | Service logs |
| `crontab -e` | Scheduled jobs |
| `export VAR=value`, `env` | Environment variables |

---

## Clean code and design principles

**SOLID (simple):**

> 🎤 **Interview answer:** "SOLID is five design principles. Single responsibility: each module has one job, which is why I separate controllers, services and repositories. Open/closed: add behaviour without modifying existing code. For example, adding a new payment gateway means adding a new adapter class, not editing if-else chains everywhere. Liskov substitution: any implementation should work wherever its parent or interface is expected, so every gateway adapter must honour the same contract. Interface segregation: small, focused interfaces instead of one huge one. Dependency inversion: depend on abstractions, not concrete classes, which is exactly what NestJS DI with a PaymentGateway interface gives us. Together with DRY, KISS and YAGNI, these keep code testable and easy to change."

- **S**ingle responsibility – one class/module does one job (controller vs service vs repository).
- **O**pen/closed – add new behaviour without changing old code (add a new payment gateway class, don't edit `if/else` everywhere).
- **L**iskov substitution – a subclass should work wherever the parent works.
- **I**nterface segregation – small focused interfaces instead of one huge one.
- **D**ependency inversion – depend on interfaces, not concrete classes (NestJS DI, `PaymentGateway` interface).

**Other principles:** DRY (don't repeat yourself), KISS (keep it simple), YAGNI (don't build what you don't need yet).

**Design patterns you have actually used:**

> 🎤 **Interview answer:** "The patterns I've actually used in production: Singleton, for the DB and Redis connection modules, thanks to Node's require cache. Factory, to choose the payment gateway by country or currency. Adapter or Strategy, with one class per payment gateway behind a common interface. Observer or pub-sub, with EventEmitter, Redis pub/sub and Kafka. Chain of responsibility, which is basically Express middleware and NestJS guards and pipes. Repository, with TypeORM repositories as the data-access layer. And Decorator, everywhere in NestJS. I use patterns when they solve a real problem, not for their own sake."

| Pattern | Where |
|---|---|
| Singleton | DB/Redis connection module (Node `require` cache) |
| Factory | Choose payment gateway by country/currency |
| Adapter / Strategy | One class per payment gateway behind a common interface |
| Observer / Pub-Sub | EventEmitter, Redis pub/sub, Kafka |
| Middleware / Chain of responsibility | Express middleware, NestJS guards/pipes |
| Repository | TypeORM repositories, data-access layer |
| Decorator | NestJS `@Controller`, `@Injectable` |

---

[Home](README.md) | [← 14 HR & Behavioural](14-hr-behavioural.md) | Next: [16 Revision Sheet →](16-revision-cheatsheet.md)
