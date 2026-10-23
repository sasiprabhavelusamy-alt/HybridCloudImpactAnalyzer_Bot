# Viva Script — Student Side (Blast-Radius Guard)

**Project:** CloudGuard — Hybrid Cloud Configuration Change Impact Analyzer
**This part:** the student side. It has a Chrome/Edge extension for the AWS Console, a guard for the AWS CLI and boto3, and the CloudGuard website.
**Repo:** github.com/Shobaa-cloud/HybridStudents

> How to use this script
> - Lines in quote blocks marked **Say:** are meant to be spoken out loud, close to word for word.
> - Everything else is background you should understand so you can answer follow-up questions in your own words.
> - Section 14 is the question bank. Practise answering those out loud without reading.
> - Section 16 maps each file to what it does. Examiners often say "show me where that happens in the code". Know those files.

---

## 1. Opening (about 45 seconds)

> **Say:** "Our project is CloudGuard, a hybrid cloud configuration change impact analyzer. Its job is to tell you what a change will break **before** you make it, not after.
>
> We built it for two audiences, because they work in completely different ways. Professionals change infrastructure through Terraform code and pull requests, so for them we built a PR bot. Students and freshers don't write Terraform. They click buttons in the AWS Console or run a few CLI commands, so for them we built the part I'm presenting now: the **Blast-Radius Guard**.
>
> It's a browser extension, plus a small Python hook for the CLI and SDK. It pauses a risky action, like terminating an EC2 instance or opening SSH to the whole internet, and explains in plain English what will happen and what else it affects. Then the student decides: cancel, or proceed. It needs no AWS keys, no server and no setup beyond loading the extension."

---

## 2. How we got the idea (our story)

> **Say:** "When we were freshers, we started building things in AWS without full technical knowledge. We were honestly scared to touch or change any configuration out of curiosity, because we thought we'd break something or get billed. A friend of ours actually got an AWS bill after she thought she had deleted everything, because something was still running that she hadn't noticed.
>
> We also didn't want to ask an AI every time before clicking something. That fear stopped us from experimenting, and experimenting is how you actually learn the cloud.
>
> Tools like AWS Config and CloudTrail do exist, but they tell you what changed **after** it has happened. Terraform plan does show a preview before applying, but it's a raw technical diff that assumes you already understand what it means, and beginners don't use Terraform anyway. So we wanted something that says **what will happen, in plain English, before the mistake happens**, right where the student is clicking. And we wanted the same idea to be useful in industry too, which is why there's a professional side."

**Why this story is strong:** it's a real problem (fear of breaking things, fear of billing), and it leads straight to the design decisions: warn before the action, plain English, zero cost, zero credentials.

**Be careful here:** don't say "terraform plan only shows things after". It shows a preview before apply. The correct point is that it's **technical, not plain English, and students don't use Terraform**.

---

## 3. Problem statement (one paragraph)

Beginners learning AWS through the Console make a small set of very common, very costly mistakes: terminating an instance and losing its disk, leaving an Elastic IP that keeps billing, and opening SSH, RDP or database ports to `0.0.0.0/0`, the entire internet. AWS's own protections are either generic ("Are you sure?"), or small warning text that doesn't stop the save, or services like AWS Config that detect a problem after it exists and cost money to run. There is no free, zero-setup tool that **intercepts the action before it runs** and explains its **blast radius** (what else it affects) in language a beginner understands.

---

## 4. What we built (three pieces)

| Piece | Who uses it | What it does |
|---|---|---|
| **CloudGuard website** (`website/`) | Everyone | Landing page that asks "Who's here?". **I'm a student** leads to the extension download and a 4-step install guide. **I'm an intern or pro** leads to the PR bot. This is the "one application, two roles" idea. |
| **Browser extension** (`aws-blast-radius-extension/`) | Students using the AWS Console | Intercepts two risky clicks, shows a warning with the blast radius, and lets the student cancel or proceed |
| **CLI/SDK guard** (`aws-blast-radius-cli-guard/`) | Students using `aws` CLI v1 or `boto3` Python scripts | Same idea for the terminal: pauses the API call and asks `Proceed anyway? [y/N]` |

**The two mistakes we catch** (deliberately small MVP scope):
1. **Terminating EC2 instances.** Termination is permanent, the root EBS volume is deleted by default, instance-store data is lost, and an Elastic IP keeps billing.
2. **Saving inbound security-group rules open to `0.0.0.0/0` or `::/0`.** SSH, RDP, databases and "All traffic" are **High risk**. Other ports get a **Heads up**. Web ports 80 and 443 pass silently, because websites are meant to be public.

---

## 5. The mechanism, named

We gave both sides a formal name so they're parallel:

| Side | Mechanism |
|---|---|
| Professional | **Pre-deployment static blast-radius analysis of Infrastructure-as-Code** |
| Student | **Pre-execution dynamic blast-radius analysis of live API calls** |

> **Say:** "The professional side is **static**: it reads a Terraform plan file before deployment, without running anything. The student side is **dynamic**: there's no file, so the 'change' is the actual action the student is performing, captured a split second before it runs. Same concept, blast radius before the change, but a different trigger."

**Why it's called "blast radius":** like an explosion, a change affects more than the thing you touched. Opening a security group affects every instance that uses that group.

---

## 6. How the browser extension works, step by step

1. **Injection.** `manifest.json` (Manifest V3) tells Chrome to inject `inventory.js`, `content.js` and `modal.css` into every page matching `https://*.console.aws.amazon.com/*`, including frames (`all_frames: true`).
2. **Listening.** `content.js` adds a click listener on the document in the **capture phase** (`addEventListener("click", onClick, true)`). Browser events travel down from the page to the button (capture) before bubbling back up. AWS's React code listens in the bubble phase, so **our listener runs first**.
3. **Identifying the button.** It uses `e.composedPath()` to find the real button, even inside Shadow DOM, and reads its visible **text** ("Terminate", "Save rules").
4. **Checking the risk.**
   - `checkTerminate`: only if the button is inside AWS's confirmation dialog. It reads instance IDs (`i-xxxxxxxx`) **only from that dialog**, so the warning always names the instance actually being terminated.
   - `checkInboundRules`: only on the **Edit inbound rules** page. It finds every rule row whose source is `0.0.0.0/0` or `::/0`, reads the port, and grades it with `assessRule`.
5. **Blocking.** If something is risky, it calls `e.preventDefault()` and `e.stopImmediatePropagation()`, so **AWS never receives the click**.
6. **Explaining.** It shows our modal: risk badge, resource IDs, reasons, the blast-radius box, a tip (for example "Use Stop instead" or "Change Source to My IP"), and two buttons. **"Cancel, keep me safe" is focused by default**, so Enter or Escape is the safe choice.
7. **Deciding.**
   - **Cancel:** nothing happens, and the "mistakes avoided" counter goes up.
   - **Proceed anyway:** we set `bypassNextClick = true` and call `button.click()` again. That re-click passes straight through to AWS, so **AWS does the action itself, under the student's own login session**. The extension never makes an AWS call.
8. **Stats.** `chrome.storage.local` keeps "warnings shown" and "mistakes avoided", which the toolbar popup displays.

### How the blast radius is found without AWS keys (`inventory.js`)

This was the answer to "AWS already shows a warning, so what do you add?". We wanted to show **what else is affected**, not just "this is risky".

- While the student browses normally, `inventory.js` reads the tables **the console is already showing**:
  - **EC2 → Instances:** each instance's ID, name, state, public IP and security groups
  - **Security Groups list:** which group ID has which name
- It stores this in `chrome.storage.local`, in the student's own browser only. It sends nothing anywhere and makes no API calls.
- When the student opens SSH on `sg-abc`, the warning says, for example, **"Blast radius: 2 instances use this security group"**, lists them, and highlights which ones have a public IP and **can be reached from the internet as soon as you save**.
- It's honest about freshness: "Based on the Instances page as you last saw it (5 min ago)." If it hasn't learned anything yet, it says "Open EC2 → Instances once, and next time the guard will list exactly which instances this change reaches."
- The popup has a **Forget** button that wipes the stored inventory (privacy control).

### Tricky engineering problems we solved (good viva material)

| Problem | What we did |
|---|---|
| The Security Groups page lives inside a same-origin **iframe** (`security-groups-react-frame`) that Chrome didn't inject into, so no warning appeared | `forEachDoc` walks into same-origin iframes from the top page and attaches the guard there, re-checking every second because frames get rebuilt as you navigate |
| Risk of **two modals** when a frame is guarded twice | A `data-brg-guarded` marker on the frame's DOM so it's only guarded once |
| AWS changes CSS class names often | We detect by **visible button text** and **resource-ID patterns** (`i-…`, `sg-…`), which survive redesigns much better |
| The source CIDR sometimes appears as an input value, sometimes as a "chip" label | We check both inputs and text nodes, then climb up to the rule row (`findRuleRow`) |
| Page text is untrusted (XSS risk) | The modal uses `textContent`, never `innerHTML` |
| Our CSS clashing with AWS's | Every class is prefixed `brg-` |

---

## 7. How the CLI/SDK guard works, step by step

> **Say:** "At first we planned only a CLI wrapper. Then we asked: why only the CLI, and not the SDK? AWS CLI v1 and boto3 scripts both run on the same library, **botocore**. So instead of two tools, we hook botocore once and catch both."

1. **Auto-loading.** `sitecustomize.py` is a special file Python runs automatically at startup if it's on `PYTHONPATH`. It calls `enable()`. The student changes no code.
2. **Patching.** `hook.py` replaces `botocore.session.Session.create_client` with our version (monkey-patching). Every EC2 client created by the CLI or boto3 goes through it.
3. **Registering.** For watched operations (`TerminateInstances`, `AuthorizeSecurityGroupIngress`), we register a handler on botocore's own event system: `before-parameter-build.ec2.TerminateInstances`. botocore fires this event **before the request is built and sent**.
4. **Checking.** `rules.py` runs the check:
   - Terminate: calls `describe_instances` and `describe_addresses` (read-only) to list the **exact EBS volumes with DeleteOnTermination=true** and any Elastic IP.
   - Ingress: flags `0.0.0.0/0` and `::/0` on sensitive ports.
5. **Confirming.** It prints a warning banner and asks `Proceed anyway? [y/N]`. **No is the default.**
6. **Aborting.** If the answer isn't `y`, it raises `BlastRadiusAborted`. An exception inside that botocore event **cancels the call**, so AWS never receives it.

**Why the CLI guard can be more precise than the extension:** it runs with the student's own CLI credentials (which they already have configured), so it can make read-only describe calls and name the exact volume IDs. The extension deliberately has no credentials, so for termination it describes AWS's default behaviour.

---

## 8. Tools and techniques, with why and why not

| Tool / technique | Why we used it | Alternative we rejected, and why |
|---|---|---|
| **Chrome extension, Manifest V3** | Runs right where students work (the Console). MV3 is the current standard, and MV2 is being phased out. Works in Chrome, Edge and Brave. | **Desktop app:** can't see what's happening inside the browser. **Separate website:** students would have to copy things over manually, too much friction. |
| **Content script** (DOM-level interception) | Can see the button, the dialog and the form, and can stop the click before AWS's code runs | **Blocking network requests (`webRequest`):** MV3 removed blocking `webRequest` for normal extensions, and `declarativeNetRequest` can't read request bodies, so we couldn't tell what's in the request. Also, by the time a request is sent it's too late to explain nicely. |
| **Capture-phase event listener** | Runs **before** AWS's own React handlers, so the action is truly stopped **before** it happens | Bubble phase: AWS would get the click first |
| **`preventDefault` + `stopImmediatePropagation`, then re-click on Proceed** | We never perform the action ourselves. AWS does it under the student's session. | Calling AWS APIs from the extension would need credentials |
| **`chrome.storage.local`** | Local only, survives restarts, needs only the `storage` permission | A backend database would mean sending data off the student's machine |
| **Text and ID-pattern detection (regex)** | Survives AWS UI redesigns better than CSS classes | CSS selectors break whenever AWS renames a class |
| **Plain JavaScript, no framework or build step** | Tiny (about 24 KB), easy to audit, zero setup | React or TypeScript would add a build step for no user benefit |
| **Python + botocore event hooks** | One hook covers **CLI v1 and boto3**, and uses botocore's official event system | A **CLI wrapper** only catches CLI commands, not SDK scripts |
| **`sitecustomize.py` + `PYTHONPATH`** | Auto-loads the guard with **zero code changes** in the student's scripts | Asking students to import our module in every script |
| **Rule-based checks** | Every warning can be explained by a specific rule, and it's instant, free and offline | **Machine learning:** there's no labelled dataset of "this click caused this incident", and a black box isn't trustworthy for a safety tool |
| **Scoped test IAM user** (for testing the CLI guard) | Least privilege: only the describe, terminate and authorize actions needed for testing | Using root or admin keys |
| **Static website (HTML/CSS/JS)** | Role picker plus download and install guide, can be hosted anywhere for free | A heavy app framework wasn't needed |
| **`package_extension.py`** | One command rebuilds the downloadable zip so the website always serves the latest version | Zipping by hand each time (error-prone) |

---

## 9. Risk logic (know these numbers)

**Sensitive ports** (open to the internet means **High risk**):
22 SSH · 3389 RDP · 3306 MySQL · 5432 PostgreSQL · 1433 SQL Server · 27017 MongoDB · 6379 Redis · 9200 Elasticsearch
(The CLI guard's rule list is a slightly shorter subset.)

| Rule | Result |
|---|---|
| "All traffic", "All TCP", "All UDP", or ports 0–65535 open to the world | **High:** every service is exposed |
| A sensitive port open to the world | **High:** "bots scan the whole internet for this port within minutes" |
| Port 80 or 443 open to the world | **No warning**, because websites are meant to be public |
| Any other port open to the world | **Heads up** (medium) |
| Outbound rules | Not checked. Outbound to `0.0.0.0/0` is AWS's normal default. |

---

## 10. Live demo plan (what to click)

**Before the viva:** extension loaded in Chrome, logged in to your free AWS account, one `t2.micro`/`t3.micro` instance running with its own security group. Open **EC2 → Instances** once and **Security Groups** once so the inventory is learned.

**Demo 1: open SSH to the world**
1. EC2 → Security Groups → select your group → **Edit inbound rules**
2. **Add rule** → Type **SSH** → Source **Anywhere-IPv4**
3. Click **Save rules**, and the **High risk** modal appears
4. Point out: the port and reason, **"Blast radius: 1 instance uses this security group"**, which instances are internet-reachable, and the "My IP" tip
5. Click **Cancel, keep me safe**, then refresh the page and show the **rule was not saved**. This proves it acts **before** the action.

**Demo 2: terminate**
1. Instances → select instance → Instance state → **Terminate** → click **Terminate** in AWS's dialog
2. Our modal appears: permanent, root disk deleted, Elastic IP keeps billing, tip "use Stop instead"
3. Cancel. Show the instance is still running.

**Demo 3: popup.** Click the toolbar icon to show "warnings shown / mistakes avoided" and the learned inventory.

**Optional Demo 4: CLI** (only show this after you've tested it yourself). `aws ec2 authorize-security-group-ingress ... --port 3306 --cidr 0.0.0.0/0` shows the banner, answer `n`, and you get `BlastRadiusAborted`.

After the demo, **delete the test resources** (zero billing).

---

## 11. Advantages (our points, polished)

1. **Warns before, not after.** The click is stopped in the capture phase, before AWS receives it. AWS Config and CloudTrail only report after the change exists.
2. **Explains the blast radius, not just "risky".** It names the instances that use the group and which ones become internet-reachable. AWS's own inbound-rule note is generic text that doesn't stop the save and doesn't say what's affected.
3. **Zero credentials.** It never asks for, stores or sees AWS keys. Its only permission is `storage`. So there's **no central server holding students' access that could get hacked**, unlike SaaS tools that need a role in your account.
4. **Zero cost, zero billing risk.** No AWS Config (which charges per configuration item), no Lambda, no servers. This mattered to us because of the billing story.
5. **Zero setup.** No AWS Config, no Terraform, no terminal, no IaC knowledge. A fresher doesn't need to learn those tools first. About a minute to install.
6. **Protects curiosity instead of blocking it.** It never forbids anything. "Proceed anyway" always works, so students can still experiment, just informed. It teaches why ("bots scan port 22 within minutes") and the safer alternative ("Stop instead of Terminate", "My IP").
7. **Privacy.** The inventory stays in the student's browser, and there's a Forget button.
8. **Covers Console, CLI and SDK.** One botocore hook covers both CLI v1 and boto3 scripts.
9. **Safe by default.** Cancel is focused, Escape cancels, clicking outside cancels. In the terminal, `N` is the default.
10. **Resilient.** Detection uses visible text and ID patterns, not fragile CSS classes.
11. **Measurable.** The popup counts warnings and mistakes avoided.

---

## 12. Limitations (say these confidently; honesty scores marks)

1. **AWS only on the student side.** Azure (`portal.azure.com`) and GCP (`console.cloud.google.com`) have completely different page layouts, so each needs its own checks. The core (catch click → check → warn → count) is reusable. The **multi-cloud part of the project lives on the professional side**, which handles AWS, Azure and GCP in one graph.
2. **Only two scenarios.** Terminate and inbound rules. This was a deliberate MVP focusing on the most costly fresher mistakes. More (public S3 buckets, deleting security groups, IAM changes) would follow the same pattern.
3. **Depends on the console's page layout.** If AWS redesigns the page, detection may need adjusting. We reduced this by matching text and IDs instead of CSS classes.
4. **The inventory is only as fresh as what the student last viewed.** The warning shows how old the data is. Matching groups by name is a fallback, and two VPCs can both have a group called "default".
5. **The terminate warning in the extension describes AWS default behaviour**, not this instance's exact volumes, because the extension has no credentials. (The CLI guard does name exact volumes.) A future "deep scan" could use short-lived STS credentials from a backend, never long-lived keys in the browser.
6. **AWS CLI v2 is not covered.** It's a compiled binary with its own bundled Python, so our botocore hook in the student's environment doesn't reach it. CLI v1 and boto3 are covered.
7. **The CLI prompt is interactive** (`input()`), so it's not suited to unattended scripts or CI without an allowlist or `--yes` option.
8. **Not on the Chrome Web Store yet**, so install is 4 steps (Developer mode → Load unpacked). Publishing costs a one-time $5 fee plus review, and would turn install into one "Add to Chrome" click.
9. **It can be bypassed.** A student can disable it or click Proceed. It's a **learning seatbelt, not a security control**, by design.

---

## 13. Future scope

- Azure and GCP console support (the same engine with new URL patterns and checks)
- More scenarios: public S3 bucket, deleting a security group still in use, `0.0.0.0/0` on IPv6, IAM policy with `*`
- Chrome Web Store publishing for one-click install
- Optional "deep scan" using short-lived STS credentials for exact volume and dependency data
- CLI v2 support through a wrapper around the v2 binary

---

## 14. Question bank (practise these out loud)

**Q1. What is your project in one line?**
A tool that tells you what a cloud change will break before you make it, in plain English: a PR bot for professionals and a browser guard for students.

**Q2. Why is it called "hybrid"?**
Two meanings. It's **multi-cloud**: the professional side analyses AWS, Azure and GCP resources in one dependency graph, including changes that cross from one cloud to another. And it's **hybrid in audience**: one project with two front doors, students and professionals, chosen on the website's role picker.

**Q3. AWS already shows a warning on 0.0.0.0/0. Why do we need yours?**
AWS's note is small generic text. It doesn't stop the save, says the same thing for every port, doesn't give a fix, and doesn't tell you what's affected. Ours pauses the save, names the port and why it's dangerous, suggests "My IP", and shows the **blast radius**: which instances use that group and which become reachable from the internet.

**Q4. Does it really stop the action before it happens, or after?**
Before. The listener runs in the capture phase, before AWS's React handlers, and calls `preventDefault` and `stopImmediatePropagation`. We proved it: click Cancel, refresh, and the rule is not there. (Early in testing it looked like it warned after, but that rule had already been saved before the extension was installed. It even had a rule ID `sgr-…`, which only saved rules get.)

**Q5. Does the extension need AWS credentials?**
No. It never asks for, stores or sees keys, and makes no AWS API calls. On Proceed, it re-clicks AWS's own button, so AWS acts under the student's existing login.

**Q6. Then how does it know which instances use a security group?**
`inventory.js` reads the Instances and Security Groups tables the console is already showing while the student browses, and stores them locally. That's why it shows "based on the Instances page as you last saw it (x min ago)".

**Q7. Why is that more secure than tools like Overmind?**
Hosted tools need standing access to your cloud account, usually a role you give their servers. If that vendor were breached, attackers could reach every customer's account: that's third-party or supply-chain risk. We hold nobody's access, so there's nothing central to steal. (Correct wording: the danger is **access roles or keys**, not account IDs. AWS doesn't treat account IDs as secret.)

**Q8. Is it continuously monitoring the account 24/7?**
No, and that's deliberate. It's **event-driven**: it reacts at the moment of action (a click, or an API call). Nothing polls the account, so there's no cost and no credentials.

**Q9. Why a browser extension and not a website or app?**
The risky action happens inside the AWS Console tab. Only an extension can see and pause that click. A separate website would need the student to copy things over, which nobody does mid-task.

**Q10. Why Manifest V3?**
It's Chrome's current extension platform, and MV2 is being phased out. It also forced a good design: MV3 doesn't allow blocking network requests, so we intercept at the UI level, which is also the point where we can explain things in plain English.

**Q11. Why intercept clicks and not network requests?**
MV3's `declarativeNetRequest` can block by URL but can't read the request body, so it can't tell what the request does. And by the time a request is sent, the user is gone from the decision. At the click we have context: the dialog, the form, and the instance IDs.

**Q12. What is the capture phase?**
DOM events travel from the document down to the target (capture), then back up (bubble). Registering with `true` as the third argument means our listener runs on the way down, before AWS's handlers.

**Q13. What happens when the student clicks Proceed?**
We set `bypassNextClick = true` and call `button.click()`. Our listener sees the flag, lets that one click through, and AWS's own handler runs normally.

**Q14. Why only two scenarios?**
They're the two most common, most expensive fresher mistakes: permanent data loss plus leftover billing, and exposing SSH or databases to internet scanners. MVP first, done properly. The design makes adding more scenarios straightforward.

**Q15. Why do ports 80 and 443 pass silently?**
Web servers are supposed to be public. Warning on them would create **alert fatigue**, and students would learn to click Proceed without reading.

**Q16. Why does the CLI guard hook botocore instead of wrapping the CLI?**
CLI v1 and boto3 both run on botocore. One hook on `create_client` plus botocore's `before-parameter-build` event covers both, with no code changes, through `sitecustomize.py`.

**Q17. What's monkey-patching, and is it safe?**
Replacing a function at runtime with our own version that calls the original. We keep the original `create_client`, call it, and just register extra handlers on the client. We mark the patch so it's never applied twice. `sitecustomize` wraps everything in try/except so a failure can never break an unrelated Python program.

**Q18. Why doesn't it cover AWS CLI v2?**
v2 is a compiled standalone binary with its own bundled Python, not a botocore process in the student's environment, so our hook doesn't load into it. That's a documented scoping decision. A wrapper around the v2 binary is future work.

**Q19. Why rule-based and not AI or ML?**
There's no labelled dataset of "this click caused this incident". Rules are explainable (every warning traces to one statable rule), instant, free, offline and deterministic. For a safety tool, trust matters more than sounding clever.

**Q20. Why not just use AWS Config or Trusted Advisor?**
AWS Config reports after a resource is non-compliant, and it's billed per configuration item recorded and per rule evaluation, which doesn't suit a zero-billing student account. Neither stops the click, and neither explains in beginner language. Also, a fresher would have to learn and set up those services first.

**Q21. Why not just teach students Terraform so they can use `terraform plan`?**
Freshers start in the Console. Asking them to learn IaC first is the barrier we're trying to remove. And `terraform plan` output is a technical diff, not "this will delete your disk". Students grow into Terraform later, and that's what the professional side is for.

**Q22. What about Azure and GCP students?**
Honest answer: the student guard is AWS-only today, because each portal has a different page structure. The engine is reusable: only the URL match and the detection functions change. Multi-cloud analysis is done on the professional side, from Terraform plans.

**Q23. Can the extension damage my AWS account?**
No. It can't create, change or delete anything. It only pauses a click and shows a message. AWS acts only if you choose Proceed.

**Q24. What data does it store, and where?**
Counters, plus a small inventory (instance ID, name, state, public IP, group names), all in `chrome.storage.local` in the student's own browser. Nothing is sent anywhere. The Forget button clears it.

**Q25. What if AWS changes its console design?**
We match visible text ("Terminate", "Save rules") and resource ID patterns, not CSS classes, which survives most redesigns. If something still breaks, `TERMINATE_LABEL` and `findRuleRow` are the two places to adjust.

**Q26. What was the hardest bug?**
The warning didn't appear on the Security Groups page. That page is inside a same-origin iframe (`security-groups-react-frame`) that Chrome wasn't injecting our script into. We fixed it by walking into same-origin frames from the top page and attaching the guard there, with a marker so no frame gets two modals.

**Q27. How did you prevent XSS in your modal?**
All text is set with `textContent`, never `innerHTML`, because page text such as instance names is untrusted.

**Q28. How did you test it?**
In my free-tier AWS account: a `t2.micro` instance, a test security group, then both scenarios. Cancel means the rule and the instance are unchanged after refresh. Proceed means AWS completes it normally. Afterwards we deleted everything, so there was no billing. The CLI/SDK guard is a working prototype, but it hasn't been tested live against AWS yet; that's our next testing step.

**Q29. What accuracy does it have?**
It's not a statistical model, so "accuracy percentage" isn't the right measure. It's deterministic: for the two supported scenarios on the current console layout, it triggered correctly in every test we ran. The honest limits are scope (two scenarios, AWS only) and inventory freshness. Those are coverage limits, not random errors.

**Q30. Who benefits?**
Students, freshers, bootcamp and college lab users, anyone learning AWS on a personal or free-tier account. Also educators running labs, who want fewer "I got billed" or "my instance got hacked" stories.

**Q31. How do students install it?**
Website → **I'm a student** → download the zip → unzip → `chrome://extensions` → Developer mode on → **Load unpacked**. On the Chrome Web Store it would be one click.

**Q32. How does the student side relate to the professional side?**
Same idea, blast radius before the change, but a different input. Professionals give a Terraform plan in a pull request (static). Students give nothing: the live action itself is the input (dynamic). The website routes each person by role.

**Q33. Did you use AI tools while building it?**
Answer honestly: yes, as a coding assistant, like most developers do today. But the problem, the decisions and the testing were ours, and I can explain every part of the code. Then name decisions that came from you (see the note at the end of this file).

**Q34. What would you do with more time?**
Azure and GCP console support, more scenarios (public S3, IAM wildcards), Chrome Web Store publishing, and optional deep scan through short-lived credentials.

**Q35. Isn't this just a pop-up?**
The pop-up is the output. The work is in deciding **when** to show it (only real risks, not 80/443, not outbound), **what to say** (the specific port, the reason, the fix), and **what else it affects** (the learned inventory linking groups to instances and public IPs), while stopping the action before AWS gets it, with no credentials.

---

## 15. Phrases to use, and traps to avoid

| Don't say | Say instead |
|---|---|
| "Terraform plan only shows after" | "Terraform plan is a technical diff, and students don't use Terraform" |
| "It's live testing" | "It intercepts the live action before it runs" |
| "It's 100% accurate" | "It's deterministic, and within its two scenarios it triggered correctly in all our tests" |
| "It's AI" | "It's rule-based, which is deliberate for explainability" |
| "It works on all clouds" (student side) | "The student guard is AWS-only; multi-cloud is on the professional side" |
| "Hackers can steal account IDs from other tools" | "Hosted tools hold access roles to customers' accounts; we hold none" |
| "It blocks students" | "It pauses and explains; the student always decides" |

---

## 16. Code map (know where everything is)

| File | What it does | Key names to remember |
|---|---|---|
| `aws-blast-radius-extension/manifest.json` | MV3 config: matches `*.console.aws.amazon.com`, `all_frames`, only `storage` permission | `content_scripts`, `permissions` |
| `src/content.js` | Click interception, risk checks, modal | `onClick`, `checkTerminate`, `checkInboundRules`, `assessRule`, `showWarningModal`, `bypassNextClick`, `guard`, `forEachDoc`, `SENSITIVE_PORTS` |
| `src/inventory.js` | Learns instances and group names from console tables, with no API calls | `learnFrom`, `readTables`, `instancesUsingGroup`, `knownInstance` |
| `src/modal.css` | Modal styles, all `brg-` prefixed | |
| `popup.html/js/css` | Toolbar popup: counters, learned inventory, Forget button | `warned`, `avoided` |
| `aws-blast-radius-cli-guard/sitecustomize.py` | Auto-loads the guard at Python startup | `enable()` |
| `blast_radius_guard/hook.py` | Patches `create_client`, registers botocore events, prompt | `WATCHED`, `before-parameter-build`, `BlastRadiusAborted` |
| `blast_radius_guard/rules.py` | Terminate and ingress checks using read-only describe calls | `check_terminate_instances`, `check_authorize_ingress` |
| `website/index.html, app.js, styles.css` | Role picker, install guide, FAQ, PR bot section | |
| `package_extension.py` | Rebuilds `website/downloads/blast-radius-guard.zip` | |

---

### Note for you (not for the examiner)

Many of the strongest design decisions in this project came from **your own questions**. Use them as proof you own it:
- "Why only CLI, why not SDK?" led to the botocore hook covering both.
- "AWS already warns, so why us?" led to the real blast-radius inventory.
- "Is it really before, not after?" led to the Cancel-then-refresh test that proves it.
- "What about Azure and GCP?" led to naming the scope honestly.
- "No one's credentials, so no vendor breach risk" led to the zero-credential design as a headline advantage.
- The billing-fear story led to zero cost and zero AWS Config.
