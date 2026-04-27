# 📜 Command প্যাটার্ন (Command Pattern)

## 📌 সংজ্ঞা (Definition)

**Gang of Four (GoF) সংজ্ঞা:**
> "Encapsulate a request as an object, thereby letting you parameterize clients with different requests, queue or log requests, and support undoable operations."

**বাংলায়:** একটি রিকোয়েস্টকে একটি অবজেক্ট হিসেবে এনক্যাপসুলেট করো, যাতে তুমি ক্লায়েন্টকে বিভিন্ন রিকোয়েস্ট দিয়ে প্যারামিটারাইজ করতে পারো, রিকোয়েস্ট কিউ বা লগ করতে পারো, এবং আনডু-যোগ্য অপারেশন সাপোর্ট করতে পারো।

### মূল ধারণা

Command প্যাটার্ন হলো একটি **Behavioral Design Pattern** যেটা একটা অ্যাকশন বা অপারেশনকে একটা স্বতন্ত্র অবজেক্টে পরিণত করে। এই অবজেক্টের মধ্যে অ্যাকশন সম্পাদনের জন্য প্রয়োজনীয় সব তথ্য থাকে — কোন মেথড কল করতে হবে, কী কী আর্গুমেন্ট দিতে হবে, এবং কোন অবজেক্টের উপর এই মেথড কল হবে।

এটা মূলত **Sender** (যে রিকোয়েস্ট পাঠায়) এবং **Receiver** (যে রিকোয়েস্ট এক্সিকিউট করে) — এদের মধ্যে **decoupling** তৈরি করে। Sender-এর জানার দরকার নেই Receiver কে বা কীভাবে কাজটা হবে। সে শুধু Command অবজেক্টকে বলে — "এক্সিকিউট করো!"

### চারটি মূল কম্পোনেন্ট

| কম্পোনেন্ট | ভূমিকা |
|---|---|
| **Command** (Interface) | `execute()` মেথড ডিক্লেয়ার করে |
| **ConcreteCommand** | Command ইন্টারফেস ইমপ্লিমেন্ট করে, Receiver-কে কল করে |
| **Invoker** | Command অবজেক্ট ধারণ করে এবং trigger করে |
| **Receiver** | আসল বিজনেস লজিক — কাজটা আসলে এখানে হয় |

---

## 🏠 বাস্তব উদাহরণ (Real-World Analogy)

### 🍽️ রেস্টুরেন্ট অর্ডার সিস্টেম

ধরো তুমি একটা রেস্টুরেন্টে গেছো:

```
তুমি (Client) → অর্ডার স্লিপ লেখো (Command তৈরি)
         ↓
ওয়েটার (Invoker) → অর্ডার স্লিপ নেয়, কিচেনে পাঠায়
         ↓
শেফ (Receiver) → অর্ডার স্লিপ দেখে রান্না করে (execute)
```

- **তুমি** জানো না কোন শেফ রান্না করবে
- **ওয়েটার** জানে না রান্নার রেসিপি কী
- **অর্ডার স্লিপ** হলো সেই Command অবজেক্ট — যেটাতে সব তথ্য আছে
- অর্ডার **ক্যান্সেল** করা = Undo অপারেশন
- একাধিক অর্ডার **কিউতে** রাখা = Command Queue

### 🇧🇩 বাংলাদেশ কনটেক্সট: bKash ট্রানজ্যাকশন

```
ইউজার (Client) → "1000 টাকা পাঠাও 01712345678 নম্বরে" (SendMoneyCommand)
         ↓
bKash App (Invoker) → Command কিউতে রাখে, PIN ভেরিফাই করে
         ↓
Payment Gateway (Receiver) → আসল টাকা ট্রান্সফার করে
         ↓
ব্যর্থ হলে? → undo() — টাকা ফেরত আসে (Rollback)
```

---

## 📊 UML ডায়াগ্রাম (ASCII)

```
┌─────────────────┐          ┌──────────────────────┐
│     Client       │          │    <<interface>>      │
│                  │─creates─▶│      Command          │
│                  │          │──────────────────────│
└────────┬────────┘          │ + execute(): void     │
         │                   │ + undo(): void        │
         │                   └──────────┬───────────┘
         │                              │
         │                    ┌─────────┴──────────┐
         │                    │                    │
         │         ┌──────────▼──────┐  ┌─────────▼────────┐
         │         │ ConcreteCommandA │  │ ConcreteCommandB  │
         │         │─────────────────│  │──────────────────│
         │         │ - receiver      │  │ - receiver        │
         │         │ - params        │  │ - params          │
         │         │─────────────────│  │──────────────────│
         │         │ + execute()     │  │ + execute()       │
         │         │ + undo()        │  │ + undo()          │
         │         └────────┬────────┘  └────────┬─────────┘
         │                  │                     │
         │                  ▼                     ▼
         │         ┌─────────────────┐   ┌────────────────┐
         │         │   ReceiverA      │   │   ReceiverB     │
         │         │─────────────────│   │────────────────│
         │         │ + action()      │   │ + action()      │
         │         └─────────────────┘   └────────────────┘
         │
         ▼
┌─────────────────┐
│     Invoker      │
│─────────────────│
│ - command        │
│ - history[]      │
│─────────────────│
│ + setCommand()   │
│ + executeCmd()   │
│ + undoCmd()      │
└─────────────────┘
```

### সিকোয়েন্স ডায়াগ্রাম

```
Client          Invoker         Command         Receiver
  │                │                │                │
  │──create cmd───▶│                │                │
  │                │                │                │
  │──execute()────▶│                │                │
  │                │──execute()────▶│                │
  │                │                │──action()─────▶│
  │                │                │                │──does work
  │                │                │◀──result───────│
  │                │◀──done─────────│                │
  │                │                │                │
  │──undo()───────▶│                │                │
  │                │──undo()───────▶│                │
  │                │                │──reverse()────▶│
  │                │                │◀──result───────│
  │◀──undone───────│                │                │
```

---

## 💻 ইমপ্লিমেন্টেশন (Implementation)

### 1️⃣ Basic Command Pattern

#### PHP 8.3

```php
<?php

declare(strict_types=1);

// Command Interface
interface Command
{
    public function execute(): void;
    public function undo(): void;
}

// Receiver — আসল কাজ এখানে হয়
class Light
{
    private bool $isOn = false;
    private int $brightness = 0;

    public function __construct(
        private readonly string $location
    ) {}

    public function turnOn(): void
    {
        $this->isOn = true;
        $this->brightness = 100;
        echo "💡 {$this->location} এর লাইট জ্বলে গেছে (brightness: {$this->brightness}%)\n";
    }

    public function turnOff(): void
    {
        $this->isOn = false;
        $this->brightness = 0;
        echo "🌑 {$this->location} এর লাইট নিভে গেছে\n";
    }

    public function dim(int $level): void
    {
        $this->brightness = max(0, min(100, $level));
        echo "🔅 {$this->location} এর লাইট ডিম করা হয়েছে ({$this->brightness}%)\n";
    }

    public function getBrightness(): int
    {
        return $this->brightness;
    }
}

// Concrete Commands
class LightOnCommand implements Command
{
    private int $previousBrightness = 0;

    public function __construct(
        private readonly Light $light
    ) {}

    public function execute(): void
    {
        $this->previousBrightness = $this->light->getBrightness();
        $this->light->turnOn();
    }

    public function undo(): void
    {
        // আগের অবস্থায় ফেরত যাও
        if ($this->previousBrightness === 0) {
            $this->light->turnOff();
        } else {
            $this->light->dim($this->previousBrightness);
        }
    }
}

class LightOffCommand implements Command
{
    private int $previousBrightness = 0;

    public function __construct(
        private readonly Light $light
    ) {}

    public function execute(): void
    {
        $this->previousBrightness = $this->light->getBrightness();
        $this->light->turnOff();
    }

    public function undo(): void
    {
        if ($this->previousBrightness > 0) {
            $this->light->turnOn();
            $this->light->dim($this->previousBrightness);
        }
    }
}

class LightDimCommand implements Command
{
    private int $previousBrightness = 0;

    public function __construct(
        private readonly Light $light,
        private readonly int $level
    ) {}

    public function execute(): void
    {
        $this->previousBrightness = $this->light->getBrightness();
        $this->light->dim($this->level);
    }

    public function undo(): void
    {
        $this->light->dim($this->previousBrightness);
    }
}

// Invoker — রিমোট কন্ট্রোল
class RemoteControl
{
    /** @var array<string, Command> */
    private array $slots = [];

    /** @var Command[] */
    private array $history = [];

    public function setCommand(string $slot, Command $command): void
    {
        $this->slots[$slot] = $command;
    }

    public function pressButton(string $slot): void
    {
        if (!isset($this->slots[$slot])) {
            echo "⚠️ '{$slot}' স্লটে কোনো কমান্ড সেট করা নেই\n";
            return;
        }

        $command = $this->slots[$slot];
        $command->execute();
        $this->history[] = $command;
    }

    public function pressUndo(): void
    {
        if (empty($this->history)) {
            echo "⚠️ আনডু করার কিছু নেই\n";
            return;
        }

        $lastCommand = array_pop($this->history);
        $lastCommand->undo();
        echo "↩️ সর্বশেষ কমান্ড আনডু করা হয়েছে\n";
    }
}

// ব্যবহার
$bedroomLight = new Light('বেডরুম');
$kitchenLight = new Light('রান্নাঘর');

$remote = new RemoteControl();
$remote->setCommand('bedroom_on', new LightOnCommand($bedroomLight));
$remote->setCommand('bedroom_off', new LightOffCommand($bedroomLight));
$remote->setCommand('bedroom_dim', new LightDimCommand($bedroomLight, 40));
$remote->setCommand('kitchen_on', new LightOnCommand($kitchenLight));

$remote->pressButton('bedroom_on');   // 💡 বেডরুম এর লাইট জ্বলে গেছে
$remote->pressButton('kitchen_on');   // 💡 রান্নাঘর এর লাইট জ্বলে গেছে
$remote->pressButton('bedroom_dim');  // 🔅 বেডরুম এর লাইট ডিম করা হয়েছে
$remote->pressUndo();                 // ↩️ আনডু — আগের brightness-এ ফিরবে
```

#### JavaScript (ES2022+)

```javascript
// Command Interface (ক্লাস-বেসড)
class Command {
    execute() { throw new Error('execute() must be implemented'); }
    undo() { throw new Error('undo() must be implemented'); }
}

// Receiver
class Light {
    #isOn = false;
    #brightness = 0;
    #location;

    constructor(location) {
        this.#location = location;
    }

    turnOn() {
        this.#isOn = true;
        this.#brightness = 100;
        console.log(`💡 ${this.#location} এর লাইট জ্বলে গেছে (brightness: ${this.#brightness}%)`);
    }

    turnOff() {
        this.#isOn = false;
        this.#brightness = 0;
        console.log(`🌑 ${this.#location} এর লাইট নিভে গেছে`);
    }

    dim(level) {
        this.#brightness = Math.max(0, Math.min(100, level));
        console.log(`🔅 ${this.#location} এর লাইট ডিম করা হয়েছে (${this.#brightness}%)`);
    }

    get brightness() { return this.#brightness; }
}

// Concrete Commands
class LightOnCommand extends Command {
    #light;
    #previousBrightness = 0;

    constructor(light) {
        super();
        this.#light = light;
    }

    execute() {
        this.#previousBrightness = this.#light.brightness;
        this.#light.turnOn();
    }

    undo() {
        this.#previousBrightness === 0
            ? this.#light.turnOff()
            : this.#light.dim(this.#previousBrightness);
    }
}

class LightOffCommand extends Command {
    #light;
    #previousBrightness = 0;

    constructor(light) {
        super();
        this.#light = light;
    }

    execute() {
        this.#previousBrightness = this.#light.brightness;
        this.#light.turnOff();
    }

    undo() {
        if (this.#previousBrightness > 0) {
            this.#light.turnOn();
            this.#light.dim(this.#previousBrightness);
        }
    }
}

// Invoker
class RemoteControl {
    #slots = new Map();
    #history = [];

    setCommand(slot, command) {
        this.#slots.set(slot, command);
    }

    pressButton(slot) {
        const command = this.#slots.get(slot);
        if (!command) {
            console.log(`⚠️ '${slot}' স্লটে কোনো কমান্ড নেই`);
            return;
        }
        command.execute();
        this.#history.push(command);
    }

    pressUndo() {
        const lastCommand = this.#history.pop();
        if (!lastCommand) {
            console.log('⚠️ আনডু করার কিছু নেই');
            return;
        }
        lastCommand.undo();
        console.log('↩️ সর্বশেষ কমান্ড আনডু করা হয়েছে');
    }
}

// ব্যবহার
const bedroomLight = new Light('বেডরুম');
const remote = new RemoteControl();
remote.setCommand('on', new LightOnCommand(bedroomLight));
remote.setCommand('off', new LightOffCommand(bedroomLight));

remote.pressButton('on');
remote.pressButton('off');
remote.pressUndo();
```

---

### 2️⃣ Undo/Redo System — টেক্সট এডিটর

এটা Command প্যাটার্নের সবচেয়ে ক্লাসিক এবং শক্তিশালী ব্যবহার। প্রতিটা এডিটিং অপারেশন একটা Command অবজেক্ট — যেটা নিজের কাজ নিজে আনডু করতে পারে।

#### PHP 8.3

```php
<?php

declare(strict_types=1);

interface EditorCommand
{
    public function execute(): void;
    public function undo(): void;
    public function getDescription(): string;
}

class TextDocument
{
    private string $content = '';
    private int $cursorPosition = 0;

    public function insert(string $text, int $position): void
    {
        $this->content = substr($this->content, 0, $position)
            . $text
            . substr($this->content, $position);
        $this->cursorPosition = $position + strlen($text);
    }

    public function delete(int $position, int $length): string
    {
        $deleted = substr($this->content, $position, $length);
        $this->content = substr($this->content, 0, $position)
            . substr($this->content, $position + $length);
        $this->cursorPosition = $position;
        return $deleted;
    }

    public function replace(int $position, int $length, string $newText): string
    {
        $old = substr($this->content, $position, $length);
        $this->content = substr($this->content, 0, $position)
            . $newText
            . substr($this->content, $position + $length);
        $this->cursorPosition = $position + strlen($newText);
        return $old;
    }

    public function getContent(): string { return $this->content; }
    public function getCursorPosition(): int { return $this->cursorPosition; }
}

// টেক্সট ইনসার্ট কমান্ড
class InsertTextCommand implements EditorCommand
{
    public function __construct(
        private readonly TextDocument $document,
        private readonly string $text,
        private readonly int $position
    ) {}

    public function execute(): void
    {
        $this->document->insert($this->text, $this->position);
    }

    public function undo(): void
    {
        // insert-এর উল্টো হলো delete
        $this->document->delete($this->position, strlen($this->text));
    }

    public function getDescription(): string
    {
        return "Insert '{$this->text}' at position {$this->position}";
    }
}

// টেক্সট ডিলিট কমান্ড
class DeleteTextCommand implements EditorCommand
{
    private string $deletedText = '';

    public function __construct(
        private readonly TextDocument $document,
        private readonly int $position,
        private readonly int $length
    ) {}

    public function execute(): void
    {
        // ডিলিট করার আগে টেক্সট সেভ করো — আনডুর জন্য লাগবে
        $this->deletedText = $this->document->delete($this->position, $this->length);
    }

    public function undo(): void
    {
        $this->document->insert($this->deletedText, $this->position);
    }

    public function getDescription(): string
    {
        return "Delete {$this->length} chars at position {$this->position}";
    }
}

// রিপ্লেস কমান্ড
class ReplaceTextCommand implements EditorCommand
{
    private string $oldText = '';

    public function __construct(
        private readonly TextDocument $document,
        private readonly int $position,
        private readonly int $length,
        private readonly string $newText
    ) {}

    public function execute(): void
    {
        $this->oldText = $this->document->replace(
            $this->position, $this->length, $this->newText
        );
    }

    public function undo(): void
    {
        $this->document->replace(
            $this->position, strlen($this->newText), $this->oldText
        );
    }

    public function getDescription(): string
    {
        return "Replace '{$this->oldText}' with '{$this->newText}'";
    }
}

// কমান্ড হিস্ট্রি ম্যানেজার — Undo/Redo ইঞ্জিন
class CommandHistory
{
    /** @var EditorCommand[] */
    private array $undoStack = [];

    /** @var EditorCommand[] */
    private array $redoStack = [];

    private int $maxHistory;

    public function __construct(int $maxHistory = 100)
    {
        $this->maxHistory = $maxHistory;
    }

    public function executeCommand(EditorCommand $command): void
    {
        $command->execute();
        $this->undoStack[] = $command;

        // নতুন কমান্ড এক্সিকিউট হলে redo স্ট্যাক খালি হবে
        // কারণ নতুন ব্রাঞ্চ তৈরি হয়েছে
        $this->redoStack = [];

        // সর্বোচ্চ হিস্ট্রি সীমা বজায় রাখো
        if (count($this->undoStack) > $this->maxHistory) {
            array_shift($this->undoStack);
        }
    }

    public function undo(): bool
    {
        if (empty($this->undoStack)) {
            echo "⚠️ আনডু করার কিছু নেই!\n";
            return false;
        }

        $command = array_pop($this->undoStack);
        $command->undo();
        $this->redoStack[] = $command;

        echo "↩️ Undo: {$command->getDescription()}\n";
        return true;
    }

    public function redo(): bool
    {
        if (empty($this->redoStack)) {
            echo "⚠️ রিডু করার কিছু নেই!\n";
            return false;
        }

        $command = array_pop($this->redoStack);
        $command->execute();
        $this->undoStack[] = $command;

        echo "↪️ Redo: {$command->getDescription()}\n";
        return true;
    }

    public function canUndo(): bool { return !empty($this->undoStack); }
    public function canRedo(): bool { return !empty($this->redoStack); }

    public function getHistory(): array
    {
        return array_map(
            fn(EditorCommand $cmd) => $cmd->getDescription(),
            $this->undoStack
        );
    }
}

// ব্যবহার
$doc = new TextDocument();
$history = new CommandHistory();

$history->executeCommand(new InsertTextCommand($doc, 'Hello ', 0));
echo "ডকুমেন্ট: '{$doc->getContent()}'\n";

$history->executeCommand(new InsertTextCommand($doc, 'World', 6));
echo "ডকুমেন্ট: '{$doc->getContent()}'\n";

$history->executeCommand(new ReplaceTextCommand($doc, 0, 5, 'Hola'));
echo "ডকুমেন্ট: '{$doc->getContent()}'\n";

$history->undo(); // Replace আনডু
echo "ডকুমেন্ট: '{$doc->getContent()}'\n"; // "Hello World"

$history->undo(); // দ্বিতীয় Insert আনডু
echo "ডকুমেন্ট: '{$doc->getContent()}'\n"; // "Hello "

$history->redo(); // Insert আবার
echo "ডকুমেন্ট: '{$doc->getContent()}'\n"; // "Hello World"
```

#### JavaScript (ES2022+)

```javascript
class TextDocument {
    #content = '';

    insert(text, position) {
        this.#content = this.#content.slice(0, position) + text + this.#content.slice(position);
    }

    delete(position, length) {
        const deleted = this.#content.slice(position, position + length);
        this.#content = this.#content.slice(0, position) + this.#content.slice(position + length);
        return deleted;
    }

    get content() { return this.#content; }
}

class InsertTextCommand {
    #doc; #text; #position;

    constructor(doc, text, position) {
        this.#doc = doc;
        this.#text = text;
        this.#position = position;
    }

    execute() { this.#doc.insert(this.#text, this.#position); }
    undo() { this.#doc.delete(this.#position, this.#text.length); }
    get description() { return `Insert '${this.#text}' at ${this.#position}`; }
}

class DeleteTextCommand {
    #doc; #position; #length; #deletedText = '';

    constructor(doc, position, length) {
        this.#doc = doc;
        this.#position = position;
        this.#length = length;
    }

    execute() { this.#deletedText = this.#doc.delete(this.#position, this.#length); }
    undo() { this.#doc.insert(this.#deletedText, this.#position); }
    get description() { return `Delete ${this.#length} chars at ${this.#position}`; }
}

// Undo/Redo ম্যানেজার
class CommandHistory {
    #undoStack = [];
    #redoStack = [];

    execute(command) {
        command.execute();
        this.#undoStack.push(command);
        this.#redoStack.length = 0; // নতুন কমান্ডে redo ক্লিয়ার
    }

    undo() {
        const cmd = this.#undoStack.pop();
        if (!cmd) return console.log('⚠️ আনডু করার কিছু নেই');
        cmd.undo();
        this.#redoStack.push(cmd);
        console.log(`↩️ Undo: ${cmd.description}`);
    }

    redo() {
        const cmd = this.#redoStack.pop();
        if (!cmd) return console.log('⚠️ রিডু করার কিছু নেই');
        cmd.execute();
        this.#undoStack.push(cmd);
        console.log(`↪️ Redo: ${cmd.description}`);
    }

    get history() { return this.#undoStack.map(c => c.description); }
}

// ব্যবহার
const doc = new TextDocument();
const history = new CommandHistory();

history.execute(new InsertTextCommand(doc, 'স্বাগতম ', 0));
history.execute(new InsertTextCommand(doc, 'বাংলাদেশ', 9));
console.log(`ডকুমেন্ট: "${doc.content}"`);

history.undo();
console.log(`আনডু পরে: "${doc.content}"`);

history.redo();
console.log(`রিডু পরে: "${doc.content}"`);
```

---

### 3️⃣ Macro Command (Composite Command)

একাধিক Command কে একটি Command হিসেবে গ্রুপ করা — এটা Composite প্যাটার্নের সাথে Command-এর কম্বিনেশন। যেমন "সকালের রুটিন" — একটা বাটনে লাইট জ্বালাও, কফি মেশিন চালু করো, AC বন্ধ করো।

#### PHP 8.3

```php
<?php

declare(strict_types=1);

// MacroCommand — একাধিক কমান্ড একসাথে
class MacroCommand implements Command
{
    /** @var Command[] */
    private array $commands;

    public function __construct(Command ...$commands)
    {
        $this->commands = $commands;
    }

    public function execute(): void
    {
        foreach ($this->commands as $command) {
            $command->execute();
        }
    }

    public function undo(): void
    {
        // আনডু করতে হবে উল্টো ক্রমে (LIFO)
        foreach (array_reverse($this->commands) as $command) {
            $command->undo();
        }
    }

    public function addCommand(Command $command): void
    {
        $this->commands[] = $command;
    }
}

// Transactional Macro — হয় সব হবে, নয় কিছুই না (All or Nothing)
class TransactionalMacroCommand implements Command
{
    /** @var Command[] */
    private array $commands;

    /** @var Command[] */
    private array $executedCommands = [];

    public function __construct(Command ...$commands)
    {
        $this->commands = $commands;
    }

    public function execute(): void
    {
        $this->executedCommands = [];

        foreach ($this->commands as $command) {
            try {
                $command->execute();
                $this->executedCommands[] = $command;
            } catch (\Throwable $e) {
                echo "❌ কমান্ড ব্যর্থ! রোলব্যাক শুরু হচ্ছে...\n";
                $this->undo(); // সব আনডু — ট্রানজ্যাকশনাল
                throw new \RuntimeException(
                    "Macro execution failed: {$e->getMessage()}", 0, $e
                );
            }
        }
    }

    public function undo(): void
    {
        foreach (array_reverse($this->executedCommands) as $command) {
            $command->undo();
        }
        $this->executedCommands = [];
    }
}

// ব্যবহার — স্মার্ট হোম "গুড মর্নিং" ম্যাক্রো
$bedroomLight = new Light('বেডরুম');
$kitchenLight = new Light('রান্নাঘর');

$morningRoutine = new MacroCommand(
    new LightOnCommand($bedroomLight),
    new LightOnCommand($kitchenLight),
    new LightDimCommand($bedroomLight, 60),
);

echo "🌅 সকালের রুটিন চালু:\n";
$morningRoutine->execute();

echo "\n🌙 রাতের রুটিন — সব আনডু:\n";
$morningRoutine->undo();
```

#### JavaScript (ES2022+)

```javascript
class MacroCommand {
    #commands;

    constructor(...commands) {
        this.#commands = commands;
    }

    execute() {
        for (const cmd of this.#commands) {
            cmd.execute();
        }
    }

    undo() {
        // উল্টো ক্রমে আনডু
        for (const cmd of [...this.#commands].reverse()) {
            cmd.undo();
        }
    }
}

// Transactional — ব্যর্থ হলে সব রোলব্যাক
class TransactionalMacro {
    #commands;
    #executed = [];

    constructor(...commands) {
        this.#commands = commands;
    }

    execute() {
        this.#executed = [];
        for (const cmd of this.#commands) {
            try {
                cmd.execute();
                this.#executed.push(cmd);
            } catch (err) {
                console.log('❌ ব্যর্থ! রোলব্যাক শুরু...');
                this.undo();
                throw new Error(`Macro failed: ${err.message}`);
            }
        }
    }

    undo() {
        for (const cmd of [...this.#executed].reverse()) {
            cmd.undo();
        }
        this.#executed = [];
    }
}

// ব্যবহার
const bedroom = new Light('বেডরুম');
const kitchen = new Light('রান্নাঘর');

const morning = new MacroCommand(
    new LightOnCommand(bedroom),
    new LightOnCommand(kitchen),
);

morning.execute();  // দুটোই জ্বলবে
morning.undo();     // দুটোই নিভবে
```

---

### 4️⃣ Queued Command — Async Execution

এটা Command প্যাটার্নের সবচেয়ে প্র্যাকটিক্যাল ব্যবহার — Laravel Queue, Bull MQ, RabbitMQ সব এই প্যাটার্নে কাজ করে। Command অবজেক্ট সিরিয়ালাইজ করে কিউতে রাখো, পরে একটা Worker এসে execute করে।

#### PHP 8.3

```php
<?php

declare(strict_types=1);

// Queued Command Interface — সিরিয়ালাইজেশন সাপোর্ট
interface QueueableCommand extends Command
{
    public function serialize(): string;
    public static function deserialize(string $data): static;
    public function getQueue(): string;
    public function getDelay(): int; // সেকেন্ডে
    public function getMaxRetries(): int;
}

// bKash Send Money Command — বাংলাদেশ কনটেক্সট
class SendMoneyCommand implements QueueableCommand
{
    private bool $executed = false;
    private ?string $transactionId = null;

    public function __construct(
        private readonly string $senderPhone,
        private readonly string $receiverPhone,
        private readonly float $amount,
        private readonly string $pin,
    ) {}

    public function execute(): void
    {
        // ব্যালেন্স চেক
        if ($this->amount <= 0) {
            throw new \InvalidArgumentException('অবৈধ পরিমাণ');
        }

        // PIN ভেরিফিকেশন (সিমুলেটেড)
        echo "🔐 PIN ভেরিফাই হচ্ছে...\n";
        echo "💸 ৳{$this->amount} পাঠানো হচ্ছে {$this->senderPhone} → {$this->receiverPhone}\n";

        // ট্রানজ্যাকশন ID জেনারেট
        $this->transactionId = 'TXN' . bin2hex(random_bytes(8));
        $this->executed = true;

        echo "✅ সফল! Transaction ID: {$this->transactionId}\n";
    }

    public function undo(): void
    {
        if (!$this->executed || !$this->transactionId) {
            echo "⚠️ আনডু করার কিছু নেই — ট্রানজ্যাকশন হয়নি\n";
            return;
        }

        // রিভার্স ট্রানজ্যাকশন
        echo "↩️ ৳{$this->amount} ফেরত আসছে {$this->receiverPhone} → {$this->senderPhone}\n";
        echo "🔄 Refund Transaction for: {$this->transactionId}\n";
        $this->executed = false;
    }

    public function serialize(): string
    {
        return json_encode([
            'type' => self::class,
            'sender' => $this->senderPhone,
            'receiver' => $this->receiverPhone,
            'amount' => $this->amount,
            'pin' => '***', // PIN কখনো সিরিয়ালাইজ করো না!
        ]);
    }

    public static function deserialize(string $data): static
    {
        $payload = json_decode($data, true);
        return new static(
            $payload['sender'],
            $payload['receiver'],
            $payload['amount'],
            '' // PIN আলাদাভাবে নিরাপদে পাঠাতে হবে
        );
    }

    public function getQueue(): string { return 'transactions'; }
    public function getDelay(): int { return 0; }
    public function getMaxRetries(): int { return 3; }
}

// Command Queue — সিম্পল ইন-মেমরি কিউ
class CommandQueue
{
    /** @var array<int, array{command: QueueableCommand, attempts: int, scheduledAt: float}> */
    private array $queue = [];

    /** @var array<string, QueueableCommand[]> */
    private array $failedJobs = [];

    public function push(QueueableCommand $command): void
    {
        $this->queue[] = [
            'command' => $command,
            'attempts' => 0,
            'scheduledAt' => microtime(true) + $command->getDelay(),
        ];
        echo "📥 কমান্ড কিউতে যোগ হয়েছে: {$command->getQueue()}\n";
    }

    public function process(): void
    {
        echo "⚙️ কিউ প্রসেসিং শুরু...\n";

        while (!empty($this->queue)) {
            $job = array_shift($this->queue);
            $command = $job['command'];
            $attempts = $job['attempts'];

            // সময় চেক (delayed job)
            if ($job['scheduledAt'] > microtime(true)) {
                $this->queue[] = $job; // আবার কিউতে রাখো
                continue;
            }

            try {
                $attempts++;
                echo "\n🔄 প্রসেসিং (attempt {$attempts}/{$command->getMaxRetries()})...\n";
                $command->execute();
                echo "✅ সফলভাবে প্রসেস হয়েছে\n";
            } catch (\Throwable $e) {
                echo "❌ ব্যর্থ: {$e->getMessage()}\n";

                if ($attempts < $command->getMaxRetries()) {
                    // আবার চেষ্টা করো
                    $this->queue[] = [
                        'command' => $command,
                        'attempts' => $attempts,
                        'scheduledAt' => microtime(true) + (2 ** $attempts), // exponential backoff
                    ];
                    echo "🔁 পুনরায় চেষ্টা হবে {$attempts} সেকেন্ড পরে\n";
                } else {
                    // Failed job
                    $this->failedJobs[$command->getQueue()][] = $command;
                    echo "💀 সর্বোচ্চ retry শেষ — failed jobs-এ যোগ হয়েছে\n";
                }
            }
        }

        echo "\n✅ কিউ প্রসেসিং সম্পন্ন\n";
    }

    public function getFailedJobs(): array { return $this->failedJobs; }
}

// ব্যবহার
$queue = new CommandQueue();

$queue->push(new SendMoneyCommand('01712345678', '01898765432', 500.00, '12345'));
$queue->push(new SendMoneyCommand('01712345678', '01611111111', 1000.00, '12345'));

$queue->process();
```

#### JavaScript (ES2022+)

```javascript
// Async Command Queue — Node.js স্টাইল
class AsyncCommandQueue {
    #queue = [];
    #processing = false;
    #concurrency;
    #activeCount = 0;

    constructor(concurrency = 3) {
        this.#concurrency = concurrency;
    }

    enqueue(command, { delay = 0, maxRetries = 3 } = {}) {
        this.#queue.push({
            command,
            delay,
            maxRetries,
            attempts: 0,
            enqueuedAt: Date.now(),
        });
        console.log(`📥 কমান্ড কিউতে যোগ হলো (queue size: ${this.#queue.length})`);
    }

    async process() {
        if (this.#processing) return;
        this.#processing = true;
        console.log('⚙️ কিউ প্রসেসিং শুরু...');

        while (this.#queue.length > 0 || this.#activeCount > 0) {
            while (this.#activeCount < this.#concurrency && this.#queue.length > 0) {
                const job = this.#queue.shift();
                this.#processJob(job);
            }
            await new Promise(r => setTimeout(r, 100));
        }

        this.#processing = false;
        console.log('✅ সব কমান্ড প্রসেস হয়েছে');
    }

    async #processJob(job) {
        this.#activeCount++;

        // delay handling
        if (job.delay > 0) {
            await new Promise(r => setTimeout(r, job.delay));
        }

        try {
            job.attempts++;
            console.log(`🔄 এক্সিকিউটিং (attempt ${job.attempts}/${job.maxRetries})...`);
            await job.command.execute();
            console.log('✅ সফল!');
        } catch (err) {
            console.log(`❌ ব্যর্থ: ${err.message}`);
            if (job.attempts < job.maxRetries) {
                job.delay = 1000 * (2 ** job.attempts); // exponential backoff
                this.#queue.push(job);
                console.log(`🔁 ${job.delay}ms পরে retry হবে`);
            } else {
                console.log('💀 সর্বোচ্চ retry শেষ');
            }
        } finally {
            this.#activeCount--;
        }
    }
}

// E-Commerce Order Command — বাংলাদেশ কনটেক্সট
class ProcessOrderCommand {
    #order;
    #processed = false;

    constructor(order) {
        this.#order = order;
    }

    async execute() {
        console.log(`📦 অর্ডার প্রসেসিং: #${this.#order.id} — ৳${this.#order.total}`);
        // সিমুলেটেড async কাজ
        await new Promise(r => setTimeout(r, 500));
        this.#processed = true;
        console.log(`✅ অর্ডার #${this.#order.id} shipped to ${this.#order.address}`);
    }

    async undo() {
        if (!this.#processed) return;
        console.log(`↩️ অর্ডার #${this.#order.id} বাতিল ও রিফান্ড`);
        this.#processed = false;
    }
}

// ব্যবহার
const queue = new AsyncCommandQueue(2); // ২টা কমান্ড একসাথে

queue.enqueue(new ProcessOrderCommand({
    id: 'BD-1001', total: 2500, address: 'ঢাকা, মিরপুর ১০'
}));

queue.enqueue(new ProcessOrderCommand({
    id: 'BD-1002', total: 850, address: 'চট্টগ্রাম, আগ্রাবাদ'
}));

queue.enqueue(new ProcessOrderCommand({
    id: 'BD-1003', total: 12000, address: 'সিলেট, জিন্দাবাজার'
}));

await queue.process();
```

---

## 🌍 Real-World Applicable Areas

### 1. Laravel Queue Jobs — এটাই Command Pattern!

Laravel-এর পুরো Queue সিস্টেমটাই Command প্যাটার্নের বাস্তব রূপ:

```php
// Laravel Job = ConcreteCommand
class SendWelcomeEmail implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(
        private readonly User $user
    ) {}

    // execute() মেথড — এখানে আসল কাজ হয়
    public function handle(Mailer $mailer): void
    {
        $mailer->to($this->user->email)->send(new WelcomeMail($this->user));
    }

    // ব্যর্থ হলে কী করবে
    public function failed(\Throwable $exception): void
    {
        Log::error("Welcome email failed for {$this->user->email}");
    }
}

// dispatch = Invoker-এর executeCommand()
SendWelcomeEmail::dispatch($user);
SendWelcomeEmail::dispatch($user)->delay(now()->addMinutes(5));
SendWelcomeEmail::dispatch($user)->onQueue('emails');
```

**ম্যাপিং:**
| Command Pattern | Laravel Queue |
|---|---|
| Command Interface | `ShouldQueue` |
| ConcreteCommand | Job ক্লাস |
| execute() | handle() |
| Invoker | `dispatch()` / `Bus::dispatch()` |
| Receiver | সার্ভিস যেটা handle()-এ inject হয় |
| Command Queue | Redis/SQS/Database queue |
| Worker | `php artisan queue:work` |

### 2. Laravel Artisan Commands

```php
// Artisan Command = Command Pattern
class ImportProducts extends \Illuminate\Console\Command
{
    protected $signature = 'products:import {file}';
    protected $description = 'CSV থেকে প্রোডাক্ট ইম্পোর্ট করো';

    public function handle(ProductImporter $importer): int
    {
        $file = $this->argument('file');
        $this->info("ইম্পোর্ট শুরু: {$file}");
        $importer->import($file);
        $this->info('✅ ইম্পোর্ট সম্পন্ন!');
        return self::SUCCESS;
    }
}
```

### 3. CQRS Command Bus

CQRS (Command Query Responsibility Segregation)-এ Command Bus হলো একটা বিশেষায়িত Command প্যাটার্ন যেখানে:
- **Command** = ইচ্ছা/অভিপ্রায় (Intent) প্রকাশ করে
- **Handler** = Receiver — আসল কাজ করে
- **Bus** = Invoker — Command কে সঠিক Handler-এ পাঠায়

### 4. Database Migration (up/down = execute/undo)

```php
// Migration = Command with undo!
class CreateUsersTable extends Migration
{
    public function up(): void { /* execute() */ }
    public function down(): void { /* undo() */ }
}
```

### 5. আরও ব্যবহারের জায়গা

| ক্ষেত্র | Command | Invoker | Receiver |
|---|---|---|---|
| GUI বাটন | ButtonClickCommand | Button | ViewController |
| Smart Home | TurnOnACCommand | VoiceAssistant | AC Device |
| Transaction | TransferMoneyCommand | BankApp | PaymentGateway |
| Task Scheduler | ScheduledTaskCommand | Cron/Scheduler | TaskRunner |
| Game | MoveCharacterCommand | InputHandler | GameCharacter |

---

## 🔥 Advanced Deep Dive

### 1. Command vs Strategy — তুলনা

এই দুটো প্যাটার্ন দেখতে অনেকটা একরকম — দুটোই একটা behavior কে অবজেক্টে encapsulate করে। কিন্তু উদ্দেশ্য সম্পূর্ণ আলাদা:

| দিক | Command | Strategy |
|---|---|---|
| **উদ্দেশ্য** | রিকোয়েস্ট encapsulate ও defer করা | অ্যালগরিদম বদলানো |
| **কখন** | কাজটা পরে হবে, কিউ হবে, আনডু হবে | কাজটা এখনই হবে, ভিন্ন উপায়ে |
| **State** | অবশ্যই state রাখে (undo-র জন্য) | সাধারণত stateless |
| **History** | হিস্ট্রি রাখা দরকার | দরকার নেই |
| **উদাহরণ** | "এই ফাইলটা ডিলিট করো" (কী করতে হবে) | "ফাইল zip করো বা encrypt করো" (কীভাবে করতে হবে) |

```php
// Strategy — কীভাবে sort করবে সেটা ঠিক করে
$sorter->setStrategy(new QuickSortStrategy());
$sorter->sort($data); // এখনই sort হবে

// Command — কী করতে হবে সেটা encapsulate করে
$queue->push(new SortDataCommand($data, 'quick'));
// পরে কোনো Worker এসে execute করবে
```

### 2. Command + Memento (Undo-র জন্য)

Memento প্যাটার্ন অবজেক্টের পুরো state সংরক্ষণ করে। Command-এর undo()-তে Memento ব্যবহার করলে যেকোনো জটিল state-ও আনডু করা যায়:

```php
<?php

declare(strict_types=1);

// Memento — state snapshot
class EditorMemento
{
    public function __construct(
        private readonly string $content,
        private readonly int $cursorPosition,
        private readonly array $formatting,
    ) {}

    public function getContent(): string { return $this->content; }
    public function getCursorPosition(): int { return $this->cursorPosition; }
    public function getFormatting(): array { return $this->formatting; }
}

// Receiver যেটা Memento সাপোর্ট করে
class RichTextEditor
{
    private string $content = '';
    private int $cursorPosition = 0;
    private array $formatting = [];

    public function type(string $text): void
    {
        $this->content .= $text;
        $this->cursorPosition += strlen($text);
    }

    public function bold(int $start, int $end): void
    {
        $this->formatting[] = ['type' => 'bold', 'start' => $start, 'end' => $end];
    }

    public function save(): EditorMemento
    {
        return new EditorMemento($this->content, $this->cursorPosition, $this->formatting);
    }

    public function restore(EditorMemento $memento): void
    {
        $this->content = $memento->getContent();
        $this->cursorPosition = $memento->getCursorPosition();
        $this->formatting = $memento->getFormatting();
    }

    public function getContent(): string { return $this->content; }
}

// Command যেটা Memento ব্যবহার করে undo-র জন্য
class TypeCommand implements EditorCommand
{
    private ?EditorMemento $backup = null;

    public function __construct(
        private readonly RichTextEditor $editor,
        private readonly string $text
    ) {}

    public function execute(): void
    {
        $this->backup = $this->editor->save(); // state সংরক্ষণ
        $this->editor->type($this->text);
    }

    public function undo(): void
    {
        if ($this->backup) {
            $this->editor->restore($this->backup); // Memento থেকে restore
        }
    }

    public function getDescription(): string
    {
        return "Type: '{$this->text}'";
    }
}
```

### 3. Command Bus Implementation

একটা পূর্ণাঙ্গ Command Bus যেটা middleware সাপোর্ট করে — CQRS আর্কিটেকচারের core component:

#### PHP 8.3

```php
<?php

declare(strict_types=1);

// Command — শুধু ডেটা ধারণ করে, কোনো লজিক নেই
interface BusCommand {}

interface CommandHandler
{
    public function handle(BusCommand $command): mixed;
}

// Middleware — command execute হওয়ার আগে/পরে কাজ
interface CommandMiddleware
{
    public function handle(BusCommand $command, \Closure $next): mixed;
}

// Logging Middleware
class LoggingMiddleware implements CommandMiddleware
{
    public function handle(BusCommand $command, \Closure $next): mixed
    {
        $name = $command::class;
        echo "📝 [{$name}] শুরু হচ্ছে...\n";
        $startTime = microtime(true);

        $result = $next($command);

        $elapsed = round((microtime(true) - $startTime) * 1000, 2);
        echo "📝 [{$name}] সম্পন্ন ({$elapsed}ms)\n";

        return $result;
    }
}

// Transaction Middleware — ডাটাবেস ট্রানজ্যাকশন
class TransactionMiddleware implements CommandMiddleware
{
    public function handle(BusCommand $command, \Closure $next): mixed
    {
        echo "🔒 Transaction শুরু\n";
        try {
            $result = $next($command);
            echo "✅ Transaction commit\n";
            return $result;
        } catch (\Throwable $e) {
            echo "↩️ Transaction rollback\n";
            throw $e;
        }
    }
}

// Validation Middleware
class ValidationMiddleware implements CommandMiddleware
{
    public function handle(BusCommand $command, \Closure $next): mixed
    {
        // Reflection দিয়ে validation attribute চেক
        $reflection = new \ReflectionClass($command);
        foreach ($reflection->getProperties() as $prop) {
            $value = $prop->getValue($command);
            if ($value === null || $value === '') {
                throw new \InvalidArgumentException(
                    "'{$prop->getName()}' খালি রাখা যাবে না"
                );
            }
        }
        return $next($command);
    }
}

// Command Bus
class CommandBus
{
    /** @var array<class-string<BusCommand>, class-string<CommandHandler>> */
    private array $handlerMap = [];

    /** @var CommandMiddleware[] */
    private array $middlewares = [];

    public function registerHandler(string $commandClass, CommandHandler $handler): void
    {
        $this->handlerMap[$commandClass] = $handler;
    }

    public function addMiddleware(CommandMiddleware $middleware): void
    {
        $this->middlewares[] = $middleware;
    }

    public function dispatch(BusCommand $command): mixed
    {
        $handler = $this->resolveHandler($command);

        // Middleware chain তৈরি করো (onion architecture)
        $pipeline = array_reduce(
            array_reverse($this->middlewares),
            fn(\Closure $next, CommandMiddleware $middleware) =>
                fn(BusCommand $cmd) => $middleware->handle($cmd, $next),
            fn(BusCommand $cmd) => $handler->handle($cmd)
        );

        return $pipeline($command);
    }

    private function resolveHandler(BusCommand $command): CommandHandler
    {
        $class = $command::class;
        if (!isset($this->handlerMap[$class])) {
            throw new \RuntimeException("{$class} এর জন্য কোনো Handler নেই");
        }
        return $this->handlerMap[$class];
    }
}

// Concrete Command & Handler
class CreateOrderCommand implements BusCommand
{
    public function __construct(
        public readonly string $customerName,
        public readonly string $product,
        public readonly float $amount,
        public readonly string $address,
    ) {}
}

class CreateOrderHandler implements CommandHandler
{
    public function handle(BusCommand $command): mixed
    {
        assert($command instanceof CreateOrderCommand);
        echo "📦 অর্ডার তৈরি হচ্ছে:\n";
        echo "   গ্রাহক: {$command->customerName}\n";
        echo "   পণ্য: {$command->product}\n";
        echo "   মূল্য: ৳{$command->amount}\n";
        echo "   ঠিকানা: {$command->address}\n";

        return ['orderId' => 'ORD-' . random_int(1000, 9999)];
    }
}

// ব্যবহার
$bus = new CommandBus();
$bus->addMiddleware(new LoggingMiddleware());
$bus->addMiddleware(new TransactionMiddleware());
$bus->addMiddleware(new ValidationMiddleware());
$bus->registerHandler(CreateOrderCommand::class, new CreateOrderHandler());

$result = $bus->dispatch(new CreateOrderCommand(
    customerName: 'রহিম উদ্দিন',
    product: 'Samsung Galaxy S24',
    amount: 115000.00,
    address: 'ঢাকা, গুলশান ২',
));

echo "🎫 Order ID: {$result['orderId']}\n";
```

#### JavaScript (ES2022+)

```javascript
// Command Bus with Middleware
class CommandBus {
    #handlers = new Map();
    #middlewares = [];

    register(commandName, handler) {
        this.#handlers.set(commandName, handler);
    }

    use(middleware) {
        this.#middlewares.push(middleware);
    }

    async dispatch(command) {
        const handler = this.#handlers.get(command.constructor.name);
        if (!handler) {
            throw new Error(`${command.constructor.name} এর handler নেই`);
        }

        // Middleware chain — onion architecture
        const pipeline = this.#middlewares.reduceRight(
            (next, middleware) => (cmd) => middleware(cmd, next),
            (cmd) => handler(cmd)
        );

        return pipeline(command);
    }
}

// Middlewares
const loggingMiddleware = async (command, next) => {
    const name = command.constructor.name;
    console.log(`📝 [${name}] শুরু...`);
    const start = performance.now();
    const result = await next(command);
    console.log(`📝 [${name}] শেষ (${(performance.now() - start).toFixed(2)}ms)`);
    return result;
};

const validationMiddleware = async (command, next) => {
    for (const [key, value] of Object.entries(command)) {
        if (value === null || value === undefined || value === '') {
            throw new Error(`'${key}' খালি রাখা যাবে না`);
        }
    }
    return next(command);
};

// Command & Handler
class PlaceOrderCommand {
    constructor(customer, product, amount) {
        this.customer = customer;
        this.product = product;
        this.amount = amount;
    }
}

const placeOrderHandler = async (command) => {
    console.log(`📦 অর্ডার: ${command.product} — ৳${command.amount}`);
    return { orderId: `ORD-${Math.random().toString(36).slice(2, 8)}` };
};

// ব্যবহার
const bus = new CommandBus();
bus.use(loggingMiddleware);
bus.use(validationMiddleware);
bus.register('PlaceOrderCommand', placeOrderHandler);

const result = await bus.dispatch(
    new PlaceOrderCommand('করিম সাহেব', 'iPhone 15 Pro', 185000)
);
console.log(`🎫 Order: ${result.orderId}`);
```

### 4. Command Serialization for Distributed Systems

ডিস্ট্রিবিউটেড সিস্টেমে Command অবজেক্ট সিরিয়ালাইজ করে এক সার্ভার থেকে অন্য সার্ভারে পাঠানো হয়:

```php
<?php

declare(strict_types=1);

// Serializable Command — JSON format-এ serialize/deserialize
class SerializableCommandBus
{
    /** @var array<string, \Closure> */
    private array $deserializers = [];

    /** @var array<string, CommandHandler> */
    private array $handlers = [];

    public function register(
        string $commandType,
        CommandHandler $handler,
        \Closure $deserializer
    ): void {
        $this->handlers[$commandType] = $handler;
        $this->deserializers[$commandType] = $deserializer;
    }

    // কমান্ড সিরিয়ালাইজ করো — Redis/RabbitMQ তে পাঠানোর জন্য
    public function serialize(BusCommand $command): string
    {
        return json_encode([
            'type' => $command::class,
            'payload' => get_object_vars($command),
            'metadata' => [
                'id' => bin2hex(random_bytes(16)),
                'timestamp' => (new \DateTimeImmutable())->format('c'),
                'correlationId' => bin2hex(random_bytes(8)),
            ],
        ], JSON_THROW_ON_ERROR);
    }

    // সিরিয়ালাইজড কমান্ড থেকে অবজেক্ট তৈরি করো
    public function deserializeAndDispatch(string $json): mixed
    {
        $data = json_decode($json, true, 512, JSON_THROW_ON_ERROR);
        $type = $data['type'];

        if (!isset($this->deserializers[$type])) {
            throw new \RuntimeException("Unknown command type: {$type}");
        }

        $command = ($this->deserializers[$type])($data['payload']);
        return $this->handlers[$type]->handle($command);
    }
}
```

### 5. Transactional Command — All or Nothing

```php
<?php

declare(strict_types=1);

// Saga Pattern — ডিস্ট্রিবিউটেড ট্রানজ্যাকশন Command দিয়ে
class OrderSaga
{
    /** @var array{command: Command, compensation: Command}[] */
    private array $steps = [];
    private array $executedSteps = [];

    public function addStep(Command $command, Command $compensation): void
    {
        $this->steps[] = [
            'command' => $command,
            'compensation' => $compensation,
        ];
    }

    public function execute(): bool
    {
        $this->executedSteps = [];

        foreach ($this->steps as $step) {
            try {
                $step['command']->execute();
                $this->executedSteps[] = $step;
            } catch (\Throwable $e) {
                echo "❌ Step ব্যর্থ: {$e->getMessage()}\n";
                echo "🔄 Compensation শুরু হচ্ছে...\n";
                $this->compensate();
                return false;
            }
        }

        return true;
    }

    private function compensate(): void
    {
        // উল্টো ক্রমে compensation চালাও
        foreach (array_reverse($this->executedSteps) as $step) {
            try {
                $step['compensation']->execute();
                echo "↩️ Compensation সফল\n";
            } catch (\Throwable $e) {
                // Compensation ব্যর্থ = ম্যানুয়াল intervention দরকার
                echo "🚨 Compensation ব্যর্থ! Manual fix দরকার: {$e->getMessage()}\n";
            }
        }
    }
}
```

### 6. Laravel Job Dispatch Internals

Laravel অভ্যন্তরীণভাবে কীভাবে Command প্যাটার্ন ব্যবহার করে:

```
dispatch(new SendEmail($user))
    ↓
Dispatcher::dispatch()          // Invoker
    ↓
PendingDispatch created         // Command wrapper
    ↓
Job serialized → JSON           // Serialization
    ↓
Pushed to Redis/SQS queue       // Command Queue
    ↓
Worker picks up job             // Worker (another Invoker)
    ↓
Job::handle() called            // execute()
    ↓
ব্যর্থ? → Job::failed()        // error handling
    ↓
retry? → কিউতে ফেরত            // retry logic
```

---

## ✅ Pros (সুবিধা)

1. **Single Responsibility Principle (SRP):** Invoker আর Receiver decouple হয়ে যায়
2. **Open/Closed Principle (OCP):** নতুন Command যোগ করলে পুরানো কোড পরিবর্তন লাগে না
3. **Undo/Redo:** স্বাভাবিকভাবেই সাপোর্ট করা যায়
4. **Deferred Execution:** কমান্ড কিউতে রাখো, পরে execute করো
5. **Logging & Auditing:** প্রতিটা Command লগ করা সহজ
6. **Macro Commands:** ছোট ছোট কমান্ড দিয়ে জটিল অপারেশন তৈরি
7. **Transactional:** ব্যর্থ হলে rollback করা সহজ
8. **Testing:** প্রতিটা Command আলাদাভাবে টেস্ট করা যায়
9. **Serialization:** কমান্ড serialize করে নেটওয়ার্কে পাঠানো যায়

## ❌ Cons (অসুবিধা)

1. **ক্লাস বিস্ফোরণ:** প্রতিটা অ্যাকশনের জন্য আলাদা ক্লাস — ছোট প্রজেক্টে অতিরিক্ত
2. **জটিলতা বৃদ্ধি:** সিম্পল মেথড কলের বদলে Command অবজেক্ট তৈরি — overhead
3. **State Management:** Undo-র জন্য পুরানো state সংরক্ষণ করতে হয় — মেমরি ব্যবহার বাড়ে
4. **Debugging কঠিন:** Invoker → Command → Receiver — কোডের flow ট্রেস করা কঠিন হতে পারে
5. **Over-engineering:** সব জায়গায় Command প্যাটার্ন লাগে না — সিম্পল CRUD-এ এটা overkill

---

## ⚠️ Common Mistakes (সাধারণ ভুল)

### ❌ ভুল ১: Command-এ বিজনেস লজিক রাখা

```php
// ❌ ভুল — Command নিজেই কাজ করছে
class CreateUserCommand implements Command
{
    public function execute(): void
    {
        // এখানে সরাসরি DB query, validation, email পাঠানো সব!
        $user = DB::table('users')->insert([...]);
        Mail::send(new WelcomeMail($user));
        Cache::forget('users_count');
    }
}

// ✅ সঠিক — Command শুধু Receiver-কে delegate করে
class CreateUserCommand implements Command
{
    public function __construct(
        private readonly UserService $userService,
        private readonly CreateUserDTO $dto,
    ) {}

    public function execute(): void
    {
        $this->userService->create($this->dto);
    }
}
```

### ❌ ভুল ২: Undo state সংরক্ষণ না করা

```php
// ❌ ভুল — undo-র জন্য আগের state নেই
class UpdatePriceCommand implements Command
{
    public function execute(): void
    {
        $this->product->setPrice($this->newPrice);
    }

    public function undo(): void
    {
        // আগের দাম কত ছিল? জানি না! 😱
        $this->product->setPrice(???);
    }
}

// ✅ সঠিক — আগের state সংরক্ষণ করো
class UpdatePriceCommand implements Command
{
    private float $previousPrice;

    public function execute(): void
    {
        $this->previousPrice = $this->product->getPrice(); // আগের দাম সেভ
        $this->product->setPrice($this->newPrice);
    }

    public function undo(): void
    {
        $this->product->setPrice($this->previousPrice); // আগের দাম ফেরত
    }
}
```

### ❌ ভুল ৩: God Command — একটা কমান্ডে অনেক কাজ

```php
// ❌ ভুল — একটা কমান্ড সব করছে
class ProcessEverythingCommand implements Command
{
    public function execute(): void
    {
        $this->validateOrder();
        $this->chargePayment();
        $this->updateInventory();
        $this->sendNotification();
        $this->generateInvoice();
    }
}

// ✅ সঠিক — ছোট ছোট কমান্ড দিয়ে MacroCommand তৈরি করো
$processOrder = new TransactionalMacroCommand(
    new ValidateOrderCommand($order),
    new ChargePaymentCommand($payment),
    new UpdateInventoryCommand($items),
    new SendNotificationCommand($user),
    new GenerateInvoiceCommand($order),
);
```

### ❌ ভুল ৪: Command reuse না করে execute দুইবার কল করা

```php
// ❌ ভুল — একই কমান্ড দুইবার execute করলে state গোলমাল হবে
$cmd = new InsertTextCommand($doc, 'hello', 0);
$cmd->execute(); // ঠিক আছে
$cmd->execute(); // 😱 আবার hello insert হবে, undo গোলমাল হবে

// ✅ সঠিক — প্রতিবারের জন্য নতুন Command তৈরি করো
$history->executeCommand(new InsertTextCommand($doc, 'hello', 0));
$history->executeCommand(new InsertTextCommand($doc, 'world', 6));
```

### ❌ ভুল ৫: সিম্পল কাজে Command প্যাটার্ন ব্যবহার

```php
// ❌ Over-engineering — সিম্পল getter-এ Command লাগে না
class GetUserNameCommand implements Command
{
    public function execute(): void { return $this->user->getName(); }
}

// ✅ সরাসরি কল করো
$name = $user->getName();
```

---

## 🧪 টেস্টিং (Testing)

### PHPUnit

```php
<?php

declare(strict_types=1);

use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\Attributes\Test;
use PHPUnit\Framework\Attributes\DataProvider;

class CommandPatternTest extends TestCase
{
    private TextDocument $document;
    private CommandHistory $history;

    protected function setUp(): void
    {
        $this->document = new TextDocument();
        $this->history = new CommandHistory();
    }

    #[Test]
    public function insert_command_adds_text_at_position(): void
    {
        $command = new InsertTextCommand($this->document, 'Hello', 0);
        $this->history->executeCommand($command);

        $this->assertSame('Hello', $this->document->getContent());
    }

    #[Test]
    public function undo_reverses_last_command(): void
    {
        $this->history->executeCommand(
            new InsertTextCommand($this->document, 'Hello', 0)
        );
        $this->history->executeCommand(
            new InsertTextCommand($this->document, ' World', 5)
        );

        $this->assertSame('Hello World', $this->document->getContent());

        $this->history->undo();
        $this->assertSame('Hello', $this->document->getContent());
    }

    #[Test]
    public function redo_reapplies_undone_command(): void
    {
        $this->history->executeCommand(
            new InsertTextCommand($this->document, 'Test', 0)
        );

        $this->history->undo();
        $this->assertSame('', $this->document->getContent());

        $this->history->redo();
        $this->assertSame('Test', $this->document->getContent());
    }

    #[Test]
    public function new_command_clears_redo_stack(): void
    {
        $this->history->executeCommand(
            new InsertTextCommand($this->document, 'A', 0)
        );
        $this->history->undo();

        // নতুন কমান্ড — redo stack ক্লিয়ার হবে
        $this->history->executeCommand(
            new InsertTextCommand($this->document, 'B', 0)
        );

        $this->assertFalse($this->history->canRedo());
    }

    #[Test]
    public function multiple_undo_redo_maintains_consistency(): void
    {
        $commands = ['Hello', ' ', 'World', '!'];

        foreach ($commands as $i => $text) {
            $position = strlen($this->document->getContent());
            $this->history->executeCommand(
                new InsertTextCommand($this->document, $text, $position)
            );
        }

        $this->assertSame('Hello World!', $this->document->getContent());

        // ৩ বার undo
        $this->history->undo();
        $this->history->undo();
        $this->history->undo();
        $this->assertSame('Hello', $this->document->getContent());

        // ২ বার redo
        $this->history->redo();
        $this->history->redo();
        $this->assertSame('Hello World', $this->document->getContent());
    }

    #[Test]
    public function delete_command_undo_restores_text(): void
    {
        $this->history->executeCommand(
            new InsertTextCommand($this->document, 'Hello World', 0)
        );

        $this->history->executeCommand(
            new DeleteTextCommand($this->document, 5, 6) // " World" ডিলিট
        );

        $this->assertSame('Hello', $this->document->getContent());

        $this->history->undo(); // ডিলিট আনডু
        $this->assertSame('Hello World', $this->document->getContent());
    }

    #[Test]
    public function macro_command_executes_all_subcommands(): void
    {
        $light1 = new Light('Room 1');
        $light2 = new Light('Room 2');

        $macro = new MacroCommand(
            new LightOnCommand($light1),
            new LightOnCommand($light2),
        );

        $macro->execute();
        $this->assertSame(100, $light1->getBrightness());
        $this->assertSame(100, $light2->getBrightness());

        $macro->undo();
        $this->assertSame(0, $light1->getBrightness());
        $this->assertSame(0, $light2->getBrightness());
    }

    #[Test]
    public function command_bus_dispatches_to_correct_handler(): void
    {
        $bus = new CommandBus();
        $handler = $this->createMock(CommandHandler::class);

        $command = new CreateOrderCommand(
            customerName: 'Test',
            product: 'Phone',
            amount: 100,
            address: 'Dhaka',
        );

        $handler->expects($this->once())
            ->method('handle')
            ->with($command)
            ->willReturn(['orderId' => 'ORD-1234']);

        $bus->registerHandler(CreateOrderCommand::class, $handler);
        $result = $bus->dispatch($command);

        $this->assertSame('ORD-1234', $result['orderId']);
    }

    #[Test]
    public function transactional_macro_rolls_back_on_failure(): void
    {
        $light = new Light('Test');
        $failCommand = new class implements Command {
            public function execute(): void {
                throw new \RuntimeException('ইচ্ছাকৃত ব্যর্থতা');
            }
            public function undo(): void {}
        };

        $macro = new TransactionalMacroCommand(
            new LightOnCommand($light),
            $failCommand,
        );

        $this->expectException(\RuntimeException::class);
        $macro->execute();

        // Rollback হওয়ার কারণে লাইট বন্ধ থাকবে
        $this->assertSame(0, $light->getBrightness());
    }
}
```

### Jest (JavaScript)

```javascript
describe('Command Pattern', () => {
    let doc;
    let history;

    beforeEach(() => {
        doc = new TextDocument();
        history = new CommandHistory();
    });

    test('InsertTextCommand টেক্সট ঢোকায়', () => {
        history.execute(new InsertTextCommand(doc, 'Hello', 0));
        expect(doc.content).toBe('Hello');
    });

    test('undo সর্বশেষ কমান্ড উল্টায়', () => {
        history.execute(new InsertTextCommand(doc, 'Hello', 0));
        history.execute(new InsertTextCommand(doc, ' World', 5));

        history.undo();
        expect(doc.content).toBe('Hello');
    });

    test('redo আনডু করা কমান্ড আবার চালায়', () => {
        history.execute(new InsertTextCommand(doc, 'Test', 0));
        history.undo();
        history.redo();

        expect(doc.content).toBe('Test');
    });

    test('নতুন কমান্ড redo stack ক্লিয়ার করে', () => {
        history.execute(new InsertTextCommand(doc, 'A', 0));
        history.undo();
        history.execute(new InsertTextCommand(doc, 'B', 0));

        // redo করার কিছু নেই — spy দিয়ে চেক
        const spy = jest.spyOn(console, 'log');
        history.redo();
        expect(spy).toHaveBeenCalledWith(expect.stringContaining('রিডু করার কিছু নেই'));
        spy.mockRestore();
    });

    test('DeleteTextCommand undo টেক্সট ফেরত আনে', () => {
        history.execute(new InsertTextCommand(doc, 'Hello World', 0));
        history.execute(new DeleteTextCommand(doc, 5, 6));

        expect(doc.content).toBe('Hello');

        history.undo();
        expect(doc.content).toBe('Hello World');
    });

    test('MacroCommand সব sub-command execute করে', () => {
        const light1 = new Light('Room 1');
        const light2 = new Light('Room 2');

        const macro = new MacroCommand(
            new LightOnCommand(light1),
            new LightOnCommand(light2),
        );

        macro.execute();
        expect(light1.brightness).toBe(100);
        expect(light2.brightness).toBe(100);

        macro.undo();
        expect(light1.brightness).toBe(0);
        expect(light2.brightness).toBe(0);
    });

    test('CommandBus সঠিক handler-এ dispatch করে', async () => {
        const bus = new CommandBus();
        const mockHandler = jest.fn().mockResolvedValue({ id: 42 });

        bus.register('TestCommand', mockHandler);

        class TestCommand { data = 'test'; }
        const result = await bus.dispatch(new TestCommand());

        expect(mockHandler).toHaveBeenCalledTimes(1);
        expect(result).toEqual({ id: 42 });
    });

    test('AsyncCommandQueue retry logic কাজ করে', async () => {
        const queue = new AsyncCommandQueue(1);
        let attempts = 0;

        const failingCommand = {
            async execute() {
                attempts++;
                if (attempts < 3) throw new Error('ব্যর্থ');
                // তৃতীয় বারে সফল
            }
        };

        queue.enqueue(failingCommand, { maxRetries: 3 });
        await queue.process();

        expect(attempts).toBe(3);
    });
});
```

---

## 🔗 সম্পর্কিত প্যাটার্ন

### Command ↔ Chain of Responsibility

```
Chain of Responsibility: রিকোয়েস্ট handlers-দের চেইনে পাঠায়
                         → কে handle করবে সেটা runtime-এ ঠিক হয়

Command:                 রিকোয়েস্ট = অবজেক্ট, sender-receiver decouple
                         → কে handle করবে সেটা আগেই ঠিক

কম্বিনেশন: Command Bus-এর middleware = Chain of Responsibility!
           Command → Middleware1 → Middleware2 → Handler
```

### Command ↔ Memento

```
Memento: অবজেক্টের state snapshot সংরক্ষণ করে
Command: অপারেশন encapsulate করে, undo সাপোর্ট করে

কম্বিনেশন: Command-এর undo() মেথডে Memento ব্যবহার করলে
           জটিল state-ও নিখুঁতভাবে restore করা যায়
```

### Command ↔ Observer

```
Observer: কোনো ইভেন্ট হলে সব subscriber-কে notify করে
Command: ইভেন্টের response-এ কী করতে হবে সেটা encapsulate করে

কম্বিনেশন: Observer ইভেন্ট পেলে → Command execute করে
           Event → EventHandler (Command) → Receiver
```

### Command ↔ Strategy

```
Strategy: একটা কাজ বিভিন্ন উপায়ে করার জন্য (HOW)
Command: একটা কাজ encapsulate করার জন্য (WHAT)

Strategy = stateless algorithm swap
Command = stateful operation + undo + queue + log
```

### Command ↔ Composite

```
Composite: অবজেক্টকে tree structure-এ organize করে
Command + Composite = MacroCommand

MacroCommand = একাধিক Command-এর Composite
             = একটা কমান্ড execute করলে সব child execute হয়
```

---

## 📏 কখন ব্যবহার করবেন / করবেন না

### ✅ ব্যবহার করবেন যখন:

1. **Undo/Redo দরকার** — টেক্সট এডিটর, গ্রাফিক্স সফটওয়্যার, IDE
2. **অপারেশন কিউ করতে হবে** — Job queue, task scheduling, background processing
3. **অপারেশন লগ করতে হবে** — Audit trail, command history, debugging
4. **Transaction সাপোর্ট দরকার** — ব্যর্থ হলে rollback, saga pattern
5. **Deferred execution** — এখন request নাও, পরে execute করো
6. **Remote execution** — কমান্ড serialize করে নেটওয়ার্কে পাঠাও
7. **Macro/Composite operations** — ছোট কমান্ড দিয়ে বড় কাজ
8. **Sender-Receiver decouple করতে হবে** — GUI buttons, API endpoints

### ❌ ব্যবহার করবেন না যখন:

1. **সিম্পল method call** — শুধু একটা মেথড কল করতে Command ক্লাস বানানো overkill
2. **Undo দরকার নেই, queue দরকার নেই** — শুধু decouple করতে চাইলে Strategy/Observer ভালো
3. **CRUD অপারেশন** — সিম্পল create/read/update/delete-এ Command pattern অতিরিক্ত জটিলতা
4. **Performance-critical code** — অবজেক্ট তৈরির overhead সমস্যা হতে পারে
5. **ছোট প্রজেক্ট** — ১০-১৫টা endpoint-এর API-তে Command Bus overkill

### সিদ্ধান্ত নেওয়ার ফ্লোচার্ট:

```
রিকোয়েস্ট defer/queue করতে হবে?
    ├── হ্যাঁ → Command ✅
    └── না →
         Undo/Redo দরকার?
         ├── হ্যাঁ → Command ✅
         └── না →
              Transaction/Rollback দরকার?
              ├── হ্যাঁ → Command ✅
              └── না →
                   অপারেশন লগ করতে হবে?
                   ├── হ্যাঁ → Command বিবেচনা করো
                   └── না → সরাসরি method call করো ✅
```

---

## 📋 সারসংক্ষেপ (Summary)

### মূল পয়েন্ট

| বিষয় | বিবরণ |
|---|---|
| **ধরন** | Behavioral Design Pattern |
| **উদ্দেশ্য** | রিকোয়েস্টকে অবজেক্ট হিসেবে encapsulate করা |
| **মূল শক্তি** | Undo/Redo, Queue, Logging, Decouple sender-receiver |
| **কম্পোনেন্ট** | Command, ConcreteCommand, Invoker, Receiver |
| **Laravel-এ** | Jobs, Artisan Commands, Bus, Migrations |
| **JS-এ** | Redux actions, Event handlers, Task runners |
| **সাবধানতা** | Class explosion, over-engineering, state management |

### এক নজরে Command প্যাটার্ন

```
┌──────────────────────────────────────────────────────┐
│                  Command Pattern                      │
│                                                      │
│  🎯 "রিকোয়েস্টকে অবজেক্ট বানাও"                    │
│                                                      │
│  📦 Encapsulate → 🕐 Defer → ↩️ Undo → 📋 Log         │
│                                                      │
│  Invoker ──▶ Command ──▶ Receiver                    │
│  (কে বলে)   (কী করতে হবে) (কে করে)                  │
│                                                      │
│  বাংলাদেশে: bKash Send Money, Daraz Order Queue,     │
│            pathao Ride Request, SSLCommerz Payment    │
└──────────────────────────────────────────────────────┘
```

### মনে রাখার সূত্র

> **"অর্ডার স্লিপ" ভাবো** — রেস্টুরেন্টে তুমি ওয়েটারকে অর্ডার দাও (Invoker), ওয়েটার একটা স্লিপে লেখে (Command), শেফ সেটা দেখে রান্না করে (Receiver)। স্লিপটা হারিয়ে গেলে? আনডু। কিচেন ব্যস্ত? কিউতে রাখো। কী অর্ডার হয়েছিল? লগ দেখো।

---

> **📝 রেফারেন্স:**
> - *Design Patterns: Elements of Reusable Object-Oriented Software* — GoF (1994)
> - *Head First Design Patterns* — O'Reilly
> - *Laravel Documentation* — Queues & Jobs
> - *Patterns of Enterprise Application Architecture* — Martin Fowler
