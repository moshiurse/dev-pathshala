# 🌳 Git Advanced — সিনিয়র ইঞ্জিনিয়ারদের জন্য গভীর গাইড

> **Git শুধু `add/commit/push` নয়। এর ভিতরে আছে একটি content-addressable filesystem, একটি immutable DAG এবং কিছু চমৎকার recovery tool।**
> এই গাইডে আমরা Git-এর internals থেকে শুরু করে monorepo strategy, conflict resolution, recovery scenario এবং বাংলাদেশী টেক কোম্পানির real-world workflow পর্যন্ত cover করব।

---

## 📖 সূচিপত্র

- [Git Internals — মূলকাঠামো](#-git-internals--মূলকাঠামো)
- [Object Model: Blob, Tree, Commit, Tag](#-object-model)
- [.git/ ডিরেক্টরি বিশ্লেষণ](#-git-ডিরেক্টরি-বিশ্লেষণ)
- [Daily Commands Beyond Basics](#-daily-commands-beyond-basics)
- [Rebase Mastery](#-rebase-mastery)
- [Cherry-pick, Revert, Reset](#-cherry-pick-revert-reset)
- [Stash, Bisect, Reflog, Worktree](#-stash-bisect-reflog-worktree)
- [Branching Strategies](#-branching-strategies)
- [Conflict Mastery](#-conflict-mastery)
- [Monorepo Strategies](#-monorepo-strategies)
- [Hooks ও Productivity](#-hooks-ও-productivity)
- [Recovery Scenarios](#-recovery-scenarios)
- [Performance ও Security](#-performance-ও-security)
- [Bangladesh Context](#-bangladesh-context)
- [Anti-patterns ও Checklist](#-anti-patterns-ও-checklist)

---

## 🧬 Git Internals — মূলকাঠামো

Git আসলে একটি **content-addressable filesystem** যার উপর একটি version control UI বসানো আছে। প্রতিটি object-এর key হলো তার content-এর SHA-1 (নতুন repo-তে SHA-256) hash।

```
                      ┌──────────────────────────┐
                      │      Working Tree        │  (আপনার ফাইল)
                      └────────────┬─────────────┘
                                   │ git add
                                   ▼
                      ┌──────────────────────────┐
                      │    Staging Area / Index  │  (.git/index)
                      └────────────┬─────────────┘
                                   │ git commit
                                   ▼
                      ┌──────────────────────────┐
                      │  Object Database (.git/objects) │
                      │  ┌────────┐ ┌────────┐         │
                      │  │ blob   │ │ tree   │         │
                      │  └────────┘ └────────┘         │
                      │  ┌────────┐ ┌────────┐         │
                      │  │ commit │ │ tag    │         │
                      │  └────────┘ └────────┘         │
                      └──────────────────────────┘
                                   ▲
                                   │ refs point to commits
                      ┌──────────────────────────┐
                      │   refs/heads/main        │  → SHA
                      │   refs/heads/feature/x   │  → SHA
                      │   HEAD → refs/heads/main │
                      └──────────────────────────┘
```

### কেন এই ডিজাইন?

- **Immutability**: একটি object একবার তৈরি হলে আর পরিবর্তন হয় না — শুধু নতুন object তৈরি হয়।
- **Deduplication**: একই content-এর জন্য একই hash → disk-এ একবার সংরক্ষণ।
- **Integrity**: SHA hash দিয়ে যেকোনো পরিবর্তন ধরা পড়ে।
- **Distributed**: প্রতিটি clone-এ পুরো history থাকে।

---

## 🧱 Object Model

Git-এ মাত্র চার ধরনের object আছে। সবগুলো `.git/objects/` এ zlib-compressed অবস্থায় থাকে।

### ১. Blob (Binary Large Object)

ফাইলের content (কোনো metadata নেই, এমনকি filename ও না)।

```bash
$ echo "Hello bKash" | git hash-object --stdin
8a3c... (SHA)

$ echo "Hello bKash" | git hash-object -w --stdin   # লিখে রাখো
$ git cat-file -p 8a3c...
Hello bKash

$ git cat-file -t 8a3c...
blob
```

### ২. Tree

একটি directory snapshot — তার ভিতরে কী কী blob/sub-tree আছে এবং তাদের নাম+permission।

```
tree 4f2a...
├── 100644 blob 8a3c...    README.md
├── 100755 blob 9b7d...    deploy.sh
└── 040000 tree 5e1f...    src/
                            ├── 100644 blob a2b1...  index.php
                            └── 100644 blob c3d4...  helper.php
```

```bash
$ git cat-file -p HEAD^{tree}
100644 blob 8a3c...    README.md
100755 blob 9b7d...    deploy.sh
040000 tree 5e1f...    src
```

### ৩. Commit

একটি tree-এর pointer + parent commit(s) + author + committer + message।

```
commit a1b2c3...
    tree 4f2a...
    parent 7e8f...
    author Rakib <rakib@bkash.com> 1714000000 +0600
    committer Rakib <rakib@bkash.com> 1714000000 +0600

    feat(payment): bKash IPN webhook signature verify
```

**একটি merge commit-এ ২+ parent থাকে।** Octopus merge-এ ২ এর বেশি parent ও থাকতে পারে।

### ৪. Tag (annotated)

একটি commit-এর pointer + tag-এর metadata + signature (optional)।

```bash
$ git tag -a v2.4.0 -m "Daraz mega sale release" -s   # GPG signed
$ git cat-file -p v2.4.0
object a1b2c3...
type commit
tag v2.4.0
tagger Rakib <rakib@daraz.com> 1714000000 +0600

Daraz mega sale release
-----BEGIN PGP SIGNATURE-----
...
```

### Refs — Movable Pointer

Refs হলো simple text file যেখানে একটি commit SHA লেখা থাকে।

```bash
$ cat .git/refs/heads/main
a1b2c3d4e5f6...

$ cat .git/HEAD
ref: refs/heads/main          # symbolic ref
```

**Branch = একটি movable ref.** যখন আপনি `git commit` করেন, current branch ref নতুন commit-এ এগিয়ে যায়।

---

## 📁 .git/ ডিরেক্টরি বিশ্লেষণ

```
.git/
├── HEAD                 ← current branch বা detached commit
├── config               ← local config
├── description          ← gitweb-এর জন্য
├── index                ← staging area (binary format)
├── hooks/               ← client-side hooks (sample sহ)
│   ├── pre-commit.sample
│   ├── pre-push.sample
│   └── ...
├── info/
│   └── exclude          ← local-only ignore (gitignore-এর বাইরে)
├── logs/                ← reflog
│   ├── HEAD
│   └── refs/heads/main
├── objects/
│   ├── 8a/3c...         ← loose objects (প্রথম 2 chars = subdir)
│   ├── pack/
│   │   ├── pack-xxx.pack    ← packed objects (compressed)
│   │   ├── pack-xxx.idx     ← index for fast lookup
│   │   └── pack-xxx.bitmap  ← reachability bitmap
│   └── info/
└── refs/
    ├── heads/           ← local branches
    ├── remotes/         ← remote-tracking branches
    └── tags/
```

### Loose vs Packed Objects

প্রথমে object গুলো **loose** হিসেবে individual file-এ থাকে। `git gc` চললে এদের একটা **pack file**-এ একসাথে compress করে রাখে (delta compression — একটা object-এর diff অন্যটার সাপেক্ষে)।

```bash
$ git count-objects -v
count: 245              # loose objects
size: 1240 KiB
in-pack: 18432          # packed
packs: 3
size-pack: 45 MiB
```

```bash
### `git gc --aggressive` এবং Object Packing — বিস্তারিত আলোচনা

<!-- 
এই বিভাগটি Git এর aggressive garbage collection কমান্ড সম্পর্কে আলোচনা করে।
এটি ব্যাখ্যা করে যে `git gc --aggressive` চালানোর সময় কী ঘটে এবং এর প্রভাবগুলি কী।

#### `git gc --aggressive` এবং Object Packing — বিস্তারিত আলোচনা

`git gc` (garbage collection) হলো Git-এর housekeeping command যা loose object গুলোকে packed format-এ compress করে repo optimize করে। `--aggressive` flag এটিকে আরও intensive করে তোলে।

```bash
git gc --aggressive    # বড় repo-তে ১৫-৩০ মিনিট পর্যন্ত সময় লাগতে পারে
git repack -ad         # পুরোনো pack delete করে পুনরায় pack তৈরি
```

**কী ঘটে?**

1. **Loose objects কে pack করে**: `.git/objects/ab/cdef...` এর পরিবর্তে compressed `.git/objects/pack/` file
2. **Delta compression**: একই ধরনের blob/tree-এর মধ্যে diff store করে (storage ৪০-৭০% কমাতে পারে)
3. **Reachability bitmap তৈরি করে**: পরবর্তী push/fetch দ্রুত করার জন্য

**পারফরম্যান্স লাভ**:
- Clone/fetch দ্রুত (জিপড ডেটা transmit)
- `git log` দ্রুত (commit-graph সহ)
- Disk space ৫০-৭০% সাশ্রয়

**সতর্কতা**:
- ⚠️ Production repo-এ সরাসরি `--aggressive` চালাবেন না — pack operation CPU-intensive
- নিশাচর সময়ে schedule করুন (GitHub Actions cron, GitLab scheduled pipeline)
- সময়কালীন: সপ্তাহে ১ বার যথেষ্ট (daily না)

**Configuration**:
```ini
[gc]
  aggressive = false
  auto = 6700
  autoPackLimit = 50
```

$ git gc --aggressive    # বড় repo-তে অনেক সময় লাগে; production-এ সাবধানে
$ git repack -ad         # repack করো, পুরোনো pack delete
```

### SHA-1 → SHA-2 Transition

SHA-1 collision (SHAttered, 2017) এর পর Git SHA-256 support যোগ করেছে (Git 2.29+, experimental)।

```bash
$ git init --object-format=sha256 myrepo
```

বাস্তবে এখনো GitHub/GitLab universal SHA-256 support দেয় না — interop tool আসছে।

---

## 🛠️ Daily Commands Beyond Basics

### `git log` — শুধু history না, একটি investigation tool

```bash
# গত সপ্তাহে Rakib কী করেছে?
git log --author="Rakib" --since="1 week ago" --oneline

# একটি ফাইল কে পরিবর্তন করেছে?
git log -p src/payment/bkash.php

# pickaxe — যে commit-এ "ipn_secret" string যোগ/বাদ পড়েছে
git log -S "ipn_secret" --all

# regex পিকঅ্যাক্স
git log -G "preg_match.*\\d{11}"

# commit message search
git log --grep="hotfix" --grep="urgent" --all-match -i

# graph
git log --all --graph --oneline --decorate

# কোনো ফাইলের রেনেমের পরও history
git log --follow src/utils/Helper.php

# diff সহ
git log -p --follow -- src/utils/Helper.php
```

### `git blame` — কোডের জন্ম-পরিচয়

```bash
# পুরো ফাইল
git blame src/payment/bkash.php

# নির্দিষ্ট লাইন range
git blame -L 45,60 src/payment/bkash.php

# function এর সব লাইন
git blame -L :verifyIPN src/payment/bkash.php

# ignore whitespace + reformatting commit (boring commits skip)
git blame --ignore-rev abc123 src/payment/bkash.php

# .git-blame-ignore-revs file ব্যবহার (Daraz-এ Prettier migration commit ignore করার জন্য)
git config blame.ignoreRevsFile .git-blame-ignore-revs
```

### `git diff` — versatility

```bash
git diff                          # working tree vs index
git diff --staged                 # index vs HEAD
git diff main..feature            # দুটি branch
git diff main...feature           # feature এর unique changes (merge-base থেকে)
git diff --stat                   # summary
git diff --word-diff              # word-level
git diff --color-words='[A-Za-z0-9_]+'   # custom token
```

---

## 🔄 Rebase Mastery

Rebase আপনার commit গুলোকে অন্য একটি base-এর উপর "replay" করে — মানে নতুন commit তৈরি হয় (নতুন SHA সহ)।

### `git rebase -i` — Interactive Rebase

```bash
$ git rebase -i HEAD~5
```

Editor-এ আসবে:

```
pick   a1b2c3d feat(order): add bKash payment
pick   b2c3d4e fix typo in error message
pick   c3d4e5f wip
pick   d4e5f6g feat(order): add Nagad payment
pick   e5f6g7h hotfix: bKash callback URL

# Commands:
# p, pick    = use commit
# r, reword  = use commit, but edit message
# e, edit    = use commit, but stop for amending
# s, squash  = use commit, but meld into previous
# f, fixup   = like squash, but discard commit message
# d, drop    = remove commit
# x, exec    = run shell command
# b, break   = stop rebase here
```

**Practical example — clean PR before review:**

```
pick   a1b2c3d feat(order): add bKash payment
fixup  b2c3d4e fix typo in error message       # → আগের commit-এর সাথে merge
drop   c3d4e5f wip                             # → ফেলে দাও
pick   d4e5f6g feat(order): add Nagad payment
fixup  e5f6g7h hotfix: bKash callback URL      # → bKash commit-এ merge
```

### `--autosquash` + `--fixup` workflow

প্রতিদিনের workflow-এ "wip"/"fix" commit না বানিয়ে এভাবে করুন:

```bash
# একটা পুরোনো commit-এ change যোগ করতে চাই
git commit --fixup=a1b2c3d
git commit --fixup=a1b2c3d
git commit --fixup=d4e5f6g

# এখন rebase করলে autosquash এই fixup-গুলো নিজে নিজে সঠিক জায়গায় বসিয়ে দেবে
git rebase -i --autosquash HEAD~10
```

`.gitconfig`-এ একবার সেট করুন:

```ini
[rebase]
    autosquash = true
    autostash = true       # rebase-এর আগে dirty changes auto-stash
```

### `git rebase --onto` — Cross-Branch Transplant

Scenario: Daraz-এ আপনি `release/v3` থেকে একটা `feature/cart-promo` cut করেছিলেন, কিন্তু তা আসলে `main`-এ যাওয়া উচিত ছিল।

```
release/v3        A───B───C  ← here cart-promo was branched
                       \
feature/cart-promo      D───E───F  ← আপনার কাজ

main              X───Y───Z
```

```bash
git rebase --onto main release/v3 feature/cart-promo
```

ফলাফল:

```
main              X───Y───Z───D'───E'───F'  ← cart-promo এখন main-এর উপর
```

**ফর্মুলা**: `git rebase --onto <new-base> <old-base> <branch>` — `<old-base>..<branch>` range-এর commit গুলো `<new-base>` এর উপর replay হবে।

### Rebase vs Merge — কখন কোনটি?

| Aspect | Rebase | Merge |
|--------|--------|-------|
| History | Linear, পরিষ্কার | Branching দেখা যায় |
| Commit SHA | পরিবর্তন হয় | অপরিবর্তিত |
| Conflict resolution | প্রতিটি commit-এ আলাদা | একবার merge-commit এ |
| Public branch-এ? | ❌ কখনো না | ✅ |
| Reverts? | কঠিন | সহজ (revert merge commit) |

**Golden rule**: যে branch অন্যরা use করছে, তা rebase করবেন না।

---

## 🍒 Cherry-pick, Revert, Reset

### `git cherry-pick`

একটি বা একাধিক commit-কে current branch-এ "copy" করে।

```bash
# একটি commit
git cherry-pick a1b2c3d

# range (a1b2c3d exclusive, e5f6g7h inclusive)
git cherry-pick a1b2c3d..e5f6g7h

# inclusive range
git cherry-pick a1b2c3d^..e5f6g7h

# একাধিক
git cherry-pick a1b2c3d e5f6g7h

# original commit-এর reference রাখো (audit trail)
git cherry-pick -x a1b2c3d
# message-এ যোগ হবে: (cherry picked from commit a1b2c3d)

# conflict হলে
git cherry-pick a1b2c3d
# ... conflict ঠিক করে
git add .
git cherry-pick --continue
# অথবা
git cherry-pick --abort
git cherry-pick --skip
```

**BD Use case (bKash hotfix)**: `release/2.4` branch-এ একটি critical security fix দরকার, কিন্তু সেটা `main`-এ already আছে।

```bash
git checkout release/2.4
git cherry-pick -x abc1234         # main-এর fix commit
git push origin release/2.4
```

### `git revert` — Public-Safe Undo

Revert একটি **নতুন commit** তৈরি করে যা পুরোনো commit-কে বাতিল করে। History পরিবর্তন হয় না।

```bash
git revert a1b2c3d                 # নতুন commit
git revert --no-commit a1b2c3d e5f6g7h    # একসাথে multiple, ম্যানুয়ালি commit
```

**Merge commit revert করার সময় ফাঁদ:**

```bash
git revert -m 1 <merge-sha>
# -m 1 মানে "first parent-এর state-এ ফিরে যাও"
# অর্থাৎ feature branch-এর সব change বাদ দাও
```

⚠️ **সাবধান**: একবার merge-commit revert করলে, পরে সেই branch আবার merge করতে চাইলে সেই revert-ও revert করতে হবে, নাহলে Git ভাববে কাজ already হয়ে গেছে।

### `git reset` — Time Machine

```
                  Working Tree    Index    HEAD
git reset --soft       ✗            ✗       ✓     (changes index/working এ থাকে)
git reset --mixed      ✗            ✓       ✓     (default; index unstage হয়)
git reset --hard       ✓            ✓       ✓     (☠️ working tree সহ wipe!)
```

```bash
# শেষ commit-কে unstage করো, কিন্তু changes রাখো
git reset --soft HEAD~1

# শেষ commit-কে unstage + unstage all (default)
git reset HEAD~1

# ☠️ সব kill — last commit + working changes সব gone
git reset --hard HEAD~1

# ORIG_HEAD = reset/merge-এর আগের HEAD; auto সেট হয়
git reset --hard ORIG_HEAD          # reset undo!
```

**`reset` vs `revert`**:
- `reset`: history rewrite — local/private branch-এ ভালো।
- `revert`: history preserve — public branch-এ লাগে।

---

## 🧰 Stash, Bisect, Reflog, Worktree

### `git stash` — অসমাপ্ত কাজ পার্ক করুন

```bash
git stash                          # working + index সরিয়ে রাখো
git stash push -m "WIP: bKash IPN" # message সহ
git stash push -u                  # untracked ফাইল ও সহ
git stash push -p                  # interactive (নির্দিষ্ট hunk)
git stash push src/payment/        # নির্দিষ্ট path

git stash list
# stash@{0}: WIP on feature/bkash: a1b2c3d feat: payment
# stash@{1}: On main: experimental refactor

git stash show -p stash@{1}        # diff দেখো
git stash apply stash@{1}          # apply করো, stash থাকবে
git stash pop                      # apply + delete (top stash)
git stash drop stash@{1}
git stash clear                    # সব মুছে দাও

# stash থেকে নতুন branch
git stash branch feature/bkash-resume stash@{0}
```

### `git bisect` — Binary Search for Regression

Pathao-এর checkout flow হঠাৎ ভেঙে গেছে, last week-এ কাজ করত। কোন commit ভাঙল?

```bash
git bisect start
git bisect bad HEAD                    # এখন বাজে
git bisect good v2.3.0                 # গত release-এ ভালো ছিল

# Git মাঝামাঝি commit-এ checkout করবে
# আপনি test করুন
git bisect good   # বা
git bisect bad

# ... কয়েক step পরে ...
# a1b2c3d is the first bad commit

git bisect reset
```

**Automated bisect** — script দিয়ে:

```bash
git bisect start HEAD v2.3.0
git bisect run npm test -- --grep "checkout"
# Git নিজে নিজে কালপ্রিট commit খুঁজে বের করবে
```

`run` command exit code: `0` = good, `1-127` (except 125) = bad, `125` = skip।

### `git reflog` — Safety Net

Reflog হলো আপনার HEAD/branch ref-এর local history (push হয় না)।

```bash
git reflog
# a1b2c3d HEAD@{0}: commit: feat: Daraz cart
# 9f8e7d6 HEAD@{1}: reset: moving to HEAD~1
# c5d4e3f HEAD@{2}: commit: WIP cart
# ...

# `reset --hard` করেছি, বাঁচাও!
git reset --hard HEAD@{2}

# branch-এর reflog ও আছে
git reflog show feature/x
```

**Default expiry**: reachable commits 90 days, unreachable 30 days। বাড়াতে চান?

```bash
git config --global gc.reflogExpire "1 year"
git config --global gc.reflogExpireUnreachable "3 months"
```

### `git worktree` — একই repo, একাধিক working dir

PR review করতে গিয়ে আপনার current feature stash করতে চান না? Worktree বানান!

```bash
git worktree add ../myapp-pr-review origin/feature/pr-123
cd ../myapp-pr-review
# এখানে পুরোপুরি স্বাধীন working tree, কিন্তু .git share করে

git worktree list
git worktree remove ../myapp-pr-review
git worktree prune
```

বড় repo (Daraz monorepo)-তে ক্লোন বার বার করার চেয়ে অনেক দ্রুত।

### `git submodule` vs `git subtree` vs Tools

| | Submodule | Subtree | Tools (Nx, Turborepo) |
|-|-----------|---------|------------------------|
| Pointer-only | ✅ | ❌ (full code) | N/A |
| Easy contribution | কঠিন | মাঝারি | সহজ |
| Single clone | ❌ (extra step) | ✅ | ✅ |
| Recommended? | পরিহার | কখনো | মনোরেপোতে preferred |

**Submodule example** (যদি অপরিহার্য হয়):

```bash
git submodule add git@github.com:bkash/sdk-php.git vendor/bkash-sdk
git submodule update --init --recursive
git submodule update --remote          # latest pull
```

আজকাল বেশিরভাগ ক্ষেত্রে **package manager** (composer, npm) ভালো।

---

## 🌿 Branching Strategies

### ১. Trunk-Based Development (TBD)

```
main  ──•──•──•──•──•──•──•──►   (everyone commits here, daily)
         \  \      \
          short-lived (≤1 day) feature branches
```

- Feature flags দিয়ে incomplete code production-এ যায়।
- প্রচুর CI test দরকার।
- **Use case**: Pathao, Google, Facebook, high-velocity teams।

### ২. GitHub Flow

```
main  ──•───────•───────•───►
         \     /  \     /
          feat/x   bugfix/y
```

- Feature branch → PR → review → squash-merge to main → deploy।
- Simple, modern, ছোট team-এর জন্য আদর্শ।

### ৩. Git Flow (legacy, কখনো useful)

```
main          ──•───────────•──►   (production releases শুধু)
                 \         /
release/2.4      ──•───•──•
                  \   /
develop      ──•──•──•──•──►       (integration)
                \    /
feature/x        •──•
hotfix/y      ──────•──•           (production থেকে)
```

- Multiple version maintenance (banking software, hardware firmware) এ কাজে লাগে।
- বেশিরভাগ web app-এর জন্য overkill।
- **bKash core banking** এর মতো compliance-heavy product-এ এখনো use হয়।

### Pull Request Etiquette

| Strategy | কখন | History |
|----------|-----|---------|
| **Merge commit** (`git merge --no-ff`) | বড় feature, context রাখতে চান | branching visible |
| **Squash & merge** | ছোট PR, "wip" commit আছে | linear, ১টি commit |
| **Rebase & merge** | পরিষ্কার commit history, no merge commit | linear, সব commit |

**GitHub-এ branch protection rule**:

```
✅ Require PR before merge
✅ Require status checks (CI)
✅ Require code review (≥2 approvers for production)
✅ Require linear history (rebase/squash only)
✅ Require signed commits (bKash, financial)
✅ Restrict who can push to main
```

### Merge Queue (GitHub Merge Queue, MergifyIO)

পুরোনো ঝামেলা: PR-A pass, PR-B pass আলাদাভাবে — কিন্তু একসাথে merge করলে main ভাঙে। Merge Queue সমাধান করে — main-এ merge হওয়ার আগে combined CI run করে।

```yaml
# .github/merge_queue.yml (GitHub native)
required_status_checks:
  - "CI / Build"
  - "CI / Test"
merge_method: squash
```

---

## ⚔️ Conflict Mastery

### Three-Way Merge Mechanics

Git তিনটি version দেখে: ours, theirs, common ancestor (merge-base)।

```
        Common Ancestor
              │
        ┌─────┴─────┐
       ours        theirs
        │            │
        └────┬───────┘
             ▼
          Merged
```

Conflict marker:

```
<<<<<<< HEAD (ours)
$gateway = 'bkash';
=======
$gateway = 'nagad';
>>>>>>> feature/nagad (theirs)
```

```bash
git checkout --ours src/payment.php       # ours রাখো
git checkout --theirs src/payment.php     # theirs রাখো
git checkout --merge src/payment.php      # marker সহ আবার দেখাও

# diff3 style — common ancestor ও দেখায় (অনেক বেশি helpful!)
git config --global merge.conflictstyle diff3
```

```
<<<<<<< HEAD
$gateway = 'bkash';
||||||| merged common ancestors
$gateway = 'cod';
=======
$gateway = 'nagad';
>>>>>>> feature/nagad
```

### `git mergetool`

```bash
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'
git mergetool
```

বড় conflict-এ `meld`, `kdiff3`, `Beyond Compare` GUI বেশ কাজে দেয়।

### `rerere` — Reuse Recorded Resolution

একই conflict বারবার? `rerere` solution remember করে রাখে।

```bash
git config --global rerere.enabled true
git config --global rerere.autoUpdate true
```

বিশেষত long-lived feature branch + frequent rebase-এ অসাধারণ।

### Rename + Edit Conflict

```bash
# X file rename করেছে, আমি edit করেছি একই file
git config --global merge.renameLimit 7000
git config --global diff.renames true
```

Rename detection threshold বাড়ালে Git ভালোভাবে মেলাবে।

---

## 🏢 Monorepo Strategies

### কেন Monorepo?

Google, Meta, Microsoft সবাই use করে। সুবিধা:
- Atomic refactor across projects
- Single source of truth
- Easy dependency management
- Code sharing

অসুবিধা:
- বড় repo size, slow operation
- CI complexity
- Tooling needed

### Polyrepo vs Monorepo

| | Monorepo | Polyrepo |
|-|----------|----------|
| Repo count | ১ | অনেক |
| Cross-project refactor | atomic commit | coordinated PRs |
| CI | affected-only চাই | independent |
| Access control | CODEOWNERS-নির্ভর | repo-level |
| Storage | বড় | ছোট/ছোট |
| Onboarding | tooling lagto | simple clone |
| Best for | tightly coupled | independent teams |

### Monorepo Tools

| Tool | Stack | Caching | Best for |
|------|-------|---------|----------|
| **Nx** | JS/TS, Java, .NET | Local + remote (Nx Cloud) | Frontend-heavy, BD startups |
| **Turborepo** | JS/TS | Local + Vercel | Vercel-deployed apps |
| **Lerna** | JS/TS | None (legacy) | লেগেসি |
| **pnpm workspaces** | JS/TS | None | Simple JS monorepo |
| **Bazel** | Polyglot, Google | Massive remote | Mega-scale |
| **Pants** | Polyglot, Twitter | Remote | Python-heavy |
| **Buck2** | Polyglot, Meta | Remote | Mega-scale |

**Affected-only build (Nx example)**:

```bash
nx affected:build --base=origin/main --head=HEAD
nx affected:test --base=origin/main
```

CI-তে শুধু changed project ও তাদের dependent build/test হবে — Daraz-এ ৩০ মিনিটের build ৩ মিনিটে নামতে পারে।

### Sparse Checkout & Partial Clone

বড় monorepo-তে full clone অসম্ভব slow। সমাধান:

```bash
# Partial clone — blob লোড on demand
git clone --filter=blob:none git@github.com:daraz/monorepo.git

# Treeless clone — আরও কম
git clone --filter=tree:0 git@github.com:daraz/monorepo.git

# Shallow clone — শুধু latest N commit
git clone --depth=1 git@github.com:daraz/monorepo.git

# Sparse checkout — শুধু আপনার team-এর folder
cd monorepo
git sparse-checkout init --cone
git sparse-checkout set apps/seller-portal libs/shared
```

### VFS for Git, Scalar

Microsoft-এর Windows source code (~300GB) এর জন্য VFS for Git বানিয়েছিল — virtual filesystem, file লাগলে তখন download। এখন **Scalar** সেই tech-কে portable করেছে:

```bash
scalar clone https://github.com/microsoft/scalar
```

### CODEOWNERS

```
# .github/CODEOWNERS

# Default
*                                  @daraz/platform

# Frontend
/apps/web/                         @daraz/frontend-team
/apps/web/checkout/                @daraz/checkout-squad @daraz/payments

# Payment-critical (২ অনুমোদন বাধ্যতামূলক)
/libs/payment/                     @daraz/payments-lead @daraz/security
/libs/bkash-sdk/                   @daraz/payments-lead @daraz/security

# Infra
/.github/workflows/                @daraz/devops
*.tf                               @daraz/devops
```

---

## 🪝 Hooks ও Productivity

### Aliases — `.gitconfig`

```ini
[alias]
    s = status -sb
    co = checkout
    br = branch
    cm = commit -m
    cma = commit --amend --no-edit
    lg = log --oneline --graph --decorate --all
    last = log -1 HEAD --stat
    unstage = reset HEAD --
    undo = reset --soft HEAD~1
    please = push --force-with-lease       # safer force-push
    wip = !git add -A && git commit -m "wip"
    fixup = commit --fixup
    squash = "!f() { git rebase -i --autosquash HEAD~$1; }; f"
    cleanup = "!git branch --merged main | grep -v 'main\\|master' | xargs -n 1 git branch -d"
    aliases = config --get-regexp ^alias\\.
```

⚠️ `git push --force-with-lease` ALWAYS use করুন `--force` এর বদলে — অন্যের commit accidentally overwrite হবে না।

### Client-side Hooks

`.git/hooks/` (committed হয় না — শুধু local)। Team-wide hook-এর জন্য `pre-commit`, `husky`, `captainhook` framework।

#### `pre-commit` framework (Python, polyglot)

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
        args: ['--maxkb=500']
      - id: detect-private-key
      - id: no-commit-to-branch
        args: [--branch, main, --branch, production]

  - repo: https://github.com/pre-commit/mirrors-eslint
    rev: v9.0.0
    hooks:
      - id: eslint
        files: \.(js|ts|tsx)$

  - repo: https://github.com/squizlabs/PHP_CodeSniffer
    rev: 3.10.0
    hooks:
      - id: php-cs

  - repo: https://github.com/zricethezav/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
```

```bash
pip install pre-commit
pre-commit install              # .git/hooks/pre-commit ইনস্টল
pre-commit install --hook-type commit-msg
pre-commit run --all-files      # initial scan
```

#### Husky + lint-staged (JS/TS)

```bash
npm install -D husky lint-staged
npx husky init
```

```json
// package.json
{
  "lint-staged": {
    "*.{js,ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{json,md,yml}": ["prettier --write"]
  },
  "scripts": {
    "prepare": "husky"
  }
}
```

```sh
# .husky/pre-commit
npx lint-staged
```

#### Captainhook (PHP)

```json
// captainhook.json
{
    "pre-commit": {
        "enabled": true,
        "actions": [
            { "action": "vendor/bin/phpcs --standard=PSR12 src/" },
            { "action": "vendor/bin/phpstan analyse src/ --level=8" },
            { "action": "vendor/bin/phpunit --testsuite=unit" }
        ]
    },
    "commit-msg": {
        "enabled": true,
        "actions": [
            { "action": "\\CaptainHook\\App\\Hook\\Message\\Action\\Regex",
              "options": { "regex": "#^(feat|fix|docs|chore|refactor)(\\(.+\\))?: .{1,72}$#" }
            }
        ]
    }
}
```

```bash
composer require --dev captainhook/captainhook
vendor/bin/captainhook install
```

### Conventional Commits

```
<type>(<scope>): <subject>

<body>

<footer>
```

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `chore`, `ci`, `build`, `revert`।

```
feat(payment): add bKash IPN signature verification

Verify HMAC-SHA256 signature from bKash callback to prevent
replay attacks. Closes #1234.

BREAKING CHANGE: requires BKASH_IPN_SECRET env var
Co-authored-by: Karim <karim@bkash.com>
```

**Tools**:
- `commitizen` — interactive commit message
- `semantic-release` — auto version bump + changelog from commits
- `changesets` — better for monorepos

### Signed Commits — GPG / SSH

#### GPG

```bash
gpg --full-generate-key                    # RSA 4096
gpg --list-secret-keys --keyid-format=long
git config --global user.signingkey ABC123DEF456
git config --global commit.gpgsign true
git config --global tag.gpgsign true

gpg --armor --export ABC123DEF456 | pbcopy
# → GitHub > Settings > SSH and GPG keys
```

#### SSH signing (Git 2.34+, simpler)

```bash
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true
```

`bKash`-এ branch protection-এ "Require signed commits" enable — verified ✅ ছাড়া PR merge হবে না।

### Git LFS

বড় binary (image, video, model file) commit করলে repo phulko।

```bash
git lfs install
git lfs track "*.psd" "*.mp4" "*.bin"
git add .gitattributes
git add huge-design.psd
git commit -m "design: add product mockup"
```

LFS storage GitHub-এ paid (1GB free)। বিকল্প: S3 + script।

---

## 🚑 Recovery Scenarios

### ১. "আমি `reset --hard` করে ফেলেছি!"

```bash
git reflog
# d4e5f6g HEAD@{1}: commit: my important work
# a1b2c3d HEAD@{0}: reset: moving to HEAD~3

git reset --hard HEAD@{1}              # বা SHA সরাসরি
```

### ২. "ভুল branch-এ commit করেছি"

```bash
# main-এ commit করেছি, feature/x-এ যাওয়া উচিত ছিল
git log --oneline -3                   # SHA নোট করুন
# a1b2c3d
# 9f8e7d6   ← এই দুটো ভুল জায়গায়
# 5e4d3c2

git checkout feature/x
git cherry-pick 5e4d3c2 9f8e7d6
git checkout main
git reset --hard a1b2c3d~2             # main কে আগের জায়গায় ফেরত
```

### ৩. "Secret commit হয়ে গেছে!"

⚠️ **প্রথমে secret rotate/revoke করুন!** Git থেকে মুছলেও কেউ আগে clone করে থাকতে পারে।

```bash
# git-filter-repo (recommended, modern)
pip install git-filter-repo

git filter-repo --path .env --invert-paths           # path মুছো
git filter-repo --replace-text replacements.txt      # string replace
# replacements.txt:
# AWS_SECRET_KEY=AKIA...==><REMOVED>

# তারপর force-push (সবাইকে জানিয়ে)
git push --force --all
git push --force --tags

# পুরোনো branch backup-এ থাকলে delete
```

**BFG Repo-Cleaner** (alternative, faster on huge repo):

```bash
java -jar bfg.jar --delete-files .env my-repo.git
java -jar bfg.jar --replace-text passwords.txt my-repo.git
cd my-repo.git
git reflog expire --expire=now --all && git gc --prune=now --aggressive
```

**Prevention**: `gitleaks` pre-commit hook + GitHub secret scanning।

### ৪. "Public branch rebase করে ফেলেছি!"

আপনি `main` rebase করে force-push করেছেন। সবার local repo এখন divergent।

**Fix**:
1. টিমকে message দিন: "STOP, do not push"।
2. নতুন SHA-তে সবাই reset করুক:
   ```bash
   git fetch origin
   git checkout main
   git reset --hard origin/main
   ```
   কারো local commit থাকলে সে আগে cherry-pick করে রাখুক।

**Prevention**: branch protection-এ force-push disable।

### ৫. Stash হারিয়ে গেছে?

```bash
# stash drop করে ফেলেছেন? 
git fsck --unreachable --no-reflogs | grep commit
# unreachable commit cd3a4e5...
git show cd3a4e5
git stash apply cd3a4e5            # বা cherry-pick
```

### ৬. ভুল commit অসংশোধিত push হয়েছে

```bash
git revert HEAD              # public-safe
git push
```

---

## ⚡ Performance ও Security

### Performance Optimization

```bash
# Commit-graph file (faster log/diff)
git config --global core.commitGraph true
git config --global gc.writeCommitGraph true
git commit-graph write --reachable

# fsmonitor (Watchman বা built-in)
git config --global core.fsmonitor true
git config --global core.untrackedCache true

# auto gc
git config --global gc.auto 6700
git config --global gc.autoPackLimit 50

# Parallel index preload
git config --global core.preloadIndex true

# Repack with bitmap
git repack -a -d --depth=250 --window=250
```

### Security Hardening

```bash
# Force-push protection on shared branch
git config --global receive.denyDeletes true
git config --global receive.denyNonFastForwards true

# Always verify signatures
git config --global merge.verifySignatures true
```

**Server-side hooks** (GitHub Actions / Gitea / GitLab):
- gitleaks scan on push
- SBOM check (link to `11-security/supply-chain-security.md`)
- License compliance scanner

---

## 🇧🇩 Bangladesh Context

### Daraz — Monorepo with Nx + GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI
on: [pull_request]
jobs:
  affected:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }       # full history for affected
      - uses: nrwl/nx-set-shas@v4
      - run: pnpm install --frozen-lockfile
      - run: pnpm nx affected -t lint test build --parallel=3
```

**Practice**:
- monorepo (apps/seller-portal, apps/customer-web, apps/admin, libs/*)
- CODEOWNERS দিয়ে squad ownership
- Branch protection: 2 approver, signed commit
- Nx Cloud দিয়ে remote cache → CI ৭০% দ্রুত

### Pathao — Trunk-Based + Feature Flags

- সবাই `main`-এ daily commit
- Feature flag (LaunchDarkly / self-hosted Unleash) দিয়ে progressive rollout
- short-lived branch (≤ ১ দিন)
- Conventional Commits + semantic-release → auto deployment

```bash
git checkout -b feat/ride-fare-update main
# কাজ ⇒ PR ⇒ squash-merge ⇒ deploy
```

### bKash — Compliance-First

- Git Flow (release branch + hotfix)
- **Required signed commit** (GPG, hardware key)
- Audit log: every push → SIEM (Splunk)
- Branch protection: 3 reviewer (1 security)
- Pre-receive hook server-side গণনা: license check, secret scan
- gitleaks + truffleHog in CI

### ছোট BD Startup (10-15 dev) — GitHub Flow

- `main` + short-lived feature branches
- Husky + lint-staged + commitlint
- Squash-merge always
- GitHub Actions free tier দিয়ে CI
- Vercel/Render auto-deploy on main

---

## ⚠️ Anti-patterns

❌ **`git push --force` shared branch-এ** → `--force-with-lease` use করুন।
❌ **বিশাল binary commit** → LFS বা S3।
❌ **Long-lived feature branch** (২+ সপ্তাহ) → daily rebase + small PR।
❌ **Generic commit message** ("update", "fix") → Conventional Commits।
❌ **Secret commit** → gitleaks pre-commit hook + scanning।
❌ **`merge` সব জায়গায়** → শুধু integration-এ; দিনের কাজ rebase।
❌ **Hooks `.git/hooks/`-এ একা সাজানো** → pre-commit/Husky/Captainhook framework।
❌ **`git pull` blindly** → `git pull --rebase` config করুন (`pull.rebase = true`)।
❌ **All-in-one মেগা PR** → ছোট, focused PR।
❌ **`master` ভুল history থাকতে দেওয়া** → reflog আছে, fix করুন।
❌ **Submodule abuse** → package manager use করুন।
❌ **`git commit -am`** untracked file miss করে → `git add` সচেতনভাবে।
❌ **`.env` commit** → `.gitignore` + secret manager।

---

## ✅ Senior Engineer Git Checklist

**Setup:**
- [ ] `pull.rebase = true`, `rebase.autosquash = true`, `rebase.autostash = true`
- [ ] `merge.conflictstyle = diff3` (অথবা `zdiff3`)
- [ ] `rerere.enabled = true`
- [ ] `commit.gpgsign = true` (বা SSH signing)
- [ ] `core.fsmonitor = true`, `core.untrackedCache = true`
- [ ] Useful alias সেটআপ (`lg`, `please`, `cleanup`)

**Workflow:**
- [ ] Conventional Commits ব্যবহার করেন
- [ ] PR ছোট রাখেন (< 400 lines preferred)
- [ ] PR-এর আগে rebase + clean up commits
- [ ] `--force-with-lease`, কখনো `--force` না
- [ ] Long-lived branch নয়, daily integrate

**Hooks:**
- [ ] pre-commit framework / Husky / Captainhook সেটআপ
- [ ] Lint, format, secret-scan, type-check, fast tests
- [ ] commit-msg hook (Conventional Commits enforce)

**Repo health:**
- [ ] CODEOWNERS file
- [ ] Branch protection (CI + review + signed commits)
- [ ] `.gitignore`, `.gitattributes` সঠিক
- [ ] LFS for binary
- [ ] Merge queue (multi-team repo)
- [ ] gitleaks/secret-scanning enabled

**Skills:**
- [ ] Reflog দিয়ে recovery জানেন
- [ ] `bisect` use করতে পারেন
- [ ] Interactive rebase fluent
- [ ] Cherry-pick ও range-cherry-pick জানেন
- [ ] Conflict diff3 mode-এ resolve করেন
- [ ] `filter-repo` দিয়ে secret/বড় ফাইল clean করতে জানেন

**Monorepo:**
- [ ] Affected-only CI (Nx, Turborepo)
- [ ] Sparse checkout / partial clone যেখানে দরকার
- [ ] Remote cache (Nx Cloud, Turborepo Remote Cache)

---

> **মনে রাখুন**: Git-এ "data হারানো" বলতে কিছু নেই — শুধু "data খুঁজে পাচ্ছি না"। `git reflog`, `git fsck --lost-found`, `git filter-repo` — এই tool গুলো আপনার বন্ধু। একজন senior engineer হিসেবে আপনার কাজ team-কে এই safety net সম্পর্কে শেখানো এবং repo-কে clean, fast ও secure রাখা।
