# 🗳️ ডিস্ট্রিবিউটেড কনসেনসাস (Distributed Consensus)

## ধারণা

ডিস্ট্রিবিউটেড কনসেনসাস হলো একাধিক নোড বা সার্ভারের মধ্যে একটি মান বা সিদ্ধান্তের ব্যাপারে একমত হওয়ার প্রক্রিয়া। নেটওয়ার্ক বিচ্ছিন্নতা, নোড ফেইলিওর, বা মেসেজ হারানোর মতো সমস্যা থাকা সত্ত্বেও সবাই একই সিদ্ধান্তে আসতে হবে।

### কেন প্রয়োজন?

ডিস্ট্রিবিউটেড সিস্টেমে:
- কোন নোড **লিডার** হবে তা নির্ধারণ করতে
- সব নোডে **একই ডেটা** থাকা নিশ্চিত করতে
- **কনফিগারেশন** পরিবর্তন সবার কাছে পৌঁছানোর জন্য
- **ডিস্ট্রিবিউটেড লক** বাস্তবায়নে

---

## 🏠 বাস্তব জীবনের উদাহরণ

### গ্রামের সালিশ সভা

একটি গ্রামে ১০ জন মুরুব্বী একটি সিদ্ধান্ত নিতে চান। সবাই একটি রুমে নেই — কেউ ফোনে, কেউ চিঠিতে যোগাযোগ করছে। সিদ্ধান্ত নিতে হলে:
- একজন **প্রস্তাব** দেবে (Propose)
- সংখ্যাগরিষ্ঠ (৬ জন) রাজি হলে সিদ্ধান্ত **গৃহীত** হবে
- কেউ যোগাযোগ বিচ্ছিন্ন হলেও সিদ্ধান্ত কার্যকর থাকবে

এটাই মূলত কনসেনসাস অ্যালগরিদমের কাজ।

---

## 📜 Paxos অ্যালগরিদম

### মূল ভূমিকা
- **Proposer**: প্রস্তাব দেয়
- **Acceptor**: প্রস্তাব গ্রহণ বা প্রত্যাখ্যান করে
- **Learner**: চূড়ান্ত সিদ্ধান্ত জানতে পারে

### দুই ধাপের প্রক্রিয়া

**ধাপ ১ — Prepare:**
- Proposer একটি ইউনিক নম্বর (n) দিয়ে Prepare রিকোয়েস্ট পাঠায়
- Acceptor যদি এর চেয়ে বড় নম্বর না দেখে থাকে, তাহলে Promise করে

**ধাপ ২ — Accept:**
- সংখ্যাগরিষ্ঠ Promise পেলে, Proposer Accept রিকোয়েস্ট পাঠায়
- Acceptor গ্রহণ করলে মান নির্ধারিত হয়

---

## 🚀 Raft অ্যালগরিদম

Raft হলো Paxos-এর তুলনায় সহজবোধ্য কনসেনসাস অ্যালগরিদম। etcd, Consul এবং CockroachDB তে ব্যবহৃত হয়।

### তিনটি ভূমিকা
- **Leader**: সব Write অপারেশন পরিচালনা করে
- **Follower**: Leader-এর নির্দেশ মেনে চলে
- **Candidate**: Leader নির্বাচনে প্রার্থী

### Leader Election প্রক্রিয়া

1. সব নোড শুরুতে **Follower**
2. নির্দিষ্ট সময়ে Leader-এর কাছ থেকে হার্টবিট না পেলে একটি Follower **Candidate** হয়
3. Candidate নিজেকে ভোট দেয় এবং অন্যদের কাছে ভোট চায়
4. সংখ্যাগরিষ্ঠ ভোট পেলে **Leader** হয়
5. Leader নিয়মিত হার্টবিট পাঠায়

---

## 💻 PHP কোড উদাহরণ

### সরলীকৃত Raft Leader Election

```php
<?php

class RaftNode
{
    public string $id;
    public string $state = 'follower'; // follower, candidate, leader
    public int $currentTerm = 0;
    public ?string $votedFor = null;
    public int $voteCount = 0;

    public function __construct(string $id)
    {
        $this->id = $id;
    }

    public function startElection(array $cluster): void
    {
        $this->state = 'candidate';
        $this->currentTerm++;
        $this->votedFor = $this->id;
        $this->voteCount = 1; // নিজেকে ভোট

        echo "🗳️ নোড {$this->id} নির্বাচন শুরু করেছে (টার্ম: {$this->currentTerm})\n";

        foreach ($cluster as $node) {
            if ($node->id !== $this->id) {
                $granted = $node->requestVote($this->id, $this->currentTerm);
                if ($granted) {
                    $this->voteCount++;
                    echo "  ✅ নোড {$node->id} ভোট দিয়েছে\n";
                } else {
                    echo "  ❌ নোড {$node->id} ভোট দেয়নি\n";
                }
            }
        }

        $majority = (int) ceil(count($cluster) / 2);
        if ($this->voteCount >= $majority) {
            $this->state = 'leader';
            echo "👑 নোড {$this->id} লিডার নির্বাচিত হয়েছে! ({$this->voteCount}/{$majority} ভোট)\n";
        } else {
            $this->state = 'follower';
            echo "😞 নোড {$this->id} পর্যাপ্ত ভোট পায়নি\n";
        }
    }

    public function requestVote(string $candidateId, int $term): bool
    {
        if ($term > $this->currentTerm && $this->votedFor === null) {
            $this->currentTerm = $term;
            $this->votedFor = $candidateId;
            return true;
        }
        return false;
    }

    public function appendEntries(string $leaderId, int $term, string $data): bool
    {
        if ($term >= $this->currentTerm) {
            $this->currentTerm = $term;
            $this->state = 'follower';
            echo "📝 নোড {$this->id}: ডেটা '{$data}' গ্রহণ করেছে (লিডার: {$leaderId})\n";
            return true;
        }
        return false;
    }
}

// ক্লাস্টার তৈরি ও নির্বাচন
$nodes = [
    new RaftNode('N1'),
    new RaftNode('N2'),
    new RaftNode('N3'),
    new RaftNode('N4'),
    new RaftNode('N5'),
];

$nodes[0]->startElection($nodes);
```

---

## 💻 JavaScript কোড উদাহরণ

### সরলীকৃত কনসেনসাস সিস্টেম

```javascript
class ConsensusNode {
    constructor(id) {
        this.id = id;
        this.state = 'follower';
        this.term = 0;
        this.votedFor = null;
        this.log = [];
    }

    requestVote(candidateId, term) {
        if (term > this.term && !this.votedFor) {
            this.term = term;
            this.votedFor = candidateId;
            return { granted: true, term: this.term };
        }
        return { granted: false, term: this.term };
    }

    appendEntry(leaderId, term, entry) {
        if (term >= this.term) {
            this.term = term;
            this.state = 'follower';
            this.log.push(entry);
            return true;
        }
        return false;
    }
}

class RaftCluster {
    constructor(nodeCount) {
        this.nodes = Array.from({ length: nodeCount }, (_, i) =>
            new ConsensusNode(`N${i + 1}`)
        );
        this.leader = null;
    }

    electLeader(candidateIndex) {
        const candidate = this.nodes[candidateIndex];
        candidate.state = 'candidate';
        candidate.term++;
        candidate.votedFor = candidate.id;
        let votes = 1;

        console.log(`🗳️ ${candidate.id} নির্বাচন শুরু করেছে (টার্ম ${candidate.term})`);

        for (const node of this.nodes) {
            if (node.id === candidate.id) continue;
            const response = node.requestVote(candidate.id, candidate.term);
            if (response.granted) {
                votes++;
                console.log(`  ✅ ${node.id} → ${candidate.id} কে ভোট দিয়েছে`);
            }
        }

        const majority = Math.ceil(this.nodes.length / 2);
        if (votes >= majority) {
            candidate.state = 'leader';
            this.leader = candidate;
            console.log(`👑 ${candidate.id} লিডার! (${votes}/${this.nodes.length} ভোট)`);
        }
    }

    replicate(data) {
        if (!this.leader) {
            console.log('❌ কোনো লিডার নেই!');
            return false;
        }

        console.log(`\n📤 লিডার ${this.leader.id} ডেটা রেপ্লিকেট করছে: "${data}"`);
        this.leader.log.push(data);
        let acks = 1;

        for (const node of this.nodes) {
            if (node.id === this.leader.id) continue;
            const success = node.appendEntry(this.leader.id, this.leader.term, data);
            if (success) {
                acks++;
                console.log(`  ✅ ${node.id} ডেটা গ্রহণ করেছে`);
            }
        }

        const majority = Math.ceil(this.nodes.length / 2);
        const committed = acks >= majority;
        console.log(committed
            ? `✅ কমিটেড (${acks}/${this.nodes.length} ACK)`
            : `❌ কমিট ব্যর্থ`);
        return committed;
    }
}

// ব্যবহার
const cluster = new RaftCluster(5);
cluster.electLeader(0);
cluster.replicate('SET user:name "রহিম"');
cluster.replicate('SET user:age 30');
```

---

## ⚖️ Paxos vs Raft তুলনা

| বৈশিষ্ট্য | Paxos | Raft |
|-----------|-------|------|
| জটিলতা | অনেক জটিল | তুলনামূলক সহজ |
| বোধগম্যতা | কঠিন | ডিজাইন করাই হয়েছে বোঝার জন্য |
| Leader | আলাদা Leader ধারণা নেই | স্পষ্ট Leader আছে |
| ব্যবহার | Google Chubby | etcd, Consul, TiKV |
| Log Replication | জটিল | সরল ও স্পষ্ট |

---

## 🛠️ বাস্তব ব্যবহার

| টুল/সিস্টেম | অ্যালগরিদম | কাজ |
|-------------|-----------|------|
| etcd | Raft | Kubernetes কনফিগারেশন |
| Consul | Raft | Service Discovery |
| ZooKeeper | ZAB (Paxos-ভিত্তিক) | ডিস্ট্রিবিউটেড কোঅর্ডিনেশন |
| CockroachDB | Raft | ডিস্ট্রিবিউটেড SQL |
| Google Spanner | Paxos | গ্লোবাল ডাটাবেজ |

---

## ✅ কখন ব্যবহার করবেন

- ডিস্ট্রিবিউটেড ডাটাবেজে ডেটা কনসিস্টেন্সি নিশ্চিত করতে
- সার্ভিস ডিসকভারি ও কনফিগারেশন ম্যানেজমেন্টে
- ডিস্ট্রিবিউটেড লক ম্যানেজমেন্টে
- Leader Election প্রয়োজন হলে (যেমন: কোন সার্ভার Master হবে)

## ❌ কখন ব্যবহার করবেন না

- সিঙ্গেল সার্ভার অ্যাপ্লিকেশনে (অপ্রয়োজনীয় জটিলতা)
- যখন Eventual Consistency গ্রহণযোগ্য (যেমন: সোশ্যাল মিডিয়া লাইক কাউন্ট)
- ছোট সিস্টেমে যেখানে একটি মাস্টার-স্লেভ সেটআপ যথেষ্ট
- হাই-থ্রুপুট রাইটে যেখানে কনসেনসাস ল্যাটেন্সি গ্রহণযোগ্য নয়
