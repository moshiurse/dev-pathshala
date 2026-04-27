# 🤝 Mediator প্যাটার্ন

## 📌 সংজ্ঞা

> **GoF Definition:** *"Define an object that encapsulates how a set of objects interact. Mediator promotes loose coupling by keeping objects from referring to each other explicitly, and it lets you vary their interaction independently."*

Mediator প্যাটার্ন হলো একটি **Behavioral Design Pattern** যেটি একগুচ্ছ অবজেক্টের মধ্যে communication-এর জটিলতা কমিয়ে আনে। সাধারণত অবজেক্টগুলো একে অপরের সাথে সরাসরি কথা বলে — এতে **tight coupling** তৈরি হয়, dependency বাড়ে, এবং সিস্টেম maintain করা কঠিন হয়ে যায়। Mediator একটি **central hub** হিসেবে কাজ করে যেখানে সব communication route হয়।

### মূল ধারণা

কল্পনা করুন একটি অফিসে ১০ জন কর্মী আছে। প্রত্যেকে প্রত্যেকের সাথে সরাসরি কথা বললে `n(n-1)/2` = ৪৫টি communication channel লাগবে। কিন্তু একজন **ম্যানেজার** (Mediator) থাকলে মাত্র ১০টি channel-ই যথেষ্ট। এটাই Mediator-এর শক্তি — **many-to-many** relationship কে **many-to-one** তে রূপান্তর করা।

### গাণিতিক ব্যাখ্যা

```
সরাসরি Communication:  O(n²) connections
Mediator ব্যবহারে:      O(n) connections

n = 5 হলে:  সরাসরি = 10, Mediator = 5
n = 10 হলে: সরাসরি = 45, Mediator = 10
n = 50 হলে: সরাসরি = 1225, Mediator = 50
```

### তিনটি মূল উপাদান

| উপাদান | ভূমিকা |
|---------|--------|
| **Mediator (Interface)** | Communication-এর contract define করে |
| **Concrete Mediator** | প্রকৃত coordination logic ধারণ করে |
| **Colleague** | যে অবজেক্টগুলো Mediator-এর মাধ্যমে communicate করে |

---

## 🏠 বাস্তব উদাহরণ

### ✈️ Air Traffic Controller (ATC)

এটি Mediator প্যাটার্নের সবচেয়ে ক্লাসিক উদাহরণ। বিমান একে অপরের সাথে সরাসরি কথা বলে না — সব communication **ATC** এর মাধ্যমে হয়:

```
বিমান-১ ──┐                    ┌── বিমান-১
বিমান-২ ──┤   সরাসরি কথা      │── বিমান-২
বিমান-৩ ──┤   (বিপজ্জনক!)    │── বিমান-৩
বিমান-৪ ──┘                    └── বিমান-৪
    ↕↕↕↕↕↕ (সবাই সবার সাথে)

          vs

বিমান-১ ──┐
বিমান-২ ──┼── ATC (Mediator) ──→ নিরাপদ coordination
বিমান-৩ ──┤
বিমান-৪ ──┘
```

ATC জানে কোন runway খালি, কোন বিমান কোথায়, কখন কাকে landing permission দিতে হবে। কোনো pilot অন্য pilot-কে বলে না "তুমি সরো, আমি নামছি"।

### 🇧🇩 বাংলাদেশের বাস্তব প্রসঙ্গ

**রাইড-শেয়ারিং ডিসপ্যাচ সিস্টেম (Pathao/Uber):**
- যাত্রী এবং ড্রাইভার একে অপরকে সরাসরি খোঁজে না
- **Dispatch System** (Mediator) সবকিছু coordinate করে
- যাত্রী ride request দেয় → Mediator → কাছের ড্রাইভারকে assign করে
- ড্রাইভার accept করে → Mediator → যাত্রীকে জানায়

**Daraz/Bikroy.com Buyer-Seller Chat:**
- ক্রেতা ও বিক্রেতা সরাসরি contact করে না
- **Platform Chat System** (Mediator) মাধ্যমে কথা হয়
- Mediator spam filter, message logging, dispute resolution সব handle করে

---

## 📊 UML ডায়াগ্রাম

```
    ┌─────────────────────────┐
    │    <<interface>>         │
    │      Mediator            │
    ├─────────────────────────┤
    │ + notify(sender, event) │
    └────────────┬────────────┘
                 │
                 │ implements
                 ▼
    ┌─────────────────────────────────┐
    │       ConcreteMediator          │
    ├─────────────────────────────────┤
    │ - colleagueA: ColleagueA        │
    │ - colleagueB: ColleagueB        │
    ├─────────────────────────────────┤
    │ + notify(sender, event): void   │
    │ + registerColleague(c): void    │
    └──────┬──────────────┬───────────┘
           │              │
    has-a  │              │  has-a
           ▼              ▼
  ┌──────────────┐  ┌──────────────┐
  │  ColleagueA  │  │  ColleagueB  │
  ├──────────────┤  ├──────────────┤
  │ - mediator   │  │ - mediator   │
  ├──────────────┤  ├──────────────┤
  │ + operationA │  │ + operationB │
  │ + changed()  │  │ + changed()  │
  └──────────────┘  └──────────────┘

  Colleague গুলো শুধু Mediator কে চেনে,
  একে অপরকে চেনে না।
```

### Sequence ডায়াগ্রাম

```
  ColleagueA          Mediator          ColleagueB
      │                  │                   │
      │  notify("eventX")│                   │
      │─────────────────>│                   │
      │                  │  react("eventX")  │
      │                  │──────────────────>│
      │                  │                   │
      │                  │   acknowledge()   │
      │                  │<──────────────────│
      │  update()        │                   │
      │<─────────────────│                   │
      │                  │                   │
```

---

## 💻 ইমপ্লিমেন্টেশন

### ১. Basic Mediator

#### PHP 8.3

```php
<?php

declare(strict_types=1);

// Mediator Interface
interface Mediator
{
    public function notify(object $sender, string $event): void;
}

// Base Colleague
abstract class Colleague
{
    public function __construct(
        protected ?Mediator $mediator = null
    ) {}

    public function setMediator(Mediator $mediator): void
    {
        $this->mediator = $mediator;
    }
}

// Concrete Colleagues
class ComponentA extends Colleague
{
    public function doActionA(): string
    {
        $result = "ComponentA: Action A সম্পন্ন হয়েছে।";
        $this->mediator?->notify($this, 'actionA');
        return $result;
    }

    public function doActionZ(): string
    {
        return "ComponentA: Action Z সম্পন্ন হয়েছে।";
    }
}

class ComponentB extends Colleague
{
    public function doActionB(): string
    {
        $result = "ComponentB: Action B সম্পন্ন হয়েছে।";
        $this->mediator?->notify($this, 'actionB');
        return $result;
    }

    public function doActionY(): string
    {
        return "ComponentB: Action Y সম্পন্ন হয়েছে।";
    }
}

// Concrete Mediator — সব coordination logic এখানে
class ConcreteMediator implements Mediator
{
    public function __construct(
        private readonly ComponentA $componentA,
        private readonly ComponentB $componentB,
    ) {
        $this->componentA->setMediator($this);
        $this->componentB->setMediator($this);
    }

    public function notify(object $sender, string $event): void
    {
        match ($event) {
            'actionA' => $this->componentB->doActionY(),
            'actionB' => $this->componentA->doActionZ(),
            default   => null,
        };
    }
}

// ব্যবহার
$compA = new ComponentA();
$compB = new ComponentB();
$mediator = new ConcreteMediator($compA, $compB);

echo $compA->doActionA(); // এটি trigger করবে componentB->doActionY()
echo $compB->doActionB(); // এটি trigger করবে componentA->doActionZ()
```

#### JavaScript (ES2022+)

```javascript
// Mediator Interface (duck typing - JS তে explicit interface নেই)
class Colleague {
    #mediator = null;

    setMediator(mediator) {
        this.#mediator = mediator;
    }

    get mediator() {
        return this.#mediator;
    }
}

class ComponentA extends Colleague {
    doActionA() {
        const result = "ComponentA: Action A সম্পন্ন হয়েছে।";
        this.mediator?.notify(this, "actionA");
        return result;
    }

    doActionZ() {
        return "ComponentA: Action Z সম্পন্ন হয়েছে।";
    }
}

class ComponentB extends Colleague {
    doActionB() {
        const result = "ComponentB: Action B সম্পন্ন হয়েছে।";
        this.mediator?.notify(this, "actionB");
        return result;
    }

    doActionY() {
        return "ComponentB: Action Y সম্পন্ন হয়েছে।";
    }
}

class ConcreteMediator {
    #componentA;
    #componentB;

    constructor(componentA, componentB) {
        this.#componentA = componentA;
        this.#componentB = componentB;
        this.#componentA.setMediator(this);
        this.#componentB.setMediator(this);
    }

    notify(sender, event) {
        switch (event) {
            case "actionA":
                this.#componentB.doActionY();
                break;
            case "actionB":
                this.#componentA.doActionZ();
                break;
        }
    }
}

// ব্যবহার
const compA = new ComponentA();
const compB = new ComponentB();
const mediator = new ConcreteMediator(compA, compB);

console.log(compA.doActionA());
console.log(compB.doActionB());
```

---

### ২. Chat Room Mediator

#### PHP 8.3 — চ্যাট রুম সিস্টেম

```php
<?php

declare(strict_types=1);

interface ChatMediator
{
    public function sendMessage(string $message, User $sender, ?string $targetName = null): void;
    public function addUser(User $user): void;
    public function getOnlineUsers(): array;
}

class User
{
    private array $inbox = [];

    public function __construct(
        private readonly string $name,
        private readonly ChatMediator $chatRoom,
    ) {
        $this->chatRoom->addUser($this);
    }

    public function getName(): string
    {
        return $this->name;
    }

    // সবাইকে মেসেজ পাঠাও
    public function sendBroadcast(string $message): void
    {
        echo "📤 {$this->name} সবাইকে বলছে: \"{$message}\"\n";
        $this->chatRoom->sendMessage($message, $this);
    }

    // নির্দিষ্ট ব্যক্তিকে মেসেজ পাঠাও
    public function sendDirectMessage(string $message, string $targetName): void
    {
        echo "📤 {$this->name} → {$targetName}: \"{$message}\"\n";
        $this->chatRoom->sendMessage($message, $this, $targetName);
    }

    // মেসেজ receive করো
    public function receive(string $message, string $fromUser): void
    {
        $entry = "[{$fromUser}]: {$message}";
        $this->inbox[] = $entry;
        echo "📥 {$this->name} পেয়েছে {$entry}\n";
    }

    public function getInbox(): array
    {
        return $this->inbox;
    }
}

class ChatRoom implements ChatMediator
{
    /** @var array<string, User> */
    private array $users = [];
    private array $messageLog = [];

    public function addUser(User $user): void
    {
        $this->users[$user->getName()] = $user;
        echo "✅ {$user->getName()} চ্যাট রুমে যোগ দিয়েছে।\n";
    }

    public function sendMessage(string $message, User $sender, ?string $targetName = null): void
    {
        $this->messageLog[] = [
            'from'    => $sender->getName(),
            'to'      => $targetName ?? 'ALL',
            'message' => $message,
            'time'    => new \DateTimeImmutable(),
        ];

        if ($targetName !== null) {
            // Direct message — শুধু target কে পাঠাও
            $target = $this->users[$targetName] ?? null;
            $target?->receive($message, $sender->getName());
            return;
        }

        // Broadcast — sender ছাড়া সবাইকে পাঠাও
        foreach ($this->users as $user) {
            if ($user !== $sender) {
                $user->receive($message, $sender->getName());
            }
        }
    }

    public function getOnlineUsers(): array
    {
        return array_keys($this->users);
    }

    public function getMessageLog(): array
    {
        return $this->messageLog;
    }
}

// ব্যবহার — Daraz-স্টাইল Buyer-Seller চ্যাট
$chatRoom = new ChatRoom();

$rahim  = new User('রহিম (বিক্রেতা)', $chatRoom);
$karim  = new User('করিম (ক্রেতা)', $chatRoom);
$admin  = new User('অ্যাডমিন', $chatRoom);

$karim->sendDirectMessage('এই ফোনের দাম কমবে?', 'রহিম (বিক্রেতা)');
$rahim->sendDirectMessage('ভাই, ৫০০ টাকা কমিয়ে দিচ্ছি।', 'করিম (ক্রেতা)');
$admin->sendBroadcast('আজ রাত ১২টায় সার্ভার maintenance হবে।');
```

#### JavaScript (ES2022+) — চ্যাট রুম সিস্টেম

```javascript
class ChatRoom {
    #users = new Map();
    #messageLog = [];

    addUser(user) {
        this.#users.set(user.name, user);
        user.chatRoom = this;
        console.log(`✅ ${user.name} চ্যাট রুমে যোগ দিয়েছে।`);
    }

    sendMessage(message, sender, targetName = null) {
        this.#messageLog.push({
            from: sender.name,
            to: targetName ?? "ALL",
            message,
            time: new Date(),
        });

        if (targetName) {
            this.#users.get(targetName)?.receive(message, sender.name);
            return;
        }

        for (const [name, user] of this.#users) {
            if (user !== sender) {
                user.receive(message, sender.name);
            }
        }
    }

    get onlineUsers() {
        return [...this.#users.keys()];
    }

    get messageLog() {
        return structuredClone(this.#messageLog);
    }
}

class ChatUser {
    #inbox = [];
    chatRoom = null;

    constructor(name) {
        this.name = name;
    }

    sendBroadcast(message) {
        console.log(`📤 ${this.name} সবাইকে বলছে: "${message}"`);
        this.chatRoom?.sendMessage(message, this);
    }

    sendDM(message, targetName) {
        console.log(`📤 ${this.name} → ${targetName}: "${message}"`);
        this.chatRoom?.sendMessage(message, this, targetName);
    }

    receive(message, fromUser) {
        const entry = `[${fromUser}]: ${message}`;
        this.#inbox.push(entry);
        console.log(`📥 ${this.name} পেয়েছে ${entry}`);
    }

    get inbox() {
        return [...this.#inbox];
    }
}

// ব্যবহার
const room = new ChatRoom();
const seller = new ChatUser("রহিম (বিক্রেতা)");
const buyer = new ChatUser("করিম (ক্রেতা)");
const admin = new ChatUser("অ্যাডমিন");

room.addUser(seller);
room.addUser(buyer);
room.addUser(admin);

buyer.sendDM("এই ফোনের দাম কমবে?", "রহিম (বিক্রেতা)");
seller.sendDM("ভাই, ৫০০ টাকা কমিয়ে দিচ্ছি।", "করিম (ক্রেতা)");
admin.sendBroadcast("আজ রাত ১২টায় সার্ভার maintenance হবে।");
```

---

### ৩. UI Dialog Mediator — ফর্ম কম্পোনেন্ট

#### PHP 8.3

```php
<?php

declare(strict_types=1);

// UI Component গুলো যখন একে অপরকে প্রভাবিত করে, Mediator সেই logic centralize করে

interface DialogMediator
{
    public function componentChanged(UIComponent $component): void;
}

abstract class UIComponent
{
    protected DialogMediator $dialog;
    protected string $name;

    public function __construct(string $name, DialogMediator $dialog)
    {
        $this->name = $name;
        $this->dialog = $dialog;
    }

    public function getName(): string
    {
        return $this->name;
    }

    // Component state পরিবর্তন হলে Mediator কে জানাও
    protected function changed(): void
    {
        $this->dialog->componentChanged($this);
    }
}

class Dropdown extends UIComponent
{
    private string $selected = '';

    public function select(string $value): void
    {
        $this->selected = $value;
        echo "🔽 [{$this->name}] নির্বাচিত: {$value}\n";
        $this->changed();
    }

    public function getSelected(): string
    {
        return $this->selected;
    }
}

class TextInput extends UIComponent
{
    private string $value = '';
    private bool $enabled = true;

    public function setValue(string $value): void
    {
        $this->value = $value;
        echo "📝 [{$this->name}] মান: {$value}\n";
    }

    public function enable(): void
    {
        $this->enabled = true;
        echo "✅ [{$this->name}] সক্রিয় করা হয়েছে\n";
    }

    public function disable(): void
    {
        $this->enabled = false;
        echo "❌ [{$this->name}] নিষ্ক্রিয় করা হয়েছে\n";
    }

    public function isEnabled(): bool
    {
        return $this->enabled;
    }
}

class Checkbox extends UIComponent
{
    private bool $checked = false;

    public function toggle(): void
    {
        $this->checked = !$this->checked;
        $state = $this->checked ? 'চেক করা হয়েছে' : 'আনচেক করা হয়েছে';
        echo "☑️ [{$this->name}] {$state}\n";
        $this->changed();
    }

    public function isChecked(): bool
    {
        return $this->checked;
    }
}

class Button extends UIComponent
{
    private bool $enabled = true;

    public function click(): void
    {
        if (!$this->enabled) {
            echo "🚫 [{$this->name}] বাটন নিষ্ক্রিয়, ক্লিক করা যাবে না\n";
            return;
        }
        echo "🖱️ [{$this->name}] ক্লিক হয়েছে\n";
        $this->changed();
    }

    public function enable(): void
    {
        $this->enabled = true;
        echo "✅ [{$this->name}] সক্রিয়\n";
    }

    public function disable(): void
    {
        $this->enabled = false;
        echo "❌ [{$this->name}] নিষ্ক্রিয়\n";
    }
}

// Registration Form Dialog — Mediator হিসেবে কাজ করে
class RegistrationDialog implements DialogMediator
{
    public readonly Dropdown $districtDropdown;
    public readonly TextInput $cityInput;
    public readonly Checkbox $termsCheckbox;
    public readonly Button $submitButton;

    public function __construct()
    {
        $this->districtDropdown = new Dropdown('জেলা', $this);
        $this->cityInput = new TextInput('শহর', $this);
        $this->termsCheckbox = new Checkbox('শর্তাবলী', $this);
        $this->submitButton = new Button('সাবমিট', $this);
        $this->submitButton->disable();
    }

    public function componentChanged(UIComponent $component): void
    {
        // জেলা নির্বাচন হলে শহরের ইনপুট সক্রিয় করো
        if ($component === $this->districtDropdown) {
            $district = $this->districtDropdown->getSelected();
            match ($district) {
                'ঢাকা'    => $this->cityInput->setValue('ঢাকা সিটি'),
                'চট্টগ্রাম' => $this->cityInput->setValue('চট্টগ্রাম সিটি'),
                default     => $this->cityInput->enable(),
            };
        }

        // শর্তাবলী চেক হলে সাবমিট বাটন সক্রিয় করো
        if ($component === $this->termsCheckbox) {
            if ($this->termsCheckbox->isChecked()) {
                $this->submitButton->enable();
            } else {
                $this->submitButton->disable();
            }
        }

        // সাবমিট বাটন ক্লিক হলে ফর্ম প্রসেস করো
        if ($component === $this->submitButton) {
            echo "🎉 ফর্ম সাবমিট হয়েছে! জেলা: {$this->districtDropdown->getSelected()}\n";
        }
    }
}

// ব্যবহার
$dialog = new RegistrationDialog();
$dialog->districtDropdown->select('ঢাকা');
$dialog->termsCheckbox->toggle();
$dialog->submitButton->click();
```

#### JavaScript (ES2022+)

```javascript
class UIComponent {
    #dialog;
    name;

    constructor(name, dialog) {
        this.name = name;
        this.#dialog = dialog;
    }

    changed() {
        this.#dialog.componentChanged(this);
    }
}

class Dropdown extends UIComponent {
    #selected = "";

    select(value) {
        this.#selected = value;
        console.log(`🔽 [${this.name}] নির্বাচিত: ${value}`);
        this.changed();
    }

    get selected() { return this.#selected; }
}

class TextInput extends UIComponent {
    #value = "";
    #enabled = true;

    setValue(val) {
        this.#value = val;
        console.log(`📝 [${this.name}] মান: ${val}`);
    }

    enable() { this.#enabled = true; console.log(`✅ [${this.name}] সক্রিয়`); }
    disable() { this.#enabled = false; console.log(`❌ [${this.name}] নিষ্ক্রিয়`); }
    get enabled() { return this.#enabled; }
}

class Checkbox extends UIComponent {
    #checked = false;

    toggle() {
        this.#checked = !this.#checked;
        console.log(`☑️ [${this.name}] ${this.#checked ? "চেক" : "আনচেক"}`);
        this.changed();
    }

    get checked() { return this.#checked; }
}

class RegistrationDialog {
    districtDropdown;
    cityInput;
    termsCheckbox;
    submitEnabled = false;

    constructor() {
        this.districtDropdown = new Dropdown("জেলা", this);
        this.cityInput = new TextInput("শহর", this);
        this.termsCheckbox = new Checkbox("শর্তাবলী", this);
    }

    componentChanged(component) {
        if (component === this.districtDropdown) {
            const districts = { "ঢাকা": "ঢাকা সিটি", "চট্টগ্রাম": "চট্টগ্রাম সিটি" };
            const city = districts[component.selected];
            if (city) this.cityInput.setValue(city);
            else this.cityInput.enable();
        }

        if (component === this.termsCheckbox) {
            this.submitEnabled = component.checked;
            console.log(`সাবমিট বাটন: ${this.submitEnabled ? "✅ সক্রিয়" : "❌ নিষ্ক্রিয়"}`);
        }
    }

    submit() {
        if (!this.submitEnabled) {
            console.log("🚫 আগে শর্তাবলী মেনে নিন!");
            return;
        }
        console.log(`🎉 ফর্ম সাবমিট! জেলা: ${this.districtDropdown.selected}`);
    }
}

// ব্যবহার
const dialog = new RegistrationDialog();
dialog.districtDropdown.select("ঢাকা");
dialog.termsCheckbox.toggle();
dialog.submit();
```

---

### ৪. Event Mediator / Event Dispatcher

Event-based Mediator হলো Mediator প্যাটার্নের সবচেয়ে শক্তিশালী এবং flexible রূপ। এটি **Pub/Sub** মডেলের সাথে মিশে যায় এবং Laravel, Express.js সহ প্রায় সব আধুনিক ফ্রেমওয়ার্কে ব্যবহৃত হয়।

#### PHP 8.3 — Event Dispatcher

```php
<?php

declare(strict_types=1);

class EventMediator
{
    /** @var array<string, array<int, callable>> */
    private array $listeners = [];
    private static ?self $instance = null;

    // Singleton — সাধারণত Event Dispatcher একটাই থাকে
    public static function getInstance(): self
    {
        return self::$instance ??= new self();
    }

    public function on(string $event, callable $listener, int $priority = 0): self
    {
        $this->listeners[$event][] = [
            'callback' => $listener,
            'priority' => $priority,
        ];

        // Priority অনুযায়ী sort — উচ্চ priority আগে execute হবে
        usort(
            $this->listeners[$event],
            fn(array $a, array $b) => $b['priority'] <=> $a['priority']
        );

        return $this;
    }

    public function emit(string $event, mixed ...$args): void
    {
        if (!isset($this->listeners[$event])) {
            return;
        }

        foreach ($this->listeners[$event] as $listener) {
            $result = ($listener['callback'])(...$args);

            // listener false return করলে propagation বন্ধ হবে
            if ($result === false) {
                break;
            }
        }
    }

    public function off(string $event, ?callable $listener = null): self
    {
        if ($listener === null) {
            unset($this->listeners[$event]);
        } else {
            $this->listeners[$event] = array_filter(
                $this->listeners[$event] ?? [],
                fn(array $l) => $l['callback'] !== $listener
            );
        }

        return $this;
    }

    public function once(string $event, callable $listener): self
    {
        $wrapper = null;
        $wrapper = function () use ($event, $listener, &$wrapper) {
            $this->off($event, $wrapper);
            return $listener(...func_get_args());
        };

        return $this->on($event, $wrapper);
    }
}

// ব্যবহার — E-Commerce Order System
$mediator = EventMediator::getInstance();

// বিভিন্ন সার্ভিস listener হিসেবে register করছে
$mediator->on('order.placed', function (array $order) {
    echo "📦 Inventory: {$order['product']} এর স্টক কমানো হচ্ছে...\n";
}, priority: 10);

$mediator->on('order.placed', function (array $order) {
    echo "💳 Payment: ৳{$order['amount']} চার্জ করা হচ্ছে...\n";
}, priority: 20); // এটি আগে execute হবে

$mediator->on('order.placed', function (array $order) {
    echo "📧 Notification: {$order['customer']} কে ইমেইল পাঠানো হচ্ছে...\n";
}, priority: 5);

$mediator->once('order.placed', function (array $order) {
    echo "🎁 Welcome Bonus: প্রথম অর্ডারের বোনাস দেওয়া হচ্ছে!\n";
});

// Order place হলে — সব listener automatic ট্রিগার হবে
$mediator->emit('order.placed', [
    'product'  => 'Samsung Galaxy A54',
    'amount'   => 32000,
    'customer' => 'আব্দুল করিম',
]);
```

#### JavaScript (ES2022+) — Event Dispatcher

```javascript
class EventMediator {
    #listeners = new Map();

    on(event, callback, { priority = 0, once = false } = {}) {
        if (!this.#listeners.has(event)) {
            this.#listeners.set(event, []);
        }

        const wrapper = once
            ? (...args) => {
                  this.off(event, wrapper);
                  return callback(...args);
              }
            : callback;

        const list = this.#listeners.get(event);
        list.push({ callback: wrapper, priority });
        list.sort((a, b) => b.priority - a.priority);

        return this;
    }

    once(event, callback, priority = 0) {
        return this.on(event, callback, { priority, once: true });
    }

    emit(event, ...args) {
        const listeners = this.#listeners.get(event) ?? [];

        for (const { callback } of listeners) {
            const result = callback(...args);
            if (result === false) break; // propagation বন্ধ
        }

        return this;
    }

    off(event, callback = null) {
        if (!callback) {
            this.#listeners.delete(event);
        } else {
            const list = this.#listeners.get(event) ?? [];
            this.#listeners.set(
                event,
                list.filter((l) => l.callback !== callback)
            );
        }
        return this;
    }

    // Async emit — যখন listener গুলো async কাজ করে
    async emitAsync(event, ...args) {
        const listeners = this.#listeners.get(event) ?? [];

        for (const { callback } of listeners) {
            const result = await callback(...args);
            if (result === false) break;
        }
    }
}

// ব্যবহার — রাইড-শেয়ারিং ডিসপ্যাচ
const dispatch = new EventMediator();

dispatch.on("ride.requested", (ride) => {
    console.log(`🔍 কাছের ড্রাইভার খোঁজা হচ্ছে: ${ride.pickup}`);
}, { priority: 10 });

dispatch.on("ride.requested", (ride) => {
    console.log(`💰 ভাড়া হিসাব: ৳${ride.estimatedFare}`);
}, { priority: 20 });

dispatch.on("ride.accepted", (ride) => {
    console.log(`✅ ড্রাইভার ${ride.driver} গ্রহণ করেছে!`);
    console.log(`📱 ${ride.passenger} কে notification পাঠানো হচ্ছে...`);
});

dispatch.emit("ride.requested", {
    passenger: "ফারহান",
    pickup: "গুলশান-২",
    destination: "মতিঝিল",
    estimatedFare: 180,
});

dispatch.emit("ride.accepted", {
    passenger: "ফারহান",
    driver: "জামাল ভাই",
});
```

---

## 🌍 Real-World Applicable Areas

### ১. Laravel Event Dispatcher

Laravel-এর `event()` helper মূলত একটি **Mediator**! বিভিন্ন অংশ একে অপরকে না জেনেও communicate করতে পারে:

```php
// Laravel-এ Mediator হিসেবে Event System
// EventServiceProvider — Mediator configuration
class EventServiceProvider extends ServiceProvider
{
    protected $listen = [
        OrderPlaced::class => [
            SendOrderConfirmation::class,  // Listener A
            UpdateInventory::class,         // Listener B
            NotifyWarehouse::class,         // Listener C
        ],
    ];
}

// Controller থেকে event fire করো — Listeners কে জানার দরকার নেই
class OrderController extends Controller
{
    public function store(Request $request): JsonResponse
    {
        $order = Order::create($request->validated());

        // event() হলো Mediator.notify() এর সমতুল্য!
        event(new OrderPlaced($order));

        return response()->json($order, 201);
    }
}
```

### ২. Express.js EventEmitter

Node.js-এর `EventEmitter` একটি built-in Mediator:

```javascript
import { EventEmitter } from "node:events";

// Express app কে Mediator হিসেবে ব্যবহার
const app = new EventEmitter();

// বিভিন্ন module স্বাধীনভাবে listen করছে
app.on("user:registered", (user) => {
    console.log(`📧 Welcome email → ${user.email}`);
});

app.on("user:registered", (user) => {
    console.log(`📊 Analytics: নতুন ব্যবহারকারী tracked`);
});

app.on("user:registered", (user) => {
    console.log(`🎁 বোনাস পয়েন্ট যোগ করা হয়েছে: ${user.name}`);
});

// Registration endpoint
app.emit("user:registered", {
    name: "তানভীর",
    email: "tanvir@example.com",
});
```

### ৩. MVC Controller — Model এবং View এর Mediator

```
  ┌─────────┐         ┌──────────────┐         ┌──────────┐
  │  Model   │◄───────│  Controller  │────────►│   View   │
  │ (Data)   │        │  (Mediator)  │         │  (UI)    │
  └─────────┘         └──────────────┘         └──────────┘
       │                      ▲                       │
       │    data changed      │    user input         │
       └──────────────────────┴───────────────────────┘

  Model আর View একে অপরকে চেনে না!
  Controller (Mediator) সব coordinate করে।
```

### ৪. Full Working Example: Ride-Sharing Dispatch

```php
<?php

declare(strict_types=1);

// সম্পূর্ণ রাইড-শেয়ারিং ডিসপ্যাচ সিস্টেম

enum RideStatus: string
{
    case REQUESTED  = 'requested';
    case ACCEPTED   = 'accepted';
    case STARTED    = 'started';
    case COMPLETED  = 'completed';
    case CANCELLED  = 'cancelled';
}

readonly class Location
{
    public function __construct(
        public string $name,
        public float $lat,
        public float $lng,
    ) {}

    public function distanceTo(self $other): float
    {
        // Haversine formula (সরলীকৃত)
        return sqrt(($this->lat - $other->lat) ** 2 + ($this->lng - $other->lng) ** 2) * 111;
    }
}

interface DispatchMediator
{
    public function requestRide(Passenger $passenger, Location $pickup, Location $dest): void;
    public function acceptRide(Driver $driver, string $rideId): void;
    public function completeRide(string $rideId): void;
}

class Passenger
{
    public function __construct(
        public readonly string $name,
        public readonly string $phone,
    ) {}

    public function notifyDriverAssigned(string $driverName, string $vehicle): void
    {
        echo "📱 {$this->name}: ড্রাইভার {$driverName} ({$vehicle}) আসছে!\n";
    }

    public function notifyRideComplete(float $fare): void
    {
        echo "📱 {$this->name}: রাইড শেষ! ভাড়া: ৳{$fare}\n";
    }
}

class Driver
{
    public bool $available = true;

    public function __construct(
        public readonly string $name,
        public readonly string $vehicle,
        public Location $currentLocation,
    ) {}

    public function notifyNewRide(string $rideId, Location $pickup, float $fare): void
    {
        echo "📱 {$this->name}: নতুন রাইড #{$rideId} — {$pickup->name}, ভাড়া: ৳{$fare}\n";
    }
}

class RideDispatcher implements DispatchMediator
{
    /** @var array<string, Driver> */
    private array $drivers = [];
    private array $rides = [];

    public function registerDriver(Driver $driver): void
    {
        $this->drivers[$driver->name] = $driver;
    }

    public function requestRide(Passenger $passenger, Location $pickup, Location $dest): void
    {
        $rideId = 'RIDE-' . random_int(1000, 9999);
        $fare = round($pickup->distanceTo($dest) * 15, 2); // ৳15/km

        $this->rides[$rideId] = [
            'passenger' => $passenger,
            'pickup'    => $pickup,
            'dest'      => $dest,
            'fare'      => $fare,
            'status'    => RideStatus::REQUESTED,
        ];

        echo "🚗 নতুন রাইড অনুরোধ: {$passenger->name} ({$pickup->name} → {$dest->name})\n";

        // কাছের available ড্রাইভার খুঁজে বের করো
        $nearestDriver = $this->findNearestDriver($pickup);

        if ($nearestDriver) {
            $nearestDriver->notifyNewRide($rideId, $pickup, $fare);
            $this->acceptRide($nearestDriver, $rideId);
        } else {
            echo "❌ কোনো ড্রাইভার পাওয়া যায়নি!\n";
        }
    }

    public function acceptRide(Driver $driver, string $rideId): void
    {
        $ride = &$this->rides[$rideId];
        $ride['driver'] = $driver;
        $ride['status'] = RideStatus::ACCEPTED;
        $driver->available = false;

        $ride['passenger']->notifyDriverAssigned($driver->name, $driver->vehicle);
        echo "✅ {$driver->name} রাইড #{$rideId} গ্রহণ করেছে।\n";
    }

    public function completeRide(string $rideId): void
    {
        $ride = &$this->rides[$rideId];
        $ride['status'] = RideStatus::COMPLETED;
        $ride['driver']->available = true;

        $ride['passenger']->notifyRideComplete($ride['fare']);
        echo "🏁 রাইড #{$rideId} সম্পন্ন!\n";
    }

    private function findNearestDriver(Location $pickup): ?Driver
    {
        $nearest = null;
        $minDist = PHP_FLOAT_MAX;

        foreach ($this->drivers as $driver) {
            if (!$driver->available) continue;

            $dist = $driver->currentLocation->distanceTo($pickup);
            if ($dist < $minDist) {
                $minDist = $dist;
                $nearest = $driver;
            }
        }

        return $nearest;
    }
}

// ব্যবহার
$dispatcher = new RideDispatcher();

$dispatcher->registerDriver(new Driver('জামাল', 'Toyota Corolla', new Location('বনানী', 23.79, 90.40)));
$dispatcher->registerDriver(new Driver('সালাম', 'Honda Civic', new Location('উত্তরা', 23.87, 90.39)));

$passenger = new Passenger('ফারহান', '01712345678');
$dispatcher->requestRide(
    $passenger,
    new Location('গুলশান-২', 23.79, 90.41),
    new Location('মতিঝিল', 23.73, 90.42),
);
```

---

## 🔥 Advanced Deep Dive

### Mediator vs Observer — তুলনা

এই দুটো প্যাটার্ন দেখতে একই রকম মনে হয়, কিন্তু fundamental পার্থক্য আছে:

| বৈশিষ্ট্য | Mediator | Observer |
|-----------|----------|----------|
| **Direction** | Bidirectional (দুদিক) | Unidirectional (একদিক) |
| **Intelligence** | Mediator জানে কী করতে হবে | Observer শুধু notify পায় |
| **Coupling** | Colleagues → Mediator | Subject ← Observer |
| **Control** | Centralized (কেন্দ্রীভূত) | Distributed (বিকেন্দ্রীকৃত) |
| **Reuse** | Mediator specific | Observers reusable |
| **Example** | ATC ↔ Planes | YouTube → Subscribers |

```
Mediator:                          Observer:
    A ↔ M ↔ B                     Subject ──→ Observer1
    C ↗   ↘ D                              ──→ Observer2
 (M সিদ্ধান্ত নেয়)                        ──→ Observer3
                                    (Observers শুধু react করে)
```

**মূল পার্থক্য:** Observer-এ Subject জানে না Observer কী করবে। Mediator-এ Mediator জানে এবং সিদ্ধান্ত নেয় কোন Colleague-কে কী বলতে হবে।

### Mediator vs Facade

| বৈশিষ্ট্য | Mediator | Facade |
|-----------|----------|--------|
| **Direction** | Bidirectional | Unidirectional (client → system) |
| **Awareness** | Components Mediator জানে | Subsystem Facade জানে না |
| **Purpose** | Component interaction | Simple interface |
| **New behavior** | হ্যাঁ, নতুন interaction যোগ করা যায় | না, existing API wrap করে |

### Event-based Mediator (Pub/Sub)

```
Traditional Mediator:           Event-based Mediator:
┌──────────────┐               ┌──────────────────┐
│   Mediator    │               │  Event Bus       │
│ if A → do B  │               │  "event" → [     │
│ if B → do C  │               │    handler1,     │
│ (hardcoded)  │               │    handler2,     │
└──────────────┘               │  ]               │
                                └──────────────────┘
                                (runtime এ dynamic)
```

Event-based Mediator বেশি flexible কারণ runtime-এ নতুন listener যোগ/বাদ করা যায়।

### API Gateway — Microservice Mediator

```
Client Request
      │
      ▼
┌─────────────────┐
│   API Gateway    │ ← Mediator
│  (Kong/Nginx)    │
├─────────────────┤
│ Rate Limiting    │
│ Auth Check       │
│ Load Balancing   │
│ Request Routing  │
└──┬───┬───┬───┬──┘
   │   │   │   │
   ▼   ▼   ▼   ▼
  Auth User Order Payment
  Svc  Svc  Svc   Svc

Service গুলো একে অপরকে সরাসরি call করে না।
Gateway সব route করে — এটাই Mediator!
```

### 🚨 God Mediator Anti-Pattern

সবচেয়ে বড় বিপদ — Mediator যখন **God Object** হয়ে যায়:

```php
// ❌ God Mediator — এটি করবেন না!
class GodMediator implements Mediator
{
    public function notify(object $sender, string $event): void
    {
        // ৫০০+ লাইনের if-else বা match...
        // সব business logic এখানে ঢুকে গেছে
        // Mediator নিজেই monster হয়ে গেছে!
    }
}

// ✅ সমাধান: ছোট ছোট specialized mediator ব্যবহার করুন
// অথবা Event-based approach নিন যেখানে logic listener এ থাকে
```

**সমাধান কৌশল:**
1. একটি বড় Mediator-কে ভেঙে কয়েকটি ছোট Mediator বানান
2. Event-based Mediator ব্যবহার করুন যেখানে logic listener-এ থাকে
3. Strategy pattern combine করুন — interaction logic আলাদা class-এ রাখুন

### Laravel Event System Internals

```php
// Laravel-এর Dispatcher ক্লাস (সরলীকৃত ধারণা):
// Illuminate\Events\Dispatcher

// ১. event() helper → app('events')->dispatch()
// ২. Dispatcher::dispatch() → সব listener খুঁজে বের করে
// ৩. প্রতিটি listener কে call করে
// ৪. Queued listener হলে queue তে পাঠায়

// এটি মূলত আমাদের EventMediator ক্লাসের production-grade version!
// Laravel Dispatcher দেখলে বুঝবেন Mediator কিভাবে framework এ কাজ করে।
```

---

## ✅ Pros (সুবিধা)

| # | সুবিধা | ব্যাখ্যা |
|---|--------|----------|
| 1 | **Loose Coupling** | Component গুলো একে অপরকে জানে না |
| 2 | **Single Responsibility** | Communication logic এক জায়গায় |
| 3 | **Open/Closed Principle** | নতুন Mediator যোগ করলে existing code পরিবর্তন লাগে না |
| 4 | **Reusability** | Component গুলো স্বাধীনভাবে reuse করা যায় |
| 5 | **Testability** | Component আলাদাভাবে test করা যায় |
| 6 | **Centralized Control** | Debugging সহজ — সব interaction এক জায়গায় |

## ❌ Cons (অসুবিধা)

| # | অসুবিধা | ব্যাখ্যা |
|---|---------|----------|
| 1 | **God Object ঝুঁকি** | Mediator অতিরিক্ত বড় হতে পারে |
| 2 | **Single Point of Failure** | Mediator ভাঙলে সব ভাঙে |
| 3 | **Performance** | সব communication একটি point দিয়ে যায় — bottleneck |
| 4 | **Over-engineering** | সামান্য interaction-এ Mediator ব্যবহার overkill |
| 5 | **Debugging Complexity** | Event-based Mediator-এ flow trace করা কঠিন |

---

## ⚠️ Common Mistakes (সচরাচর ভুল)

### ভুল ১: Mediator-এ Business Logic রাখা

```php
// ❌ ভুল — Mediator-এ business logic
class OrderMediator implements Mediator
{
    public function notify(object $sender, string $event): void
    {
        if ($event === 'order.placed') {
            // Mediator নিজে tax calculate করছে — এটি ভুল!
            $tax = $sender->getAmount() * 0.15;
            $total = $sender->getAmount() + $tax;
            $this->paymentService->charge($total);
        }
    }
}

// ✅ সঠিক — Mediator শুধু coordinate করে
class OrderMediator implements Mediator
{
    public function notify(object $sender, string $event): void
    {
        if ($event === 'order.placed') {
            // Mediator শুধু বলছে কাকে কী করতে হবে
            $this->taxService->calculateFor($sender);
            $this->paymentService->chargeFor($sender);
        }
    }
}
```

### ভুল ২: সব Component এক Mediator-এ ঢোকানো

```php
// ❌ ভুল — একটি মাত্র Mediator সব কিছু handle করছে
class AppMediator
{
    // Auth, Order, Payment, Notification, Shipping, Analytics...
    // সবকিছু একটি ক্লাসে — God Object!
}

// ✅ সঠিক — Domain অনুযায়ী আলাদা Mediator
class AuthMediator { /* শুধু auth-related coordination */ }
class OrderMediator { /* শুধু order-related coordination */ }
class NotificationMediator { /* শুধু notification coordination */ }
```

### ভুল ৩: Circular Notification

```php
// ❌ ভুল — A → Mediator → B → Mediator → A → ... (অসীম loop!)
class BadMediator implements Mediator
{
    public function notify(object $sender, string $event): void
    {
        if ($sender instanceof A) {
            $this->b->doSomething(); // এটি notify() আবার call করবে
        }
        if ($sender instanceof B) {
            $this->a->doSomething(); // এটি আবার notify() call করবে!
        }
    }
}

// ✅ সঠিক — guard condition রাখুন
class SafeMediator implements Mediator
{
    private bool $processing = false;

    public function notify(object $sender, string $event): void
    {
        if ($this->processing) return; // Re-entry প্রতিরোধ

        $this->processing = true;
        // coordination logic...
        $this->processing = false;
    }
}
```

### ভুল ৪: Colleague গুলো একে অপরকে সরাসরি reference করা

```javascript
// ❌ ভুল
class ComponentA {
    constructor(componentB) {
        this.componentB = componentB; // সরাসরি reference — Mediator-এর উদ্দেশ্য ব্যর্থ!
    }
}

// ✅ সঠিক
class ComponentA {
    #mediator;
    constructor(mediator) {
        this.#mediator = mediator; // শুধু Mediator জানে
    }
}
```

---

## 🧪 টেস্টিং

### PHPUnit

```php
<?php

declare(strict_types=1);

use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\Attributes\Test;
use PHPUnit\Framework\Attributes\DataProvider;

class ChatRoomMediatorTest extends TestCase
{
    private ChatRoom $chatRoom;

    protected function setUp(): void
    {
        $this->chatRoom = new ChatRoom();
    }

    #[Test]
    public function user_can_join_chat_room(): void
    {
        $user = new User('টেস্ট ইউজার', $this->chatRoom);

        $this->assertContains(
            'টেস্ট ইউজার',
            $this->chatRoom->getOnlineUsers()
        );
    }

    #[Test]
    public function broadcast_message_reaches_all_except_sender(): void
    {
        $sender   = new User('প্রেরক', $this->chatRoom);
        $receiver1 = new User('প্রাপক-১', $this->chatRoom);
        $receiver2 = new User('প্রাপক-২', $this->chatRoom);

        $sender->sendBroadcast('হ্যালো সবাই!');

        // প্রেরক নিজে মেসেজ পাবে না
        $this->assertEmpty($sender->getInbox());

        // প্রাপকরা পাবে
        $this->assertCount(1, $receiver1->getInbox());
        $this->assertCount(1, $receiver2->getInbox());
        $this->assertStringContainsString('হ্যালো সবাই!', $receiver1->getInbox()[0]);
    }

    #[Test]
    public function direct_message_reaches_only_target(): void
    {
        $sender   = new User('প্রেরক', $this->chatRoom);
        $target   = new User('টার্গেট', $this->chatRoom);
        $bystander = new User('অন্যজন', $this->chatRoom);

        $sender->sendDirectMessage('গোপন মেসেজ', 'টার্গেট');

        $this->assertCount(1, $target->getInbox());
        $this->assertEmpty($bystander->getInbox());
        $this->assertEmpty($sender->getInbox());
    }

    #[Test]
    public function message_log_is_maintained(): void
    {
        $user1 = new User('ক', $this->chatRoom);
        $user2 = new User('খ', $this->chatRoom);

        $user1->sendBroadcast('মেসেজ-১');
        $user2->sendDirectMessage('মেসেজ-২', 'ক');

        $log = $this->chatRoom->getMessageLog();
        $this->assertCount(2, $log);
        $this->assertSame('ALL', $log[0]['to']);
        $this->assertSame('ক', $log[1]['to']);
    }
}

class EventMediatorTest extends TestCase
{
    #[Test]
    public function listeners_are_called_in_priority_order(): void
    {
        $mediator = new EventMediator();
        $callOrder = [];

        $mediator->on('test', function () use (&$callOrder) {
            $callOrder[] = 'low';
        }, priority: 1);

        $mediator->on('test', function () use (&$callOrder) {
            $callOrder[] = 'high';
        }, priority: 10);

        $mediator->emit('test');

        $this->assertSame(['high', 'low'], $callOrder);
    }

    #[Test]
    public function once_listener_fires_only_once(): void
    {
        $mediator = new EventMediator();
        $count = 0;

        $mediator->once('test', function () use (&$count) {
            $count++;
        });

        $mediator->emit('test');
        $mediator->emit('test');
        $mediator->emit('test');

        $this->assertSame(1, $count);
    }

    #[Test]
    public function propagation_stops_when_listener_returns_false(): void
    {
        $mediator = new EventMediator();
        $reached = false;

        $mediator->on('test', fn() => false, priority: 10);
        $mediator->on('test', function () use (&$reached) {
            $reached = true;
        }, priority: 1);

        $mediator->emit('test');

        $this->assertFalse($reached);
    }

    #[Test]
    public function off_removes_specific_listener(): void
    {
        $mediator = new EventMediator();
        $called = false;

        $listener = function () use (&$called) { $called = true; };

        $mediator->on('test', $listener);
        $mediator->off('test', $listener);
        $mediator->emit('test');

        $this->assertFalse($called);
    }
}

class DialogMediatorTest extends TestCase
{
    #[Test]
    public function selecting_district_updates_city(): void
    {
        $dialog = new RegistrationDialog();

        $dialog->districtDropdown->select('ঢাকা');

        $this->expectOutputRegex('/ঢাকা সিটি/');
    }

    #[Test]
    public function submit_button_disabled_without_terms(): void
    {
        $dialog = new RegistrationDialog();

        // শর্তাবলী ছাড়া সাবমিট — কাজ করবে না
        $dialog->submitButton->click();

        $this->expectOutputRegex('/নিষ্ক্রিয়/');
    }
}
```

### Jest (JavaScript)

```javascript
import { describe, it, expect, jest, beforeEach } from "@jest/globals";

describe("ChatRoom Mediator", () => {
    let chatRoom;

    beforeEach(() => {
        chatRoom = new ChatRoom();
    });

    it("ব্যবহারকারী চ্যাটরুমে যোগ দিতে পারে", () => {
        const user = new ChatUser("টেস্ট");
        chatRoom.addUser(user);

        expect(chatRoom.onlineUsers).toContain("টেস্ট");
    });

    it("broadcast সব ব্যবহারকারীকে পৌঁছায় (sender ছাড়া)", () => {
        const sender = new ChatUser("প্রেরক");
        const receiver1 = new ChatUser("প্রাপক-১");
        const receiver2 = new ChatUser("প্রাপক-২");

        chatRoom.addUser(sender);
        chatRoom.addUser(receiver1);
        chatRoom.addUser(receiver2);

        sender.sendBroadcast("হ্যালো!");

        expect(sender.inbox).toHaveLength(0);
        expect(receiver1.inbox).toHaveLength(1);
        expect(receiver2.inbox).toHaveLength(1);
        expect(receiver1.inbox[0]).toContain("হ্যালো!");
    });

    it("DM শুধু নির্দিষ্ট ব্যক্তির কাছে যায়", () => {
        const sender = new ChatUser("প্রেরক");
        const target = new ChatUser("টার্গেট");
        const other = new ChatUser("অন্যজন");

        [sender, target, other].forEach((u) => chatRoom.addUser(u));

        sender.sendDM("গোপন!", "টার্গেট");

        expect(target.inbox).toHaveLength(1);
        expect(other.inbox).toHaveLength(0);
    });

    it("message log সঠিকভাবে রাখে", () => {
        const u1 = new ChatUser("ক");
        const u2 = new ChatUser("খ");
        chatRoom.addUser(u1);
        chatRoom.addUser(u2);

        u1.sendBroadcast("মেসেজ-১");
        u2.sendDM("মেসেজ-২", "ক");

        const log = chatRoom.messageLog;
        expect(log).toHaveLength(2);
        expect(log[0].to).toBe("ALL");
        expect(log[1].to).toBe("ক");
    });
});

describe("EventMediator", () => {
    let mediator;

    beforeEach(() => {
        mediator = new EventMediator();
    });

    it("priority অনুযায়ী listener execute হয়", () => {
        const order = [];

        mediator.on("test", () => order.push("low"), { priority: 1 });
        mediator.on("test", () => order.push("high"), { priority: 10 });

        mediator.emit("test");

        expect(order).toEqual(["high", "low"]);
    });

    it("once listener একবারই fire হয়", () => {
        const fn = jest.fn();

        mediator.once("test", fn);

        mediator.emit("test");
        mediator.emit("test");
        mediator.emit("test");

        expect(fn).toHaveBeenCalledTimes(1);
    });

    it("false return করলে propagation থামে", () => {
        const secondListener = jest.fn();

        mediator.on("test", () => false, { priority: 10 });
        mediator.on("test", secondListener, { priority: 1 });

        mediator.emit("test");

        expect(secondListener).not.toHaveBeenCalled();
    });

    it("off দিয়ে listener remove করা যায়", () => {
        const fn = jest.fn();

        mediator.on("test", fn);
        mediator.off("test", fn);
        mediator.emit("test");

        expect(fn).not.toHaveBeenCalled();
    });

    it("event data সঠিকভাবে pass হয়", () => {
        const fn = jest.fn();

        mediator.on("order", fn);
        mediator.emit("order", { id: 1, total: 500 });

        expect(fn).toHaveBeenCalledWith({ id: 1, total: 500 });
    });
});
```

---

## 🔗 সম্পর্কিত প্যাটার্ন

### Observer প্যাটার্ন
Observer হলো Mediator-এর "ছোট ভাই"। Observer-এ subject তার observer-দের notify করে, কিন্তু কী হবে সেটা জানে না। Mediator জানে এবং সিদ্ধান্ত নেয়। Mediator-এর ভেতরে অনেক সময় Observer pattern ব্যবহার হয় (Event-based Mediator)।

### Facade প্যাটার্ন
Facade একটি subsystem-এর simplified interface দেয় — unidirectional। Mediator বিভিন্ন peer object-দের মধ্যে bidirectional communication চালায়। Facade জানে subsystem সম্পর্কে, কিন্তু subsystem Facade সম্পর্কে জানে না। Mediator-এ colleagues Mediator-কে জানে।

### Command প্যাটার্ন
Command প্যাটার্নের সাথে Mediator চমৎকারভাবে কাজ করে। Mediator requests কে Command object হিসেবে encapsulate করে colleague-দের কাছে পাঠাতে পারে। এতে undo/redo, queue, logging সুবিধা পাওয়া যায়।

### Chain of Responsibility
Chain of Responsibility-তে request একটি chain বরাবর যায়। Mediator-এ সব request central point দিয়ে যায়। দুটো combine করলে Mediator request পেয়ে chain-এ পাঠাতে পারে — এটি middleware pipeline এর ভিত্তি।

```
Mediator + Command:
  Colleague → Mediator → Command → Target Colleague

Mediator + Chain of Responsibility:
  Colleague → Mediator → Handler1 → Handler2 → ... → Final Handler

Mediator + Observer:
  Colleague → Mediator (EventBus) → Listener1, Listener2, ...
```

---

## 📏 কখন ব্যবহার করবেন / করবেন না

### ✅ কখন ব্যবহার করবেন

| পরিস্থিতি | কারণ |
|-----------|------|
| অনেক object একে অপরের সাথে communicate করে | `n²` coupling কমিয়ে `n` করতে |
| Communication logic পরিবর্তনযোগ্য হওয়া দরকার | Mediator swap করলেই behavior পরিবর্তন |
| Component reuse গুরুত্বপূর্ণ | Component আর specific peer-এর উপর নির্ভর করে না |
| UI form-এর component interaction | Dialog mediator হিসেবে কাজ করে |
| Chat/Messaging system | ChatRoom mediator |
| Event-driven architecture | Event Dispatcher হলো Mediator |
| Microservice orchestration | API Gateway mediator |
| Workflow/State machine | Central coordinator দরকার |

### ❌ কখন ব্যবহার করবেন না

| পরিস্থিতি | কারণ |
|-----------|------|
| মাত্র ২-৩টি object communicate করে | Over-engineering হবে |
| Simple Observer যথেষ্ট | অপ্রয়োজনীয় জটিলতা |
| Communication pattern fixed এবং simple | Mediator-এর flexibility লাগবে না |
| Performance critical path | Mediator bottleneck হতে পারে |
| Objects এর মধ্যে শুধু one-way notification | Observer ব্যবহার করুন |

### সিদ্ধান্ত গাছ (Decision Tree)

```
অনেক object interact করে?
├── না → Mediator লাগবে না
└── হ্যাঁ → Interaction bidirectional?
    ├── না → Observer ব্যবহার করুন
    └── হ্যাঁ → Interaction logic complex?
        ├── না → Simple delegation যথেষ্ট
        └── হ্যাঁ → Interaction runtime-এ পরিবর্তন হয়?
            ├── না → Traditional Mediator
            └── হ্যাঁ → Event-based Mediator
```

---

## 📋 সারসংক্ষেপ

### মূল শিক্ষা

```
┌──────────────────────────────────────────────────────────┐
│                   Mediator প্যাটার্ন                      │
├──────────────────────────────────────────────────────────┤
│  ✦ উদ্দেশ্য: Object-দের মধ্যে loose coupling নিশ্চিত     │
│    করা এবং communication centralize করা                  │
│                                                          │
│  ✦ কিভাবে: একটি central object (Mediator) সব            │
│    interaction coordinate করে                             │
│                                                          │
│  ✦ মনে রাখুন:                                            │
│    - Mediator = Traffic Controller                       │
│    - Colleagues শুধু Mediator চেনে, peer চেনে না          │
│    - Event-based Mediator সবচেয়ে flexible                │
│    - God Mediator anti-pattern থেকে সাবধান               │
│    - Laravel event(), Node EventEmitter = Mediator!       │
│                                                          │
│  ✦ বাংলাদেশ প্রসঙ্গ:                                     │
│    - Pathao/Uber dispatch = Mediator                     │
│    - Daraz chat = ChatRoom Mediator                      │
│    - bKash payment routing = Mediator                    │
└──────────────────────────────────────────────────────────┘
```

### Quick Reference

```php
// PHP — ন্যূনতম Mediator
interface Mediator {
    public function notify(object $sender, string $event): void;
}

class MyMediator implements Mediator {
    public function notify(object $sender, string $event): void {
        // coordinate colleagues based on event
    }
}
```

```javascript
// JS — ন্যূনতম Mediator
class EventMediator {
    #listeners = new Map();
    on(event, fn) { /* register */ }
    emit(event, ...args) { /* notify */ }
    off(event, fn) { /* remove */ }
}
```

### একনজরে তুলনা

| প্যাটার্ন | Mediator-এর সাথে সম্পর্ক |
|-----------|--------------------------|
| Observer | Mediator-এর ভেতরে ব্যবহৃত হতে পারে |
| Facade | Unidirectional vs Bidirectional |
| Command | Mediator command দিয়ে communicate করতে পারে |
| CoR | Mediator request chain-এ পাঠাতে পারে |

> **"যখন সবাই সবার সাথে কথা বলে, কেউ কারো কথা শোনে না।
> একজন Mediator দিন — শৃঙ্খলা আসবে।"** 🤝
