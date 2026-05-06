# 🏦 Distributed Transactions — 2PC, 3PC এবং Saga-র সাথে তুলনা

## 📋 সুচিপত্র
- [সংজ্ঞা ও ধারণা](#সংজ্ঞা-ও-ধারণা)
- [বাস্তব জীবনের উদাহরণ](#বাস্তব-জীবনের-উদাহরণ)
- [Two-Phase Commit (2PC)](#two-phase-commit-2pc)
- [Three-Phase Commit (3PC)](#three-phase-commit-3pc)
- [ASCII ডায়াগ্রাম](#ascii-ডায়াগ্রাম)
- [PHP কোড উদাহরণ](#php-কোড-উদাহরণ)
- [JavaScript কোড উদাহরণ](#javascript-কোড-উদাহরণ)
- [2PC vs Saga তুলনা](#2pc-vs-saga-তুলনা)
- [কখন ব্যবহার করবেন / করবেন না](#কখন-ব্যবহার-করবেন--করবেন-না)

---

## 🎯 সংজ্ঞা ও ধারণা

**Distributed Transaction** হলো এমন একটি ট্রানজ্যাকশন যা একাধিক ডাটাবেস বা সার্ভিসে একই সাথে পরিবর্তন করে এবং সবগুলো পরিবর্তন হয় সম্পূর্ণ সফল হবে অথবা সম্পূর্ণ ব্যর্থ হবে (All-or-Nothing)।

### মূল সমস্যা:
যখন একটি অপারেশন একাধিক independent সিস্টেমে ঘটে, তখন কীভাবে নিশ্চিত করবেন যে সবগুলো একসাথে commit বা rollback হবে?

### ACID Properties Distributed Context-এ:
- **Atomicity**: সব node-এ হয় সব হবে, না হলে কিছুই হবে না
- **Consistency**: সব node-এর data consistent থাকবে
- **Isolation**: concurrent transactions একে অপরকে প্রভাবিত করবে না
- **Durability**: commit হলে permanently stored থাকবে

---

## 🌍 বাস্তব জীবনের উদাহরণ

### সোনালী ব্যাংক → জনতা ব্যাংক ফান্ড ট্রান্সফার

ধরুন, রহিম সাহেব সোনালী ব্যাংক থেকে জনতা ব্যাংকে ৫০,০০০ টাকা ট্রান্সফার করতে চান:

```
পদক্ষেপ ১: সোনালী ব্যাংক থেকে ৫০,০০০ টাকা কাটা
পদক্ষেপ ২: জনতা ব্যাংকে ৫০,০০০ টাকা যোগ করা
```

**সমস্যা**: যদি পদক্ষেপ ১ সফল হয় কিন্তু পদক্ষেপ ২ ব্যর্থ হয়?
- টাকা সোনালী থেকে কেটে গেছে কিন্তু জনতায় যোগ হয়নি!
- ৫০,০০০ টাকা হাওয়া হয়ে গেল! 😱

### বাংলাদেশ ব্যাংক RTGS (Real-Time Gross Settlement):
Bangladesh Bank-এর RTGS সিস্টেম exactly এই সমস্যা সমাধান করে — প্রতিদিন হাজার হাজার inter-bank transaction নিরাপদে সম্পন্ন করে।

---

## 🔄 Two-Phase Commit (2PC)

### Phase 1: Prepare (Voting Phase)
- Coordinator সব participant-কে জিজ্ঞেস করে: "তুমি কি commit করতে পারবে?"
- প্রতিটি participant হয় "YES" (prepare) অথবা "NO" (abort) ভোট দেয়
- YES মানে: "আমি commit করতে প্রস্তুত, data lock করে রেখেছি"

### Phase 2: Commit/Abort (Decision Phase)
- সব participant YES দিলে → Coordinator "COMMIT" পাঠায়
- একজনও NO দিলে → Coordinator "ABORT" পাঠায়

### Coordinator-এর ভূমিকা:
- Transaction শুরু করে
- সব participant-এর ভোট সংগ্রহ করে
- Final decision নেয় (commit/abort)
- Decision সব participant-কে জানায়

### Timeout Handling:
- Participant timeout → Coordinator ABORT সিদ্ধান্ত নেয়
- Coordinator timeout → Participant অনিশ্চিত থাকে (BLOCKING সমস্যা!)

---

## 🔄 Three-Phase Commit (3PC)

3PC হলো 2PC-এর non-blocking উন্নত সংস্করণ:

### Phase 1: CanCommit (প্রশ্ন)
- Coordinator: "তুমি কি commit করতে পারবে?"
- Participant: "হ্যাঁ" / "না"

### Phase 2: PreCommit (প্রস্তুতি)
- সবাই হ্যাঁ বললে → Coordinator "PreCommit" পাঠায়
- Participant lock নেয় এবং ACK পাঠায়

### Phase 3: DoCommit (চূড়ান্ত)
- সব ACK পেলে → Coordinator "DoCommit" পাঠায়
- Participant commit করে

### 3PC-এর সুবিধা:
- Coordinator crash হলেও participant timeout-এ সিদ্ধান্ত নিতে পারে
- Non-blocking: কেউ indefinitely অপেক্ষা করে না

---

## 📊 ASCII ডায়াগ্রাম

### 2PC Protocol Flow:

```
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│ Coordinator │         │ সোনালী ব্যাংক│         │ জনতা ব্যাংক  │
│  (BD Bank)  │         │ (Participant)│         │ (Participant)│
└──────┬──────┘         └──────┬──────┘         └──────┬──────┘
       │                       │                       │
       │   PHASE 1: PREPARE    │                       │
       │──────────────────────>│                       │
       │   "৫০K কাটতে পারবে?" │                       │
       │                       │                       │
       │   "৫০K যোগ করতে পারবে?"                      │
       │──────────────────────────────────────────────>│
       │                       │                       │
       │   VOTE: YES ✓         │                       │
       │<──────────────────────│                       │
       │                       │                       │
       │   VOTE: YES ✓                                 │
       │<──────────────────────────────────────────────│
       │                       │                       │
       │   PHASE 2: COMMIT     │                       │
       │──────────────────────>│                       │
       │   "COMMIT করো"        │                       │
       │                       │                       │
       │──────────────────────────────────────────────>│
       │                       │                       │
       │   ACK ✓               │                       │
       │<──────────────────────│                       │
       │                       │                       │
       │   ACK ✓                                       │
       │<──────────────────────────────────────────────│
       │                       │                       │
       ▼                       ▼                       ▼
  [Transaction Complete - ৫০,০০০ টাকা ট্রান্সফার সম্পন্ন]
```

### Failure Scenario - Participant Crash:

```
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│ Coordinator │         │ সোনালী ব্যাংক│         │ জনতা ব্যাংক  │
└──────┬──────┘         └──────┬──────┘         └──────┬──────┘
       │                       │                       │
       │   PREPARE             │                       │
       │──────────────────────>│                       │
       │──────────────────────────────────────────────>│
       │                       │                       │
       │   VOTE: YES ✓         │                       │
       │<──────────────────────│                       │
       │                       │                    ╔══╧══╗
       │   TIMEOUT! ⏰          │                    ║CRASH║
       │   (জনতা ব্যাংক        │                    ╚══╤══╝
       │    respond করেনি)     │                       │
       │                       │                       │
       │   ABORT! ✗            │                       │
       │──────────────────────>│                       │
       │   "Rollback করো"      │                       │
       │                       │                       │
       ▼                       ▼                       ▼
  [Transaction Aborted - কোনো টাকা কাটা হয়নি]
```

### 2PC vs 3PC Comparison:

```
    2PC (Two-Phase Commit)          3PC (Three-Phase Commit)
    ═══════════════════════          ═══════════════════════════

    ┌─────────┐                     ┌─────────────┐
    │ PREPARE │ ◄── Phase 1         │  CanCommit  │ ◄── Phase 1
    └────┬────┘                     └──────┬──────┘
         │                                 │
         ▼                                 ▼
    ┌─────────┐                     ┌─────────────┐
    │ COMMIT  │ ◄── Phase 2         │  PreCommit  │ ◄── Phase 2
    └─────────┘                     └──────┬──────┘
                                           │
                                           ▼
                                    ┌─────────────┐
                                    │  DoCommit   │ ◄── Phase 3
                                    └─────────────┘

    সমস্যা: Blocking!              সমাধান: Non-blocking
    Coordinator crash =             Coordinator crash =
    Participants আটকে যায়          Participants timeout-এ
                                    নিজে সিদ্ধান্ত নিতে পারে
```

---

## 💻 PHP কোড উদাহরণ

```php
<?php

/**
 * Two-Phase Commit Coordinator
 * সোনালী ব্যাংক থেকে জনতা ব্যাংকে ট্রান্সফার
 */
class TwoPhaseCommitCoordinator
{
    private array $participants = [];
    private string $transactionId;
    private string $state = 'INIT';
    private int $timeoutSeconds = 30;

    public function __construct()
    {
        $this->transactionId = uniqid('txn_', true);
    }

    public function addParticipant(TransactionParticipant $participant): void
    {
        $this->participants[] = $participant;
    }

    /**
     * মূল ট্রানজ্যাকশন execution
     */
    public function execute(array $operations): bool
    {
        echo "🏦 Transaction শুরু: {$this->transactionId}\n";

        // Phase 1: Prepare
        $allPrepared = $this->preparePhase($operations);

        if ($allPrepared) {
            // Phase 2: Commit
            return $this->commitPhase();
        } else {
            // Phase 2: Abort
            $this->abortPhase();
            return false;
        }
    }

    /**
     * Phase 1: সব participant-কে prepare করতে বলা
     */
    private function preparePhase(array $operations): bool
    {
        echo "\n📋 Phase 1: PREPARE শুরু...\n";
        $this->state = 'PREPARING';
        $votes = [];

        foreach ($this->participants as $index => $participant) {
            $operation = $operations[$index] ?? null;

            if (!$operation) {
                echo "  ❌ Participant {$index}: অপারেশন নেই\n";
                return false;
            }

            try {
                $startTime = microtime(true);
                $vote = $participant->prepare($this->transactionId, $operation);
                $elapsed = round((microtime(true) - $startTime) * 1000);

                if ($elapsed > $this->timeoutSeconds * 1000) {
                    echo "  ⏰ Participant {$index}: Timeout ({$elapsed}ms)\n";
                    return false;
                }

                $votes[] = $vote;
                $statusIcon = $vote ? '✓' : '✗';
                echo "  {$statusIcon} {$participant->getName()}: " . ($vote ? 'YES' : 'NO') . " ({$elapsed}ms)\n";

            } catch (\Exception $e) {
                echo "  💥 {$participant->getName()}: Error - {$e->getMessage()}\n";
                return false;
            }
        }

        return !in_array(false, $votes, true);
    }

    /**
     * Phase 2: সবাইকে commit করতে বলা
     */
    private function commitPhase(): bool
    {
        echo "\n✅ Phase 2: COMMIT শুরু...\n";
        $this->state = 'COMMITTING';
        $allCommitted = true;

        foreach ($this->participants as $participant) {
            try {
                $result = $participant->commit($this->transactionId);
                $icon = $result ? '✓' : '✗';
                echo "  {$icon} {$participant->getName()}: " . ($result ? 'Committed' : 'Failed') . "\n";

                if (!$result) {
                    $allCommitted = false;
                }
            } catch (\Exception $e) {
                echo "  💥 {$participant->getName()}: Commit Error - {$e->getMessage()}\n";
                $allCommitted = false;
            }
        }

        $this->state = $allCommitted ? 'COMMITTED' : 'FAILED';
        echo "\n🏁 Transaction ফলাফল: " . ($allCommitted ? 'সফল ✓' : 'ব্যর্থ ✗') . "\n";
        return $allCommitted;
    }

    /**
     * Abort: সবাইকে rollback করতে বলা
     */
    private function abortPhase(): void
    {
        echo "\n🚫 Phase 2: ABORT - সবাইকে Rollback করা হচ্ছে...\n";
        $this->state = 'ABORTING';

        foreach ($this->participants as $participant) {
            try {
                $participant->rollback($this->transactionId);
                echo "  ↩️ {$participant->getName()}: Rolled back\n";
            } catch (\Exception $e) {
                echo "  ⚠️ {$participant->getName()}: Rollback Error - {$e->getMessage()}\n";
            }
        }

        $this->state = 'ABORTED';
        echo "\n🚫 Transaction বাতিল করা হয়েছে\n";
    }
}

/**
 * Participant Interface
 */
interface TransactionParticipant
{
    public function getName(): string;
    public function prepare(string $txnId, array $operation): bool;
    public function commit(string $txnId): bool;
    public function rollback(string $txnId): void;
}

/**
 * ব্যাংক Participant - সোনালী/জনতা ব্যাংক
 */
class BankParticipant implements TransactionParticipant
{
    private string $name;
    private float $balance;
    private array $preparedOps = [];
    private array $locks = [];

    public function __construct(string $name, float $balance)
    {
        $this->name = $name;
        $this->balance = $balance;
    }

    public function getName(): string
    {
        return $this->name;
    }

    public function prepare(string $txnId, array $operation): bool
    {
        $type = $operation['type'];     // 'debit' বা 'credit'
        $amount = $operation['amount'];
        $account = $operation['account'];

        echo "    🔍 {$this->name}: Checking {$type} {$amount} BDT for {$account}\n";

        // Debit হলে ব্যালেন্স চেক করা
        if ($type === 'debit' && $this->balance < $amount) {
            echo "    ❌ {$this->name}: অপর্যাপ্ত ব্যালেন্স (আছে: {$this->balance}, দরকার: {$amount})\n";
            return false;
        }

        // Lock নেওয়া
        $this->locks[$txnId] = true;
        $this->preparedOps[$txnId] = $operation;

        echo "    🔒 {$this->name}: Account locked, prepared for {$type}\n";
        return true;
    }

    public function commit(string $txnId): bool
    {
        if (!isset($this->preparedOps[$txnId])) {
            return false;
        }

        $operation = $this->preparedOps[$txnId];

        if ($operation['type'] === 'debit') {
            $this->balance -= $operation['amount'];
        } else {
            $this->balance += $operation['amount'];
        }

        // Lock release
        unset($this->locks[$txnId]);
        unset($this->preparedOps[$txnId]);

        echo "    💰 {$this->name}: New balance = {$this->balance} BDT\n";
        return true;
    }

    public function rollback(string $txnId): void
    {
        unset($this->locks[$txnId]);
        unset($this->preparedOps[$txnId]);
        echo "    ↩️ {$this->name}: Lock released, no changes made\n";
    }
}

// ========== ব্যবহার ==========

$coordinator = new TwoPhaseCommitCoordinator();

// ব্যাংক participants তৈরি
$sonaliBank = new BankParticipant('সোনালী ব্যাংক', 200000);
$janataBank = new BankParticipant('জনতা ব্যাংক', 100000);

$coordinator->addParticipant($sonaliBank);
$coordinator->addParticipant($janataBank);

// ৫০,০০০ টাকা ট্রান্সফার
$operations = [
    ['type' => 'debit', 'amount' => 50000, 'account' => 'rahim-001'],
    ['type' => 'credit', 'amount' => 50000, 'account' => 'karim-002'],
];

$result = $coordinator->execute($operations);

echo "\n" . str_repeat('═', 50) . "\n";
echo $result
    ? "✅ রহিম → করিম: ৫০,০০০ টাকা সফলভাবে ট্রান্সফার হয়েছে"
    : "❌ ট্রান্সফার ব্যর্থ - কোনো টাকা কাটা হয়নি";
echo "\n";
```

---

## 🟨 JavaScript কোড উদাহরণ

```javascript
/**
 * Two-Phase Commit with Recovery
 * বাংলাদেশ ব্যাংক RTGS সিমুলেশন
 */

class DistributedTransactionCoordinator {
    constructor(options = {}) {
        this.transactionId = `txn_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
        this.participants = [];
        this.state = 'INIT';
        this.timeout = options.timeout || 30000; // 30 সেকেন্ড
        this.recoveryLog = []; // WAL (Write-Ahead Log)
    }

    addParticipant(participant) {
        this.participants.push(participant);
    }

    /**
     * Write-Ahead Log এ entry লেখা (recovery-র জন্য)
     */
    logDecision(phase, decision, details = {}) {
        const entry = {
            txnId: this.transactionId,
            phase,
            decision,
            timestamp: new Date().toISOString(),
            ...details
        };
        this.recoveryLog.push(entry);
        console.log(`  📝 WAL: ${phase} → ${decision}`);
    }

    /**
     * মূল 2PC execution
     */
    async execute(operations) {
        console.log(`\n🏦 === Inter-Bank Transfer Started ===`);
        console.log(`   Transaction ID: ${this.transactionId}`);
        console.log(`   Timeout: ${this.timeout}ms\n`);

        try {
            // Phase 1: Prepare
            const prepared = await this.preparePhase(operations);

            if (prepared) {
                this.logDecision('GLOBAL', 'COMMIT');
                return await this.commitPhase();
            } else {
                this.logDecision('GLOBAL', 'ABORT');
                await this.abortPhase();
                return false;
            }
        } catch (error) {
            console.error(`💥 Coordinator Error: ${error.message}`);
            this.logDecision('GLOBAL', 'ABORT', { error: error.message });
            await this.abortPhase();
            return false;
        }
    }

    /**
     * Phase 1: Prepare - সব participant-কে ভোট দিতে বলা
     */
    async preparePhase(operations) {
        console.log('📋 ═══ Phase 1: PREPARE ═══');
        this.state = 'PREPARING';

        const votePromises = this.participants.map((participant, index) => {
            const operation = operations[index];
            return this.prepareWithTimeout(participant, operation);
        });

        const votes = await Promise.allSettled(votePromises);

        let allAgreed = true;
        votes.forEach((result, index) => {
            const name = this.participants[index].name;
            if (result.status === 'fulfilled' && result.value === true) {
                console.log(`   ✓ ${name}: VOTE YES`);
                this.logDecision('PREPARE', 'YES', { participant: name });
            } else {
                const reason = result.status === 'rejected' ? result.reason.message : 'VOTE NO';
                console.log(`   ✗ ${name}: ${reason}`);
                this.logDecision('PREPARE', 'NO', { participant: name, reason });
                allAgreed = false;
            }
        });

        return allAgreed;
    }

    /**
     * Timeout সহ prepare call
     */
    prepareWithTimeout(participant, operation) {
        return new Promise((resolve, reject) => {
            const timer = setTimeout(() => {
                reject(new Error(`TIMEOUT: ${participant.name} ${this.timeout}ms এ respond করেনি`));
            }, this.timeout);

            participant.prepare(this.transactionId, operation)
                .then(result => {
                    clearTimeout(timer);
                    resolve(result);
                })
                .catch(err => {
                    clearTimeout(timer);
                    reject(err);
                });
        });
    }

    /**
     * Phase 2: Commit
     */
    async commitPhase() {
        console.log('\n✅ ═══ Phase 2: COMMIT ═══');
        this.state = 'COMMITTING';

        const results = await Promise.allSettled(
            this.participants.map(p => p.commit(this.transactionId))
        );

        let success = true;
        results.forEach((result, index) => {
            const name = this.participants[index].name;
            if (result.status === 'fulfilled' && result.value) {
                console.log(`   ✓ ${name}: COMMITTED`);
            } else {
                console.log(`   ✗ ${name}: COMMIT FAILED`);
                success = false;
            }
        });

        this.state = success ? 'COMMITTED' : 'PARTIALLY_COMMITTED';
        return success;
    }

    /**
     * Abort Phase: সবাইকে rollback করানো
     */
    async abortPhase() {
        console.log('\n🚫 ═══ Phase 2: ABORT ═══');
        this.state = 'ABORTING';

        await Promise.allSettled(
            this.participants.map(p => p.rollback(this.transactionId))
        );

        this.state = 'ABORTED';
        console.log('   ↩️ সব participant rollback করেছে');
    }

    /**
     * Coordinator Crash Recovery
     */
    async recover() {
        console.log('\n🔧 === Recovery শুরু ===');
        const lastDecision = this.recoveryLog[this.recoveryLog.length - 1];

        if (!lastDecision) {
            console.log('   কোনো pending transaction নেই');
            return;
        }

        console.log(`   Last logged: ${lastDecision.phase} → ${lastDecision.decision}`);

        if (lastDecision.decision === 'COMMIT') {
            console.log('   → Commit decision ছিল, সবাইকে আবার commit বলা হচ্ছে...');
            await this.commitPhase();
        } else {
            console.log('   → Abort decision ছিল, সবাইকে rollback বলা হচ্ছে...');
            await this.abortPhase();
        }
    }
}

/**
 * Bank Participant
 */
class BankService {
    constructor(name, initialBalance) {
        this.name = name;
        this.balance = initialBalance;
        this.preparedTransactions = new Map();
        this.committed = new Map();
    }

    async prepare(txnId, operation) {
        // নেটওয়ার্ক latency সিমুলেট
        await this.simulateLatency(100, 500);

        const { type, amount, account } = operation;

        if (type === 'debit' && this.balance < amount) {
            throw new Error(`অপর্যাপ্ত ব্যালেন্স: আছে ${this.balance}, দরকার ${amount}`);
        }

        // Transaction prepare (lock resources)
        this.preparedTransactions.set(txnId, operation);
        return true;
    }

    async commit(txnId) {
        await this.simulateLatency(50, 200);

        const operation = this.preparedTransactions.get(txnId);
        if (!operation) return false;

        if (operation.type === 'debit') {
            this.balance -= operation.amount;
        } else {
            this.balance += operation.amount;
        }

        this.preparedTransactions.delete(txnId);
        this.committed.set(txnId, operation);
        console.log(`   💰 ${this.name}: নতুন ব্যালেন্স = ${this.balance.toLocaleString()} BDT`);
        return true;
    }

    async rollback(txnId) {
        await this.simulateLatency(50, 100);
        this.preparedTransactions.delete(txnId);
    }

    simulateLatency(min, max) {
        const delay = Math.floor(Math.random() * (max - min)) + min;
        return new Promise(resolve => setTimeout(resolve, delay));
    }
}

// ========== ব্যবহার ==========

async function interBankTransfer() {
    const coordinator = new DistributedTransactionCoordinator({ timeout: 5000 });

    // ব্যাংক সার্ভিস তৈরি
    const sonaliBank = new BankService('সোনালী ব্যাংক', 500000);
    const janataBank = new BankService('জনতা ব্যাংক', 300000);

    coordinator.addParticipant(sonaliBank);
    coordinator.addParticipant(janataBank);

    // ৫০,০০০ টাকা ট্রান্সফার
    const operations = [
        { type: 'debit', amount: 50000, account: 'rahim-sonali-001' },
        { type: 'credit', amount: 50000, account: 'karim-janata-002' }
    ];

    const success = await coordinator.execute(operations);

    console.log('\n' + '═'.repeat(50));
    console.log(success
        ? '✅ ট্রান্সফার সফল: ৫০,০০০ BDT (সোনালী → জনতা)'
        : '❌ ট্রান্সফার ব্যর্থ: কোনো পরিবর্তন হয়নি');
}

interBankTransfer();
```

---

## ⚖️ 2PC vs Saga তুলনা

| বৈশিষ্ট্য | 2PC | Saga |
|-----------|-----|------|
| Consistency | Strong (ACID) | Eventual |
| Blocking | হ্যাঁ (participant lock ধরে রাখে) | না |
| Performance | ধীর (lock holding) | দ্রুত |
| Availability | কম (coordinator SPOF) | বেশি |
| Rollback | Automatic (coordinator abort) | Compensating transactions |
| Use Case | ব্যাংক transfer, financial | E-commerce, booking |
| Latency Tolerance | Low latency required | High latency OK |
| Scale | ২-৫ participants | অনেক services |

### কখন 2PC:
- Financial transactions যেখানে strong consistency দরকার
- অল্প সংখ্যক participant (২-৩)
- Low-latency network (same datacenter)

### কখন Saga:
- Long-running business processes
- অনেক microservices জড়িত
- Eventual consistency acceptable
- High availability দরকার

---

## ⚠️ 2PC-র সমস্যাগুলো

### 1. Blocking Problem
```
Coordinator crash হলে:
┌──────────────┐     ┌──────────────┐
│  Participant │     │  Participant │
│   PREPARED   │     │   PREPARED   │
│   (locked)   │     │   (locked)   │
│              │     │              │
│  "কী করব?   │     │  "কী করব?   │
│   Commit?    │     │   Commit?    │
│   Abort?"    │     │   Abort?"    │
│              │     │              │
│  ⏰ waiting...│     │  ⏰ waiting...│
└──────────────┘     └──────────────┘
      Resources locked indefinitely! 😱
```

### 2. Single Point of Failure
- Coordinator crash = পুরো system আটকে যায়

### 3. Performance Impact
- Lock holding time: Prepare → Commit পর্যন্ত সব resource locked
- Network round trips: কমপক্ষে 2N messages (N = participant count)

---

## ✅ কখন ব্যবহার করবেন / করবেন না

### ✅ ব্যবহার করবেন:
- 🏦 Inter-bank fund transfer (সোনালী ↔ জনতা)
- 💳 Critical financial operations যেখানে data loss acceptable না
- 🗄️ Same datacenter-এ distributed databases sync
- 📊 XA transactions (MySQL + PostgreSQL একসাথে)
- 🔄 অল্প সংখ্যক (২-৩) participant

### ❌ ব্যবহার করবেন না:
- 🌐 Cross-datacenter transactions (latency বেশি)
- 📱 Microservices architecture (অনেক services)
- 🛒 E-commerce order processing (Saga ভালো)
- 🔄 Long-running processes (minutes/hours)
- 📈 High-throughput systems (lock bottleneck)
- 🌍 Geo-distributed systems

### 🎯 বাস্তব সিদ্ধান্ত গাইড:

```
আপনার Transaction কোন ধরনের?
│
├── Strong consistency দরকার? (টাকা-পয়সা)
│   ├── হ্যাঁ → Participants কয়টা?
│   │   ├── ২-৩ → 2PC ✓
│   │   └── ৩+ → Saga with careful design
│   └── না → Saga / Eventual consistency
│
├── Cross-datacenter?
│   ├── হ্যাঁ → Saga (2PC কাজ করবে না ভালো)
│   └── না → 2PC consider করতে পারেন
│
└── High availability critical?
    ├── হ্যাঁ → Saga
    └── না → 2PC acceptable
```

---

## 📚 সারসংক্ষেপ

| Protocol | Blocking | Fault Tolerance | Messages | Use Case |
|----------|----------|-----------------|----------|----------|
| 2PC | হ্যাঁ | কম | 2N | Financial, DB sync |
| 3PC | না | মাঝারি | 3N | Rare (network partition issue) |
| Saga | না | বেশি | Variable | Microservices, e-commerce |

**মনে রাখুন**: 2PC হলো distributed systems-এর সবচেয়ে strict consistency guarantee, কিন্তু এর মূল্য হলো availability এবং performance। CAP theorem অনুযায়ী — Consistency এবং Availability একসাথে পাওয়া সম্ভব না distributed system-এ।
