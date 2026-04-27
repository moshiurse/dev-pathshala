# 💾 Memento প্যাটার্ন

## 📌 সংজ্ঞা

**Memento** হলো একটি **Behavioral Design Pattern** যা কোনো অবজেক্টের অভ্যন্তরীণ অবস্থা (internal state) ক্যাপচার করে বাইরে সংরক্ষণ করে, যাতে পরবর্তীতে সেই অবস্থায় ফিরে যাওয়া যায় — **encapsulation ভঙ্গ না করে**।

> **GoF Definition:** *"Without violating encapsulation, capture and externalize an object's internal state so that the object can be restored to this state later."*

### মূল ধারণা

Memento প্যাটার্নের মূল উদ্দেশ্য হলো **snapshot** তৈরি করা। আপনি যখন কোনো ডকুমেন্ট এডিট করেন, প্রতিটি পরিবর্তনের আগে একটি snapshot সংরক্ষণ করা হয়। পরবর্তীতে `Ctrl+Z` চাপলে আগের snapshot থেকে state পুনরুদ্ধার হয়।

### তিনটি মূল অংশগ্রহণকারী (Participants)

| অংশগ্রহণকারী | দায়িত্ব |
|---|---|
| **Originator** | যে অবজেক্টের state সংরক্ষণ করা হবে। এটি নিজেই memento তৈরি করে এবং memento থেকে নিজেকে পুনরুদ্ধার করে। |
| **Memento** | Originator-এর internal state-এর snapshot ধরে রাখে। বাইরের কেউ এই state পরিবর্তন করতে পারে না। |
| **Caretaker** | Memento সংরক্ষণ ও পরিচালনা করে, কিন্তু memento-র ভেতরের state দেখতে বা পরিবর্তন করতে পারে না। |

### কেন Encapsulation গুরুত্বপূর্ণ?

সরাসরি অবজেক্টের internal state expose করলে tight coupling তৈরি হয়। Memento প্যাটার্ন এই সমস্যা সমাধান করে — Caretaker শুধু memento অবজেক্ট ধরে রাখে, কিন্তু ভেতরের ডেটা সম্পর্কে কিছু জানে না। এটি **Information Hiding** এবং **Single Responsibility Principle** উভয়ই মেনে চলে।

---

## 🏠 বাস্তব উদাহরণ

### 🎮 গেম সেভ/লোড সিস্টেম

ধরুন আপনি একটি RPG গেম খেলছেন। বসের সাথে ফাইটের আগে আপনি গেম সেভ করলেন:

```
📍 Save Point:
   - Level: 15
   - Health: 100
   - Position: Boss Room Entrance
   - Inventory: [Sword, Shield, Potion x3]

😵 Boss Fight — আপনি হেরে গেলেন!

🔄 Load Save:
   - সব কিছু আগের অবস্থায় ফিরে গেলো!
```

এখানে:
- **Originator** = Game Character (যার state সংরক্ষণ হচ্ছে)
- **Memento** = Save File (state-এর snapshot)
- **Caretaker** = Save System (save file গুলো ম্যানেজ করে)

### 🛒 বাংলাদেশী ই-কমার্স কার্ট

দারাজ বা চালডাল-এ শপিং করার সময়:

```
🛒 কার্টে আইটেম: [চাল ৫কেজি, ডাল ১কেজি, তেল ১লি]
💾 কার্ট State সেভ (ব্রাউজার বন্ধ করার আগে)

পরের দিন ব্রাউজার খুললেন:
🔄 কার্ট পুনরুদ্ধার — সব আইটেম যেমন ছিলো তেমনই আছে!
```

### 📝 ডকুমেন্ট ড্রাফট সিস্টেম

বাংলায় ব্লগ লেখার সময় প্রতিটি ড্রাফট একটি memento:

```
Draft 1: "আজকের বিষয় হলো..."
Draft 2: "আজকের বিষয় হলো ডিজাইন প্যাটার্ন..."
Draft 3: "আজকের বিষয় হলো ডিজাইন প্যাটার্ন এবং এর ব্যবহার..."

Undo → Draft 2 এ ফিরে যাও
Undo → Draft 1 এ ফিরে যাও
Redo → Draft 2 এ আবার যাও
```

---

## 📊 UML ডায়াগ্রাম

### ক্লাস ডায়াগ্রাম (ASCII)

```
┌─────────────────────┐         ┌─────────────────────┐
│     Caretaker        │         │     Originator       │
│─────────────────────│         │─────────────────────│
│ - history: Memento[] │────────▶│ - state: mixed       │
│─────────────────────│         │─────────────────────│
│ + backup(): void     │         │ + save(): Memento    │
│ + undo(): void       │         │ + restore(m): void   │
│ + redo(): void       │         │ + setState(s): void  │
│ + getHistory(): list │         │ + getState(): mixed  │
└─────────────────────┘         └──────────┬──────────┘
                                            │ creates
                                            ▼
                                ┌─────────────────────┐
                                │      Memento         │
                                │─────────────────────│
                                │ - state: mixed       │
                                │ - timestamp: Date    │
                                │─────────────────────│
                                │ + getState(): mixed  │
                                │ + getTimestamp(): Date│
                                │ + getName(): string  │
                                └─────────────────────┘
```

### সিকোয়েন্স ডায়াগ্রাম (ASCII)

```
  Caretaker          Originator           Memento
     │                    │                   │
     │  backup()          │                   │
     │───────────────────▶│                   │
     │                    │  new Memento()    │
     │                    │──────────────────▶│
     │                    │  return memento   │
     │◀───────────────────│                   │
     │  store(memento)    │                   │
     │                    │                   │
     │  undo()            │                   │
     │  get last memento  │                   │
     │───────────────────▶│                   │
     │                    │  getState()       │
     │                    │──────────────────▶│
     │                    │  return state     │
     │                    │◀──────────────────│
     │                    │  restore state    │
     │  done              │                   │
     │◀───────────────────│                   │
```

---

## 💻 ইমপ্লিমেন্টেশন

### 1️⃣ Basic Memento

#### PHP 8.3

```php
<?php

declare(strict_types=1);

/**
 * Memento: Originator-এর state-এর immutable snapshot
 */
final readonly class Memento
{
    public function __construct(
        private string $state,
        private DateTimeImmutable $timestamp = new DateTimeImmutable(),
    ) {}

    public function getState(): string
    {
        return $this->state;
    }

    public function getTimestamp(): DateTimeImmutable
    {
        return $this->timestamp;
    }

    public function getName(): string
    {
        return sprintf(
            '%s @ %s',
            substr($this->state, 0, 20),
            $this->timestamp->format('H:i:s')
        );
    }
}

/**
 * Originator: যে অবজেক্টের state সেভ/রিস্টোর হবে
 */
class Originator
{
    public function __construct(
        private string $state = '',
    ) {}

    public function setState(string $state): void
    {
        $this->state = $state;
    }

    public function getState(): string
    {
        return $this->state;
    }

    public function save(): Memento
    {
        return new Memento($this->state);
    }

    public function restore(Memento $memento): void
    {
        $this->state = $memento->getState();
    }
}

/**
 * Caretaker: Memento ম্যানেজ করে, কিন্তু ভেতরের state জানে না
 */
class Caretaker
{
    /** @var Memento[] */
    private array $undoStack = [];

    /** @var Memento[] */
    private array $redoStack = [];

    public function __construct(
        private readonly Originator $originator,
    ) {}

    public function backup(): void
    {
        $this->undoStack[] = $this->originator->save();
        $this->redoStack = []; // নতুন backup-এ redo history মুছে যায়
    }

    public function undo(): void
    {
        if (empty($this->undoStack)) {
            throw new UnderflowException('Undo stack খালি!');
        }

        // বর্তমান state redo stack-এ রাখো
        $this->redoStack[] = $this->originator->save();

        $memento = array_pop($this->undoStack);
        $this->originator->restore($memento);
    }

    public function redo(): void
    {
        if (empty($this->redoStack)) {
            throw new UnderflowException('Redo stack খালি!');
        }

        $this->undoStack[] = $this->originator->save();

        $memento = array_pop($this->redoStack);
        $this->originator->restore($memento);
    }

    /** @return string[] */
    public function getHistory(): array
    {
        return array_map(
            fn(Memento $m) => $m->getName(),
            $this->undoStack
        );
    }
}

// ব্যবহার
$originator = new Originator();
$caretaker = new Caretaker($originator);

$originator->setState('State #1');
$caretaker->backup();

$originator->setState('State #2');
$caretaker->backup();

$originator->setState('State #3');

echo "বর্তমান: " . $originator->getState() . "\n"; // State #3

$caretaker->undo();
echo "Undo: " . $originator->getState() . "\n"; // State #2

$caretaker->undo();
echo "Undo: " . $originator->getState() . "\n"; // State #1

$caretaker->redo();
echo "Redo: " . $originator->getState() . "\n"; // State #2
```

#### JavaScript (ES2022+)

```javascript
/**
 * Memento: immutable state snapshot
 * Private fields (#) দিয়ে encapsulation নিশ্চিত
 */
class Memento {
  #state;
  #timestamp;

  constructor(state) {
    // Deep clone করে encapsulation রক্ষা করি
    this.#state = structuredClone(state);
    this.#timestamp = new Date();
    Object.freeze(this);
  }

  getState() {
    return structuredClone(this.#state);
  }

  getTimestamp() {
    return this.#timestamp;
  }

  getName() {
    const preview = JSON.stringify(this.#state).slice(0, 25);
    return `${preview}... @ ${this.#timestamp.toLocaleTimeString()}`;
  }
}

/**
 * Originator: state-এর মালিক
 */
class Originator {
  #state;

  constructor(initialState = '') {
    this.#state = initialState;
  }

  setState(state) {
    this.#state = state;
  }

  getState() {
    return this.#state;
  }

  save() {
    return new Memento(this.#state);
  }

  restore(memento) {
    this.#state = memento.getState();
  }
}

/**
 * Caretaker: history ম্যানেজার
 */
class Caretaker {
  #originator;
  #undoStack = [];
  #redoStack = [];

  constructor(originator) {
    this.#originator = originator;
  }

  backup() {
    this.#undoStack.push(this.#originator.save());
    this.#redoStack.length = 0;
  }

  undo() {
    if (this.#undoStack.length === 0) {
      throw new Error('Undo stack খালি!');
    }
    this.#redoStack.push(this.#originator.save());
    const memento = this.#undoStack.pop();
    this.#originator.restore(memento);
  }

  redo() {
    if (this.#redoStack.length === 0) {
      throw new Error('Redo stack খালি!');
    }
    this.#undoStack.push(this.#originator.save());
    const memento = this.#redoStack.pop();
    this.#originator.restore(memento);
  }

  getHistory() {
    return this.#undoStack.map(m => m.getName());
  }
}

// ব্যবহার
const originator = new Originator();
const caretaker = new Caretaker(originator);

originator.setState('State #1');
caretaker.backup();

originator.setState('State #2');
caretaker.backup();

originator.setState('State #3');

console.log('বর্তমান:', originator.getState()); // State #3

caretaker.undo();
console.log('Undo:', originator.getState()); // State #2

caretaker.undo();
console.log('Undo:', originator.getState()); // State #1

caretaker.redo();
console.log('Redo:', originator.getState()); // State #2
```

---

### 2️⃣ Text Editor with Undo/Redo (Full History Stack)

#### PHP 8.3

```php
<?php

declare(strict_types=1);

final readonly class EditorMemento
{
    public function __construct(
        private string $content,
        private int $cursorPosition,
        private string $selectedText,
        private DateTimeImmutable $savedAt = new DateTimeImmutable(),
    ) {}

    public function getContent(): string { return $this->content; }
    public function getCursorPosition(): int { return $this->cursorPosition; }
    public function getSelectedText(): string { return $this->selectedText; }
    public function getSavedAt(): DateTimeImmutable { return $this->savedAt; }

    public function getDescription(): string
    {
        $preview = mb_substr($this->content, 0, 30, 'UTF-8');
        return sprintf('[%s] "%s..." (cursor: %d)',
            $this->savedAt->format('H:i:s'),
            $preview,
            $this->cursorPosition
        );
    }
}

class TextEditor
{
    private string $content = '';
    private int $cursorPosition = 0;
    private string $selectedText = '';

    public function type(string $text): self
    {
        $before = mb_substr($this->content, 0, $this->cursorPosition, 'UTF-8');
        $after = mb_substr($this->content, $this->cursorPosition, encoding: 'UTF-8');
        $this->content = $before . $text . $after;
        $this->cursorPosition += mb_strlen($text, 'UTF-8');
        $this->selectedText = '';
        return $this;
    }

    public function delete(int $count = 1): self
    {
        if ($this->cursorPosition > 0) {
            $before = mb_substr($this->content, 0, $this->cursorPosition - $count, 'UTF-8');
            $after = mb_substr($this->content, $this->cursorPosition, encoding: 'UTF-8');
            $this->content = $before . $after;
            $this->cursorPosition = max(0, $this->cursorPosition - $count);
        }
        return $this;
    }

    public function moveCursor(int $position): self
    {
        $this->cursorPosition = max(0, min($position, mb_strlen($this->content, 'UTF-8')));
        return $this;
    }

    public function select(int $start, int $end): self
    {
        $this->selectedText = mb_substr($this->content, $start, $end - $start, 'UTF-8');
        return $this;
    }

    public function getContent(): string { return $this->content; }
    public function getCursorPosition(): int { return $this->cursorPosition; }

    public function save(): EditorMemento
    {
        return new EditorMemento(
            $this->content,
            $this->cursorPosition,
            $this->selectedText,
        );
    }

    public function restore(EditorMemento $memento): void
    {
        $this->content = $memento->getContent();
        $this->cursorPosition = $memento->getCursorPosition();
        $this->selectedText = $memento->getSelectedText();
    }
}

class EditorHistory
{
    /** @var EditorMemento[] */
    private array $undoStack = [];
    /** @var EditorMemento[] */
    private array $redoStack = [];

    public function __construct(
        private readonly TextEditor $editor,
        private readonly int $maxHistory = 50,
    ) {}

    public function snapshot(): void
    {
        if (count($this->undoStack) >= $this->maxHistory) {
            array_shift($this->undoStack); // পুরনো history মুছে দাও
        }
        $this->undoStack[] = $this->editor->save();
        $this->redoStack = [];
    }

    public function undo(): bool
    {
        if (empty($this->undoStack)) return false;

        $this->redoStack[] = $this->editor->save();
        $memento = array_pop($this->undoStack);
        $this->editor->restore($memento);
        return true;
    }

    public function redo(): bool
    {
        if (empty($this->redoStack)) return false;

        $this->undoStack[] = $this->editor->save();
        $memento = array_pop($this->redoStack);
        $this->editor->restore($memento);
        return true;
    }

    public function canUndo(): bool { return !empty($this->undoStack); }
    public function canRedo(): bool { return !empty($this->redoStack); }
    public function getUndoCount(): int { return count($this->undoStack); }

    /** @return string[] */
    public function listHistory(): array
    {
        return array_map(
            fn(EditorMemento $m) => $m->getDescription(),
            $this->undoStack
        );
    }
}

// ব্যবহার: বাংলা টেক্সট এডিটিং
$editor = new TextEditor();
$history = new EditorHistory($editor);

$editor->type('আমার সোনার বাংলা');
$history->snapshot();

$editor->type(' আমি তোমায় ভালোবাসি');
$history->snapshot();

$editor->type(' — জাতীয় সংগীত');
$history->snapshot();

echo "বর্তমান: {$editor->getContent()}\n";
// আমার সোনার বাংলা আমি তোমায় ভালোবাসি — জাতীয় সংগীত

$history->undo();
echo "Undo 1: {$editor->getContent()}\n";
// আমার সোনার বাংলা আমি তোমায় ভালোবাসি

$history->undo();
echo "Undo 2: {$editor->getContent()}\n";
// আমার সোনার বাংলা

$history->redo();
echo "Redo: {$editor->getContent()}\n";
// আমার সোনার বাংলা আমি তোমায় ভালোবাসি

echo "History count: {$history->getUndoCount()}\n";
```

#### JavaScript (ES2022+)

```javascript
class EditorMemento {
  #content;
  #cursorPosition;
  #selectedText;
  #savedAt;

  constructor(content, cursorPosition, selectedText) {
    this.#content = content;
    this.#cursorPosition = cursorPosition;
    this.#selectedText = selectedText;
    this.#savedAt = new Date();
    Object.freeze(this);
  }

  getContent() { return this.#content; }
  getCursorPosition() { return this.#cursorPosition; }
  getSelectedText() { return this.#selectedText; }
  getSavedAt() { return this.#savedAt; }

  getDescription() {
    const preview = this.#content.slice(0, 30);
    return `[${this.#savedAt.toLocaleTimeString()}] "${preview}..." (cursor: ${this.#cursorPosition})`;
  }
}

class TextEditor {
  #content = '';
  #cursorPosition = 0;
  #selectedText = '';

  type(text) {
    const before = this.#content.slice(0, this.#cursorPosition);
    const after = this.#content.slice(this.#cursorPosition);
    this.#content = before + text + after;
    this.#cursorPosition += text.length;
    this.#selectedText = '';
    return this;
  }

  delete(count = 1) {
    if (this.#cursorPosition > 0) {
      const before = this.#content.slice(0, this.#cursorPosition - count);
      const after = this.#content.slice(this.#cursorPosition);
      this.#content = before + after;
      this.#cursorPosition = Math.max(0, this.#cursorPosition - count);
    }
    return this;
  }

  moveCursor(pos) {
    this.#cursorPosition = Math.max(0, Math.min(pos, this.#content.length));
    return this;
  }

  getContent() { return this.#content; }
  getCursorPosition() { return this.#cursorPosition; }

  save() {
    return new EditorMemento(this.#content, this.#cursorPosition, this.#selectedText);
  }

  restore(memento) {
    this.#content = memento.getContent();
    this.#cursorPosition = memento.getCursorPosition();
    this.#selectedText = memento.getSelectedText();
  }
}

class EditorHistory {
  #editor;
  #undoStack = [];
  #redoStack = [];
  #maxHistory;

  constructor(editor, maxHistory = 50) {
    this.#editor = editor;
    this.#maxHistory = maxHistory;
  }

  snapshot() {
    if (this.#undoStack.length >= this.#maxHistory) {
      this.#undoStack.shift();
    }
    this.#undoStack.push(this.#editor.save());
    this.#redoStack.length = 0;
  }

  undo() {
    if (this.#undoStack.length === 0) return false;
    this.#redoStack.push(this.#editor.save());
    this.#editor.restore(this.#undoStack.pop());
    return true;
  }

  redo() {
    if (this.#redoStack.length === 0) return false;
    this.#undoStack.push(this.#editor.save());
    this.#editor.restore(this.#redoStack.pop());
    return true;
  }

  canUndo() { return this.#undoStack.length > 0; }
  canRedo() { return this.#redoStack.length > 0; }

  listHistory() {
    return this.#undoStack.map(m => m.getDescription());
  }
}

// ব্যবহার
const editor = new TextEditor();
const history = new EditorHistory(editor);

editor.type('আমার সোনার বাংলা');
history.snapshot();

editor.type(' আমি তোমায় ভালোবাসি');
history.snapshot();

editor.delete(9);
history.snapshot();

console.log('বর্তমান:', editor.getContent());

history.undo();
console.log('Undo:', editor.getContent());
```

---

### 3️⃣ Form State Save/Restore (বাংলাদেশী ই-কমার্স)

#### PHP 8.3

```php
<?php

declare(strict_types=1);

final readonly class FormMemento
{
    public function __construct(
        private array $fields,
        private array $validationErrors,
        private string $label,
        private DateTimeImmutable $createdAt = new DateTimeImmutable(),
    ) {}

    public function getFields(): array { return $this->fields; }
    public function getValidationErrors(): array { return $this->validationErrors; }
    public function getLabel(): string { return $this->label; }
    public function getCreatedAt(): DateTimeImmutable { return $this->createdAt; }
}

class CheckoutForm
{
    private array $fields = [
        'name' => '',
        'phone' => '',
        'address' => '',
        'division' => '',
        'paymentMethod' => '',
        'bkashNumber' => '',
    ];

    private array $validationErrors = [];

    public function setField(string $key, string $value): void
    {
        if (!array_key_exists($key, $this->fields)) {
            throw new InvalidArgumentException("ফিল্ড '{$key}' বৈধ নয়!");
        }
        $this->fields[$key] = $value;
        $this->validate();
    }

    public function setFields(array $data): void
    {
        foreach ($data as $key => $value) {
            $this->setField($key, $value);
        }
    }

    private function validate(): void
    {
        $this->validationErrors = [];

        if (empty($this->fields['name'])) {
            $this->validationErrors['name'] = 'নাম দিতে হবে';
        }

        if (!empty($this->fields['phone']) && !preg_match('/^01[3-9]\d{8}$/', $this->fields['phone'])) {
            $this->validationErrors['phone'] = 'সঠিক বাংলাদেশী ফোন নম্বর দিন (01XXXXXXXXX)';
        }

        if ($this->fields['paymentMethod'] === 'bkash' && empty($this->fields['bkashNumber'])) {
            $this->validationErrors['bkashNumber'] = 'বিকাশ নম্বর দিতে হবে';
        }
    }

    public function getFields(): array { return $this->fields; }
    public function getErrors(): array { return $this->validationErrors; }
    public function isValid(): bool { return empty($this->validationErrors); }

    public function saveDraft(string $label = 'Draft'): FormMemento
    {
        return new FormMemento(
            $this->fields,
            $this->validationErrors,
            $label,
        );
    }

    public function restoreDraft(FormMemento $memento): void
    {
        $this->fields = $memento->getFields();
        $this->validationErrors = $memento->getValidationErrors();
    }
}

class FormDraftManager
{
    /** @var array<string, FormMemento> */
    private array $drafts = [];

    public function __construct(
        private readonly CheckoutForm $form,
    ) {}

    public function saveDraft(string $name): void
    {
        $this->drafts[$name] = $this->form->saveDraft($name);
    }

    public function loadDraft(string $name): void
    {
        if (!isset($this->drafts[$name])) {
            throw new RuntimeException("ড্রাফট '{$name}' পাওয়া যায়নি!");
        }
        $this->form->restoreDraft($this->drafts[$name]);
    }

    public function deleteDraft(string $name): void
    {
        unset($this->drafts[$name]);
    }

    /** @return string[] */
    public function listDrafts(): array
    {
        return array_map(
            fn(FormMemento $m) => sprintf('%s (%s)',
                $m->getLabel(),
                $m->getCreatedAt()->format('d M Y, h:i A')
            ),
            $this->drafts
        );
    }
}

// ব্যবহার: চালডাল-স্টাইল চেকআউট
$form = new CheckoutForm();
$draftManager = new FormDraftManager($form);

// প্রথমবার ফর্ম পূরণ
$form->setFields([
    'name' => 'রহিম উদ্দিন',
    'phone' => '01712345678',
    'address' => 'মিরপুর ১০, ঢাকা',
    'division' => 'ঢাকা',
    'paymentMethod' => 'cod',
]);

// ড্রাফট সেভ — ব্যবহারকারী পরে ফিরে আসতে পারবে
$draftManager->saveDraft('ঢাকা_অর্ডার');

// ফর্ম পরিবর্তন — অন্য ঠিকানায় অর্ডার
$form->setFields([
    'name' => 'করিম সাহেব',
    'phone' => '01898765432',
    'address' => 'লালমাটিয়া, ঢাকা',
    'division' => 'ঢাকা',
    'paymentMethod' => 'bkash',
    'bkashNumber' => '01898765432',
]);

$draftManager->saveDraft('বিকাশ_অর্ডার');

// আগের ড্রাফটে ফিরে যাও
$draftManager->loadDraft('ঢাকা_অর্ডার');
echo $form->getFields()['name']; // রহিম উদ্দিন
```

#### JavaScript (ES2022+)

```javascript
class FormMemento {
  #fields;
  #errors;
  #label;
  #createdAt;

  constructor(fields, errors, label) {
    this.#fields = structuredClone(fields);
    this.#errors = structuredClone(errors);
    this.#label = label;
    this.#createdAt = new Date();
    Object.freeze(this);
  }

  getFields() { return structuredClone(this.#fields); }
  getErrors() { return structuredClone(this.#errors); }
  getLabel() { return this.#label; }
  getCreatedAt() { return this.#createdAt; }
}

class CheckoutForm {
  #fields = {
    name: '',
    phone: '',
    address: '',
    division: '',
    paymentMethod: '',
    bkashNumber: '',
  };
  #errors = {};

  setField(key, value) {
    if (!(key in this.#fields)) throw new Error(`ফিল্ড '${key}' বৈধ নয়!`);
    this.#fields[key] = value;
    this.#validate();
  }

  setFields(data) {
    Object.entries(data).forEach(([k, v]) => {
      if (k in this.#fields) this.#fields[k] = v;
    });
    this.#validate();
  }

  #validate() {
    this.#errors = {};
    if (!this.#fields.name) this.#errors.name = 'নাম দিতে হবে';
    if (this.#fields.phone && !/^01[3-9]\d{8}$/.test(this.#fields.phone)) {
      this.#errors.phone = 'সঠিক বাংলাদেশী ফোন নম্বর দিন';
    }
    if (this.#fields.paymentMethod === 'bkash' && !this.#fields.bkashNumber) {
      this.#errors.bkashNumber = 'বিকাশ নম্বর দিতে হবে';
    }
  }

  getFields() { return structuredClone(this.#fields); }
  getErrors() { return structuredClone(this.#errors); }
  isValid() { return Object.keys(this.#errors).length === 0; }

  saveDraft(label = 'Draft') {
    return new FormMemento(this.#fields, this.#errors, label);
  }

  restoreDraft(memento) {
    this.#fields = memento.getFields();
    this.#errors = memento.getErrors();
  }
}

class FormDraftManager {
  #form;
  #drafts = new Map();

  constructor(form) {
    this.#form = form;
  }

  saveDraft(name) {
    this.#drafts.set(name, this.#form.saveDraft(name));
  }

  loadDraft(name) {
    const draft = this.#drafts.get(name);
    if (!draft) throw new Error(`ড্রাফট '${name}' পাওয়া যায়নি!`);
    this.#form.restoreDraft(draft);
  }

  deleteDraft(name) {
    this.#drafts.delete(name);
  }

  listDrafts() {
    return [...this.#drafts.entries()].map(
      ([name, m]) => `${m.getLabel()} (${m.getCreatedAt().toLocaleString('bn-BD')})`
    );
  }
}
```

---

### 4️⃣ Game Checkpoint System

#### PHP 8.3

```php
<?php

declare(strict_types=1);

enum Difficulty: string
{
    case Easy = 'সহজ';
    case Medium = 'মাঝারি';
    case Hard = 'কঠিন';
}

final readonly class GameMemento
{
    public function __construct(
        private string $playerName,
        private int $level,
        private int $health,
        private int $score,
        private array $inventory,
        private array $position,
        private Difficulty $difficulty,
        private DateTimeImmutable $savedAt = new DateTimeImmutable(),
    ) {}

    public function getPlayerName(): string { return $this->playerName; }
    public function getLevel(): int { return $this->level; }
    public function getHealth(): int { return $this->health; }
    public function getScore(): int { return $this->score; }
    public function getInventory(): array { return $this->inventory; }
    public function getPosition(): array { return $this->position; }
    public function getDifficulty(): Difficulty { return $this->difficulty; }
    public function getSavedAt(): DateTimeImmutable { return $this->savedAt; }

    public function getSummary(): string
    {
        return sprintf(
            '🎮 %s | Lv.%d | ❤️%d | 🏆%d | 📍(%d,%d) | %s',
            $this->playerName,
            $this->level,
            $this->health,
            $this->score,
            $this->position['x'],
            $this->position['y'],
            $this->savedAt->format('d/m/Y H:i')
        );
    }
}

class GameCharacter
{
    private int $health = 100;
    private int $score = 0;
    private int $level = 1;
    private array $inventory = [];
    private array $position = ['x' => 0, 'y' => 0];

    public function __construct(
        private readonly string $name,
        private readonly Difficulty $difficulty = Difficulty::Medium,
    ) {}

    public function takeDamage(int $amount): void
    {
        $multiplier = match($this->difficulty) {
            Difficulty::Easy => 0.5,
            Difficulty::Medium => 1.0,
            Difficulty::Hard => 1.5,
        };
        $this->health = max(0, $this->health - (int)($amount * $multiplier));
    }

    public function heal(int $amount): void
    {
        $this->health = min(100, $this->health + $amount);
    }

    public function addScore(int $points): void { $this->score += $points; }
    public function levelUp(): void { $this->level++; }
    public function addItem(string $item): void { $this->inventory[] = $item; }
    public function moveTo(int $x, int $y): void { $this->position = ['x' => $x, 'y' => $y]; }

    public function isAlive(): bool { return $this->health > 0; }
    public function getHealth(): int { return $this->health; }
    public function getScore(): int { return $this->score; }

    public function saveCheckpoint(): GameMemento
    {
        return new GameMemento(
            $this->name,
            $this->level,
            $this->health,
            $this->score,
            $this->inventory,
            $this->position,
            $this->difficulty,
        );
    }

    public function loadCheckpoint(GameMemento $memento): void
    {
        $this->level = $memento->getLevel();
        $this->health = $memento->getHealth();
        $this->score = $memento->getScore();
        $this->inventory = $memento->getInventory();
        $this->position = $memento->getPosition();
    }
}

class SaveManager
{
    /** @var array<string, GameMemento> */
    private array $slots = [];
    private array $autoSaves = [];

    public function __construct(
        private readonly int $maxAutoSaves = 3,
    ) {}

    public function saveToSlot(string $slotName, GameCharacter $character): void
    {
        $this->slots[$slotName] = $character->saveCheckpoint();
    }

    public function loadFromSlot(string $slotName, GameCharacter $character): void
    {
        if (!isset($this->slots[$slotName])) {
            throw new RuntimeException("সেভ স্লট '{$slotName}' খালি!");
        }
        $character->loadCheckpoint($this->slots[$slotName]);
    }

    public function autoSave(GameCharacter $character): void
    {
        $this->autoSaves[] = $character->saveCheckpoint();
        if (count($this->autoSaves) > $this->maxAutoSaves) {
            array_shift($this->autoSaves);
        }
    }

    public function loadLastAutoSave(GameCharacter $character): void
    {
        if (empty($this->autoSaves)) {
            throw new RuntimeException('কোনো অটো-সেভ নেই!');
        }
        $character->loadCheckpoint(end($this->autoSaves));
    }

    /** @return string[] */
    public function listSaves(): array
    {
        $saves = [];
        foreach ($this->slots as $name => $memento) {
            $saves[] = "[{$name}] {$memento->getSummary()}";
        }
        foreach ($this->autoSaves as $i => $memento) {
            $saves[] = "[Auto-{$i}] {$memento->getSummary()}";
        }
        return $saves;
    }
}

// গেমপ্লে সিমুলেশন
$player = new GameCharacter('বীরবল', Difficulty::Hard);
$saveManager = new SaveManager(maxAutoSaves: 3);

$player->moveTo(10, 20);
$player->addItem('তলোয়ার');
$player->addScore(500);
$saveManager->autoSave($player);

$player->levelUp();
$player->moveTo(50, 80);
$player->addItem('ঢাল');
$saveManager->saveToSlot('বস_ফাইটের_আগে', $player);

// বস ফাইট — ক্ষতি হলো!
$player->takeDamage(80);
echo "Health after boss: {$player->getHealth()}\n"; // খুবই কম!

// লোড করো!
$saveManager->loadFromSlot('বস_ফাইটের_আগে', $player);
echo "Health restored: {$player->getHealth()}\n"; // পূর্ণ health!

foreach ($saveManager->listSaves() as $save) {
    echo $save . "\n";
}
```

#### JavaScript (ES2022+)

```javascript
class GameMemento {
  #state;
  #savedAt;

  constructor(state) {
    this.#state = structuredClone(state);
    this.#savedAt = new Date();
    Object.freeze(this);
  }

  getState() { return structuredClone(this.#state); }
  getSavedAt() { return this.#savedAt; }

  getSummary() {
    const s = this.#state;
    return `🎮 ${s.name} | Lv.${s.level} | ❤️${s.health} | 🏆${s.score}`;
  }
}

class GameCharacter {
  #name;
  #level = 1;
  #health = 100;
  #score = 0;
  #inventory = [];
  #position = { x: 0, y: 0 };

  constructor(name) {
    this.#name = name;
  }

  takeDamage(amount) { this.#health = Math.max(0, this.#health - amount); }
  heal(amount) { this.#health = Math.min(100, this.#health + amount); }
  addScore(pts) { this.#score += pts; }
  levelUp() { this.#level++; }
  addItem(item) { this.#inventory.push(item); }
  moveTo(x, y) { this.#position = { x, y }; }
  isAlive() { return this.#health > 0; }
  getHealth() { return this.#health; }

  saveCheckpoint() {
    return new GameMemento({
      name: this.#name,
      level: this.#level,
      health: this.#health,
      score: this.#score,
      inventory: this.#inventory,
      position: this.#position,
    });
  }

  loadCheckpoint(memento) {
    const s = memento.getState();
    this.#level = s.level;
    this.#health = s.health;
    this.#score = s.score;
    this.#inventory = s.inventory;
    this.#position = s.position;
  }
}

class SaveManager {
  #slots = new Map();
  #autoSaves = [];
  #maxAutoSaves;

  constructor(maxAutoSaves = 3) {
    this.#maxAutoSaves = maxAutoSaves;
  }

  saveToSlot(name, character) {
    this.#slots.set(name, character.saveCheckpoint());
  }

  loadFromSlot(name, character) {
    const memento = this.#slots.get(name);
    if (!memento) throw new Error(`সেভ স্লট '${name}' খালি!`);
    character.loadCheckpoint(memento);
  }

  autoSave(character) {
    this.#autoSaves.push(character.saveCheckpoint());
    if (this.#autoSaves.length > this.#maxAutoSaves) this.#autoSaves.shift();
  }

  loadLastAutoSave(character) {
    if (this.#autoSaves.length === 0) throw new Error('কোনো অটো-সেভ নেই!');
    character.loadCheckpoint(this.#autoSaves.at(-1));
  }

  listSaves() {
    const saves = [];
    for (const [name, m] of this.#slots) saves.push(`[${name}] ${m.getSummary()}`);
    this.#autoSaves.forEach((m, i) => saves.push(`[Auto-${i}] ${m.getSummary()}`));
    return saves;
  }
}
```

---

## 🌍 Real-World Applicable Areas

### কোথায় কোথায় Memento প্যাটার্ন ব্যবহৃত হয়?

| ক্ষেত্র | বর্ণনা | উদাহরণ |
|---|---|---|
| **Undo/Redo** | টেক্সট এডিটরে Ctrl+Z / Ctrl+Y | VS Code, Notepad++, Google Docs |
| **Browser History** | Back/Forward নেভিগেশন | প্রতিটি পেজ ভিজিট একটি memento |
| **Database Transaction** | Rollback অপারেশন | BEGIN → কাজ → ROLLBACK/COMMIT |
| **Git Commits** | প্রতিটি commit একটি snapshot | `git checkout <hash>` দিয়ে আগের অবস্থায় যাওয়া |
| **Form Draft** | ফর্মের ড্রাফট সেভ | Gmail draft, Facebook post draft |
| **Shopping Cart** | কার্ট state সংরক্ষণ | দারাজে ব্রাউজার বন্ধ করলেও কার্ট থাকে |
| **Canvas/Drawing** | ড্রয়িং অ্যাপে স্টেপ সেভ | Photoshop history panel |
| **Game Saves** | গেম চেকপয়েন্ট | RPG গেমের সেভ পয়েন্ট |
| **Configuration** | সেটিংস রোলব্যাক | সিস্টেম রিস্টোর পয়েন্ট |
| **Workflow State** | প্রসেস স্টেট ম্যানেজমেন্ট | Order processing pipeline |

### বাংলাদেশ প্রসঙ্গে বাস্তব ব্যবহার

1. **বিকাশ/নগদ ট্রানজেকশন**: ব্যর্থ ট্রানজেকশনে ব্যালেন্স রোলব্যাক
2. **সরকারি ই-সেবা ফর্ম**: NID আবেদনের ড্রাফট সেভ
3. **পত্রিকার CMS**: প্রথম আলো/ডেইলি স্টারের আর্টিকেল ড্রাফট সিস্টেম
4. **শিক্ষা পোর্টাল**: অনলাইন পরীক্ষায় উত্তর সেভ (সময় শেষ হলে শেষ সেভ পয়েন্ট থেকে)

---

## 🔥 Advanced Deep Dive

### 1. Memento vs Serialization

```
┌───────────────────────────────────┬───────────────────────────────────┐
│           Memento                  │         Serialization             │
├───────────────────────────────────┼───────────────────────────────────┤
│ In-memory snapshot                │ Persistent storage (disk/DB)      │
│ Same process lifecycle            │ Cross-process/cross-session       │
│ Encapsulation সংরক্ষিত           │ State expose হতে পারে            │
│ দ্রুত (memory copy)              │ ধীর (I/O overhead)               │
│ Complex object graphs সহজে       │ Circular refs সমস্যা             │
│ Pattern-specific                  │ General purpose                   │
└───────────────────────────────────┴───────────────────────────────────┘
```

### 2. Memento + Command Pattern (Undo Operations)

দুটি প্যাটার্ন মিলিয়ে শক্তিশালী undo সিস্টেম তৈরি করা যায়:

```php
<?php

// Command প্রতিটি অপারেশনকে represent করে
// Memento সেই অপারেশনের আগের state সংরক্ষণ করে

interface Command
{
    public function execute(): void;
    public function undo(): void;
    public function getDescription(): string;
}

final class ChangeTextCommand implements Command
{
    private ?EditorMemento $beforeState = null;

    public function __construct(
        private readonly TextEditor $editor,
        private readonly string $newText,
    ) {}

    public function execute(): void
    {
        $this->beforeState = $this->editor->save(); // Memento সেভ!
        $this->editor->type($this->newText);
    }

    public function undo(): void
    {
        if ($this->beforeState !== null) {
            $this->editor->restore($this->beforeState); // Memento রিস্টোর!
        }
    }

    public function getDescription(): string
    {
        return "Type: '{$this->newText}'";
    }
}

final class DeleteTextCommand implements Command
{
    private ?EditorMemento $beforeState = null;

    public function __construct(
        private readonly TextEditor $editor,
        private readonly int $charCount,
    ) {}

    public function execute(): void
    {
        $this->beforeState = $this->editor->save();
        $this->editor->delete($this->charCount);
    }

    public function undo(): void
    {
        if ($this->beforeState !== null) {
            $this->editor->restore($this->beforeState);
        }
    }

    public function getDescription(): string
    {
        return "Delete {$this->charCount} chars";
    }
}

class CommandHistory
{
    /** @var Command[] */
    private array $executed = [];
    /** @var Command[] */
    private array $undone = [];

    public function execute(Command $command): void
    {
        $command->execute();
        $this->executed[] = $command;
        $this->undone = [];
    }

    public function undo(): void
    {
        if (empty($this->executed)) return;
        $command = array_pop($this->executed);
        $command->undo();
        $this->undone[] = $command;
    }

    public function redo(): void
    {
        if (empty($this->undone)) return;
        $command = array_pop($this->undone);
        $command->execute();
        $this->executed[] = $command;
    }

    /** @return string[] */
    public function getLog(): array
    {
        return array_map(fn(Command $c) => $c->getDescription(), $this->executed);
    }
}
```

### 3. Memory Optimization: Incremental/Delta Memento

পূর্ণ state snapshot মেমোরি-ব্যয়বহুল। **Delta Memento** শুধু পরিবর্তনগুলো সংরক্ষণ করে:

```javascript
/**
 * DeltaMemento: শুধু পরিবর্তিত ফিল্ডগুলো সংরক্ষণ করে
 * বড় অবজেক্টের ক্ষেত্রে অনেক মেমোরি বাঁচায়
 */
class DeltaMemento {
  #changes;
  #timestamp;

  constructor(changes) {
    this.#changes = structuredClone(changes);
    this.#timestamp = Date.now();
    Object.freeze(this);
  }

  getChanges() { return structuredClone(this.#changes); }
  getTimestamp() { return this.#timestamp; }
}

class DeltaTracker {
  #currentState;
  #history = [];

  constructor(initialState) {
    this.#currentState = structuredClone(initialState);
  }

  update(newState) {
    const delta = {};
    const previousValues = {};
    let hasChanges = false;

    for (const [key, value] of Object.entries(newState)) {
      if (JSON.stringify(this.#currentState[key]) !== JSON.stringify(value)) {
        previousValues[key] = this.#currentState[key];
        delta[key] = value;
        hasChanges = true;
      }
    }

    if (hasChanges) {
      // আগের মান সেভ করো (undo-র জন্য)
      this.#history.push(new DeltaMemento(previousValues));
      Object.assign(this.#currentState, delta);
    }
  }

  undo() {
    if (this.#history.length === 0) return false;
    const memento = this.#history.pop();
    const changes = memento.getChanges();
    Object.assign(this.#currentState, changes);
    return true;
  }

  getState() { return structuredClone(this.#currentState); }

  getMemoryUsage() {
    const fullSize = JSON.stringify(this.#currentState).length;
    const deltaSize = this.#history.reduce(
      (sum, m) => sum + JSON.stringify(m.getChanges()).length, 0
    );
    return {
      fullSnapshotSize: fullSize,
      totalDeltaSize: deltaSize,
      savings: `${(((fullSize * this.#history.length) - deltaSize) / (fullSize * this.#history.length) * 100).toFixed(1)}%`,
    };
  }
}

// ব্যবহার
const tracker = new DeltaTracker({
  title: 'প্রাথমিক শিরোনাম',
  body: 'অনেক বড় একটি টেক্সট...'.repeat(100),
  tags: ['javascript', 'design-pattern'],
  metadata: { author: 'রহিম', views: 0 },
});

// শুধু title পরিবর্তন — body এর পুরো কপি সেভ হবে না!
tracker.update({ title: 'নতুন শিরোনাম' });
tracker.update({ tags: ['javascript', 'memento', 'bangla'] });

console.log(tracker.getMemoryUsage());
// { fullSnapshotSize: 2548, totalDeltaSize: 89, savings: "98.3%" }
```

### 4. Memento with Compression (PHP)

বড় state-এর ক্ষেত্রে compression ব্যবহার:

```php
<?php

final readonly class CompressedMemento
{
    private string $compressedData;
    private int $originalSize;

    public function __construct(array $state)
    {
        $serialized = serialize($state);
        $this->originalSize = strlen($serialized);
        $this->compressedData = gzcompress($serialized, 9);
    }

    public function getState(): array
    {
        $decompressed = gzuncompress($this->compressedData);
        return unserialize($decompressed);
    }

    public function getCompressionRatio(): float
    {
        return round(
            (1 - strlen($this->compressedData) / $this->originalSize) * 100,
            2
        );
    }

    public function getCompressedSize(): int
    {
        return strlen($this->compressedData);
    }
}

// বড় ডেটাসেটের ক্ষেত্রে উপকারী
$largeState = [
    'content' => str_repeat('বাংলা টেক্সট ', 1000),
    'metadata' => array_fill(0, 100, ['key' => 'value']),
];

$memento = new CompressedMemento($largeState);
echo "Compression: {$memento->getCompressionRatio()}%\n";
echo "Compressed size: {$memento->getCompressedSize()} bytes\n";

$restored = $memento->getState();
assert($restored === $largeState); // পুরোপুরি একই!
```

### 5. Memento in Event Sourcing

Event Sourcing-এ প্রতিটি event একটি ক্ষুদ্র memento। সব event replay করে current state পাওয়া যায়:

```php
<?php

declare(strict_types=1);

readonly class DomainEvent
{
    public function __construct(
        public string $type,
        public array $payload,
        public DateTimeImmutable $occurredAt = new DateTimeImmutable(),
    ) {}
}

class OrderAggregate
{
    private string $status = 'draft';
    private array $items = [];
    private float $total = 0.0;

    /** @var DomainEvent[] */
    private array $events = [];

    /** @var array[] - Snapshots প্রতি N events-এ */
    private array $snapshots = [];
    private int $snapshotInterval = 10;

    public function addItem(string $name, float $price, int $qty): void
    {
        $this->apply(new DomainEvent('item_added', [
            'name' => $name,
            'price' => $price,
            'quantity' => $qty,
        ]));
    }

    public function confirm(): void
    {
        $this->apply(new DomainEvent('order_confirmed', []));
    }

    private function apply(DomainEvent $event): void
    {
        $this->events[] = $event;

        match ($event->type) {
            'item_added' => $this->handleItemAdded($event->payload),
            'order_confirmed' => $this->status = 'confirmed',
            default => throw new RuntimeException("Unknown event: {$event->type}"),
        };

        // প্রতি N events-এ snapshot (memento) সেভ
        if (count($this->events) % $this->snapshotInterval === 0) {
            $this->snapshots[] = $this->takeSnapshot();
        }
    }

    private function handleItemAdded(array $payload): void
    {
        $this->items[] = $payload;
        $this->total += $payload['price'] * $payload['quantity'];
    }

    private function takeSnapshot(): array
    {
        return [
            'status' => $this->status,
            'items' => $this->items,
            'total' => $this->total,
            'eventCount' => count($this->events),
        ];
    }

    /**
     * নির্দিষ্ট সময়ে অর্ডারের অবস্থা পুনরুদ্ধার
     * Snapshot থেকে শুরু করে event replay করে — অনেক দ্রুত!
     */
    public function reconstructAt(int $eventIndex): array
    {
        // সবচেয়ে কাছের snapshot খুঁজো
        $nearestSnapshot = null;
        $startFrom = 0;

        foreach ($this->snapshots as $snapshot) {
            if ($snapshot['eventCount'] <= $eventIndex) {
                $nearestSnapshot = $snapshot;
                $startFrom = $snapshot['eventCount'];
            }
        }

        // Snapshot থেকে শুরু করে বাকি events replay করো
        $state = $nearestSnapshot ?? ['status' => 'draft', 'items' => [], 'total' => 0.0];

        for ($i = $startFrom; $i < $eventIndex; $i++) {
            $event = $this->events[$i];
            if ($event->type === 'item_added') {
                $state['items'][] = $event->payload;
                $state['total'] += $event->payload['price'] * $event->payload['quantity'];
            } elseif ($event->type === 'order_confirmed') {
                $state['status'] = 'confirmed';
            }
        }

        return $state;
    }

    public function getEventCount(): int { return count($this->events); }
    public function getSnapshotCount(): int { return count($this->snapshots); }
}
```

### 6. Caretaker Strategies

#### Limited History (PHP)

```php
<?php

class LimitedHistoryCaretaker
{
    private SplDoublyLinkedList $history;

    public function __construct(
        private readonly Originator $originator,
        private readonly int $maxSize = 20,
        private readonly int $maxAgeSeconds = 3600,
    ) {
        $this->history = new SplDoublyLinkedList();
        $this->history->setIteratorMode(SplDoublyLinkedList::IT_MODE_LIFO);
    }

    public function backup(): void
    {
        $this->history->push($this->originator->save());
        $this->enforceMaxSize();
        $this->cleanupExpired();
    }

    private function enforceMaxSize(): void
    {
        while ($this->history->count() > $this->maxSize) {
            $this->history->shift(); // পুরনোটা মুছে দাও
        }
    }

    private function cleanupExpired(): void
    {
        $cutoff = new DateTimeImmutable("-{$this->maxAgeSeconds} seconds");
        $cleaned = new SplDoublyLinkedList();

        for ($this->history->rewind(); $this->history->valid(); $this->history->next()) {
            $memento = $this->history->current();
            if ($memento->getTimestamp() > $cutoff) {
                $cleaned->push($memento);
            }
        }

        $this->history = $cleaned;
    }

    public function undo(): void
    {
        if ($this->history->isEmpty()) return;
        $memento = $this->history->pop();
        $this->originator->restore($memento);
    }

    public function getSize(): int { return $this->history->count(); }
}
```

#### Time-based Cleanup (JavaScript)

```javascript
class TimedCaretaker {
  #originator;
  #history = [];
  #maxAge; // milliseconds

  constructor(originator, { maxSize = 50, maxAgeMs = 3_600_000 } = {}) {
    this.#originator = originator;
    this.#maxSize = maxSize;
    this.#maxAge = maxAgeMs;

    // পর্যায়ক্রমে পুরনো memento গুলো পরিষ্কার করো
    this.#cleanupInterval = setInterval(() => this.#cleanup(), 60_000);
  }

  #maxSize;
  #cleanupInterval;

  backup() {
    this.#history.push(this.#originator.save());
    if (this.#history.length > this.#maxSize) {
      this.#history.shift();
    }
  }

  undo() {
    if (this.#history.length === 0) return false;
    this.#originator.restore(this.#history.pop());
    return true;
  }

  #cleanup() {
    const cutoff = Date.now() - this.#maxAge;
    this.#history = this.#history.filter(
      m => m.getTimestamp().getTime() > cutoff
    );
  }

  destroy() {
    clearInterval(this.#cleanupInterval);
    this.#history.length = 0;
  }

  get size() { return this.#history.length; }
}
```

### 7. PHP serialize/unserialize with Memento

PHP-তে `serialize()` এবং `unserialize()` ব্যবহার করে persistent memento তৈরি:

```php
<?php

class PersistentMementoStore
{
    public function __construct(
        private readonly string $storagePath,
    ) {
        if (!is_dir($storagePath)) {
            mkdir($storagePath, 0755, true);
        }
    }

    public function save(string $key, Memento $memento): void
    {
        $filepath = $this->getFilePath($key);
        $data = serialize($memento);
        file_put_contents($filepath, $data, LOCK_EX);
    }

    public function load(string $key): Memento
    {
        $filepath = $this->getFilePath($key);
        if (!file_exists($filepath)) {
            throw new RuntimeException("Memento '{$key}' পাওয়া যায়নি!");
        }

        $data = file_get_contents($filepath);
        $memento = unserialize($data, ['allowed_classes' => [Memento::class, DateTimeImmutable::class]]);

        if (!$memento instanceof Memento) {
            throw new RuntimeException('Invalid memento data!');
        }

        return $memento;
    }

    public function delete(string $key): void
    {
        $filepath = $this->getFilePath($key);
        if (file_exists($filepath)) {
            unlink($filepath);
        }
    }

    /** @return string[] */
    public function listKeys(): array
    {
        $files = glob($this->storagePath . '/*.memento');
        return array_map(
            fn(string $f) => basename($f, '.memento'),
            $files
        );
    }

    private function getFilePath(string $key): string
    {
        $safeKey = preg_replace('/[^a-zA-Z0-9_-]/', '_', $key);
        return "{$this->storagePath}/{$safeKey}.memento";
    }
}
```

### 8. JS structuredClone for Deep State Capture

`structuredClone()` হলো JavaScript-এ deep copy করার আধুনিক উপায়:

```javascript
/**
 * structuredClone বনাম অন্যান্য deep copy পদ্ধতি:
 *
 * ✅ structuredClone():
 *    - Circular references হ্যান্ডল করে
 *    - Date, Map, Set, ArrayBuffer সমর্থন করে
 *    - Native, দ্রুত
 *
 * ❌ JSON.parse(JSON.stringify()):
 *    - Date → string হয়ে যায়
 *    - undefined, function, Symbol হারিয়ে যায়
 *    - Circular references → Error!
 *
 * ❌ Spread ({...obj}):
 *    - শুধু shallow copy
 *    - Nested objects shared থাকে
 */

class RobustMemento {
  #state;
  #metadata;

  constructor(state, metadata = {}) {
    // structuredClone সব complex type সঠিকভাবে কপি করে
    this.#state = structuredClone(state);
    this.#metadata = {
      timestamp: Date.now(),
      stateSize: new Blob([JSON.stringify(state)]).size,
      ...metadata,
    };
    Object.freeze(this);
  }

  getState() {
    return structuredClone(this.#state);
  }

  getMetadata() {
    return { ...this.#metadata };
  }
}

// Complex state-এর ক্ষেত্রে structuredClone নির্ভরযোগ্য
const complexState = {
  items: [new Map([['price', 500], ['name', 'চাল']])],
  createdAt: new Date(),
  config: new Set(['option1', 'option2']),
  nested: { deep: { value: 42 } },
};

const memento = new RobustMemento(complexState);
const restored = memento.getState();

// Map, Set, Date সবকিছু সঠিকভাবে পুনরুদ্ধার হয়েছে
console.log(restored.items[0] instanceof Map);       // true
console.log(restored.createdAt instanceof Date);      // true
console.log(restored.config instanceof Set);          // true
console.log(restored.nested.deep.value);              // 42
```

---

## ✅ Pros (সুবিধা)

| সুবিধা | বিবরণ |
|---|---|
| **Encapsulation রক্ষা** | অবজেক্টের internal state সরাসরি expose হয় না |
| **Single Responsibility** | Originator state ম্যানেজ করে, Caretaker history ম্যানেজ করে |
| **সহজ Undo/Redo** | Stack-based history দিয়ে সহজেই undo/redo করা যায় |
| **State Isolation** | প্রতিটি memento স্বাধীন — একটি পরিবর্তন করলে অন্যটি প্রভাবিত হয় না |
| **Snapshot Flexibility** | যেকোনো সময়ে state snapshot নেওয়া যায় |
| **Testability** | State পুনরুদ্ধার করে নির্দিষ্ট scenario টেস্ট করা সহজ |

## ❌ Cons (অসুবিধা)

| অসুবিধা | বিবরণ |
|---|---|
| **Memory Consumption** | প্রতিটি snapshot মেমোরি ব্যবহার করে — বড় state-এ সমস্যা |
| **Performance Overhead** | Deep copy ব্যয়বহুল হতে পারে |
| **State Consistency** | Complex object graph-এ সব reference সঠিকভাবে কপি নাও হতে পারে |
| **Storage Management** | History সীমাহীন রাখলে memory leak হতে পারে |
| **Versioning সমস্যা** | Originator-এর structure পরিবর্তন হলে পুরনো memento incompatible হতে পারে |

---

## ⚠️ Common Mistakes (সাধারণ ভুল)

### 1. Memory Leak — সীমাহীন History

```javascript
// ❌ ভুল: কখনো পুরনো memento মুছে না!
class BadCaretaker {
  #history = []; // ক্রমাগত বাড়তেই থাকে!

  backup(originator) {
    this.#history.push(originator.save()); // কোনো সীমা নেই!
  }
}

// ✅ সঠিক: সীমা নির্ধারণ করো
class GoodCaretaker {
  #history = [];
  #maxSize;

  constructor(maxSize = 100) {
    this.#maxSize = maxSize;
  }

  backup(originator) {
    if (this.#history.length >= this.#maxSize) {
      this.#history.shift(); // পুরনোটা মুছে দাও
    }
    this.#history.push(originator.save());
  }
}
```

### 2. Internal State Expose করা

```php
// ❌ ভুল: Memento-র state বাইরে expose!
class LeakyMemento
{
    public array $state; // Public! যে কেউ পরিবর্তন করতে পারে!
}

// ✅ সঠিক: readonly এবং private ব্যবহার করো
final readonly class SafeMemento
{
    public function __construct(
        private array $state,
    ) {}

    public function getState(): array
    {
        return $this->state; // readonly class — পরিবর্তনযোগ্য নয়
    }
}
```

### 3. Shallow Copy সমস্যা

```javascript
// ❌ ভুল: Shallow copy — reference shared!
class ShallowMemento {
  constructor(state) {
    this.state = { ...state }; // Nested objects এখনও shared!
  }
}

const original = { items: [1, 2, 3], nested: { value: 10 } };
const memento = new ShallowMemento(original);
original.nested.value = 999;
console.log(memento.state.nested.value); // 999! ভুল!

// ✅ সঠিক: structuredClone ব্যবহার করো
class DeepMemento {
  #state;
  constructor(state) {
    this.#state = structuredClone(state); // সত্যিকারের deep copy
  }
  getState() { return structuredClone(this.#state); }
}
```

### 4. Redo Stack ভুলে যাওয়া

```javascript
// ❌ ভুল: নতুন backup-এ redo stack ক্লিয়ার না করা
backup() {
  this.undoStack.push(originator.save());
  // redo stack ক্লিয়ার করতে ভুলে গেছি!
  // ফলে undo → নতুন কাজ → redo করলে ভুল state আসবে
}

// ✅ সঠিক:
backup() {
  this.undoStack.push(originator.save());
  this.redoStack.length = 0; // redo history মুছে দাও!
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

class MementoTest extends TestCase
{
    private TextEditor $editor;
    private EditorHistory $history;

    protected function setUp(): void
    {
        $this->editor = new TextEditor();
        $this->history = new EditorHistory($this->editor);
    }

    #[Test]
    public function it_saves_and_restores_state(): void
    {
        $this->editor->type('হ্যালো');
        $memento = $this->editor->save();

        $this->editor->type(' বিশ্ব');
        $this->assertSame('হ্যালো বিশ্ব', $this->editor->getContent());

        $this->editor->restore($memento);
        $this->assertSame('হ্যালো', $this->editor->getContent());
    }

    #[Test]
    public function undo_restores_previous_state(): void
    {
        $this->editor->type('প্রথম');
        $this->history->snapshot();

        $this->editor->type(' দ্বিতীয়');
        $this->history->snapshot();

        $this->editor->type(' তৃতীয়');

        $this->history->undo();
        $this->assertSame('প্রথম দ্বিতীয়', $this->editor->getContent());

        $this->history->undo();
        $this->assertSame('প্রথম', $this->editor->getContent());
    }

    #[Test]
    public function redo_restores_undone_state(): void
    {
        $this->editor->type('A');
        $this->history->snapshot();

        $this->editor->type('B');
        $this->history->snapshot();

        $this->history->undo(); // B → A+B
        $this->history->undo(); // A+B → A

        $this->history->redo(); // A → A+B
        $this->assertStringContainsString('B', $this->editor->getContent());
    }

    #[Test]
    public function new_edit_clears_redo_stack(): void
    {
        $this->editor->type('A');
        $this->history->snapshot();

        $this->editor->type('B');
        $this->history->snapshot();

        $this->history->undo();

        $this->editor->type('C');
        $this->history->snapshot();

        // Redo আর কাজ করা উচিত নয়
        $this->assertFalse($this->history->redo());
    }

    #[Test]
    public function empty_undo_returns_false(): void
    {
        $this->assertFalse($this->history->undo());
    }

    #[Test]
    public function respects_max_history_limit(): void
    {
        $limitedHistory = new EditorHistory($this->editor, maxHistory: 3);

        for ($i = 1; $i <= 5; $i++) {
            $this->editor->type("State{$i}");
            $limitedHistory->snapshot();
        }

        $this->assertSame(3, $limitedHistory->getUndoCount());
    }

    #[Test]
    public function memento_is_immutable(): void
    {
        $this->editor->type('Original');
        $memento = $this->editor->save();

        $this->editor->type(' Modified');

        // Memento-র state পরিবর্তন হওয়া উচিত নয়
        $this->assertSame('Original', $memento->getContent());
    }

    #[Test]
    public function cursor_position_is_preserved(): void
    {
        $this->editor->type('Hello World');
        $this->editor->moveCursor(5);
        $memento = $this->editor->save();

        $this->editor->moveCursor(0);
        $this->editor->restore($memento);

        $this->assertSame(5, $this->editor->getCursorPosition());
    }
}

class GameMementoTest extends TestCase
{
    #[Test]
    public function game_save_and_load_preserves_all_state(): void
    {
        $player = new GameCharacter('টেস্ট', Difficulty::Medium);
        $player->addScore(1000);
        $player->levelUp();
        $player->addItem('তলোয়ার');
        $player->moveTo(100, 200);

        $checkpoint = $player->saveCheckpoint();

        $player->takeDamage(50);
        $player->addScore(500);
        $this->assertSame(50, $player->getHealth());

        $player->loadCheckpoint($checkpoint);
        $this->assertSame(100, $player->getHealth());
        $this->assertSame(1000, $player->getScore());
    }

    #[Test]
    public function save_manager_handles_multiple_slots(): void
    {
        $player = new GameCharacter('Hero');
        $manager = new SaveManager();

        $player->addScore(100);
        $manager->saveToSlot('slot1', $player);

        $player->addScore(200);
        $manager->saveToSlot('slot2', $player);

        $manager->loadFromSlot('slot1', $player);
        $this->assertSame(100, $player->getScore());

        $manager->loadFromSlot('slot2', $player);
        $this->assertSame(300, $player->getScore());
    }

    #[Test]
    public function auto_save_maintains_limit(): void
    {
        $player = new GameCharacter('Hero');
        $manager = new SaveManager(maxAutoSaves: 2);

        for ($i = 0; $i < 5; $i++) {
            $player->addScore(100);
            $manager->autoSave($player);
        }

        $saves = $manager->listSaves();
        $autoSaves = array_filter($saves, fn($s) => str_contains($s, 'Auto'));
        $this->assertCount(2, $autoSaves);
    }
}
```

### Jest (JavaScript)

```javascript
describe('Memento Pattern', () => {
  let editor;
  let history;

  beforeEach(() => {
    editor = new TextEditor();
    history = new EditorHistory(editor);
  });

  describe('TextEditor', () => {
    test('save এবং restore সঠিকভাবে কাজ করে', () => {
      editor.type('হ্যালো');
      const memento = editor.save();

      editor.type(' বিশ্ব');
      expect(editor.getContent()).toBe('হ্যালো বিশ্ব');

      editor.restore(memento);
      expect(editor.getContent()).toBe('হ্যালো');
    });

    test('memento immutable — পরবর্তী পরিবর্তনে প্রভাবিত হয় না', () => {
      editor.type('Original');
      const memento = editor.save();

      editor.type(' Changed');
      expect(memento.getContent()).toBe('Original');
    });

    test('cursor position সংরক্ষিত হয়', () => {
      editor.type('Hello');
      editor.moveCursor(3);
      const memento = editor.save();

      editor.moveCursor(0);
      editor.restore(memento);
      expect(editor.getCursorPosition()).toBe(3);
    });
  });

  describe('EditorHistory', () => {
    test('undo আগের state-এ ফিরে যায়', () => {
      editor.type('First');
      history.snapshot();

      editor.type(' Second');
      history.snapshot();

      history.undo();
      expect(editor.getContent()).toContain('First');
      expect(editor.getContent()).toContain('Second');

      history.undo();
      expect(editor.getContent()).toBe('First');
    });

    test('redo undo-কৃত state পুনরুদ্ধার করে', () => {
      editor.type('A');
      history.snapshot();
      editor.type('B');
      history.snapshot();

      history.undo();
      history.redo();
      expect(editor.getContent()).toContain('B');
    });

    test('নতুন edit-এ redo stack ক্লিয়ার হয়', () => {
      editor.type('A');
      history.snapshot();
      editor.type('B');
      history.snapshot();

      history.undo();
      editor.type('C');
      history.snapshot();

      expect(history.redo()).toBe(false);
    });

    test('খালি stack-এ undo false রিটার্ন করে', () => {
      expect(history.undo()).toBe(false);
    });

    test('max history সীমা মেনে চলে', () => {
      const limitedHistory = new EditorHistory(editor, 3);

      for (let i = 0; i < 10; i++) {
        editor.type(`${i}`);
        limitedHistory.snapshot();
      }

      let undoCount = 0;
      while (limitedHistory.undo()) undoCount++;
      expect(undoCount).toBe(3);
    });
  });

  describe('GameCharacter', () => {
    test('checkpoint সব state সংরক্ষণ করে', () => {
      const player = new GameCharacter('Hero');
      player.addScore(500);
      player.levelUp();
      player.addItem('Sword');

      const checkpoint = player.saveCheckpoint();

      player.takeDamage(80);
      player.addScore(100);
      expect(player.getHealth()).toBe(20);

      player.loadCheckpoint(checkpoint);
      expect(player.getHealth()).toBe(100);
    });

    test('SaveManager একাধিক slot হ্যান্ডল করে', () => {
      const player = new GameCharacter('Hero');
      const manager = new SaveManager();

      player.addScore(100);
      manager.saveToSlot('A', player);

      player.addScore(200);
      manager.saveToSlot('B', player);

      manager.loadFromSlot('A', player);
      expect(player.getHealth()).toBe(100);
    });

    test('অবৈধ slot লোড করলে Error হয়', () => {
      const player = new GameCharacter('Hero');
      const manager = new SaveManager();

      expect(() => manager.loadFromSlot('nonexistent', player))
        .toThrow('সেভ স্লট');
    });
  });

  describe('DeltaTracker', () => {
    test('delta memento শুধু পরিবর্তন সংরক্ষণ করে', () => {
      const tracker = new DeltaTracker({
        name: 'Test',
        data: 'x'.repeat(1000),
        count: 0,
      });

      tracker.update({ count: 1 });
      tracker.update({ count: 2 });

      const usage = tracker.getMemoryUsage();
      expect(usage.totalDeltaSize).toBeLessThan(usage.fullSnapshotSize);
    });

    test('undo সঠিকভাবে কাজ করে', () => {
      const tracker = new DeltaTracker({ value: 'A' });
      tracker.update({ value: 'B' });
      tracker.update({ value: 'C' });

      tracker.undo();
      expect(tracker.getState().value).toBe('B');

      tracker.undo();
      expect(tracker.getState().value).toBe('A');
    });
  });

  describe('structuredClone deep copy', () => {
    test('nested objects সঠিকভাবে copy হয়', () => {
      const state = {
        nested: { deep: { value: 42 } },
        items: [1, 2, 3],
        date: new Date(),
      };

      const memento = new RobustMemento(state);
      state.nested.deep.value = 999;
      state.items.push(4);

      const restored = memento.getState();
      expect(restored.nested.deep.value).toBe(42);
      expect(restored.items).toEqual([1, 2, 3]);
      expect(restored.date).toBeInstanceOf(Date);
    });
  });
});
```

---

## 🔗 সম্পর্কিত প্যাটার্ন

### Command + Memento

| বৈশিষ্ট্য | Command | Memento |
|---|---|---|
| **উদ্দেশ্য** | অপারেশনকে অবজেক্ট হিসেবে encapsulate | State snapshot সংরক্ষণ |
| **Undo কৌশল** | Reverse operation চালায় | আগের state restore করে |
| **যখন ব্যবহার** | অপারেশন reversible হলে | State complex হলে |
| **একসাথে ব্যবহার** | Command execute-এর আগে Memento সেভ করলে নিশ্চিত undo পাওয়া যায় |

### Iterator + Memento

Iterator-এর বর্তমান position সেভ করতে Memento ব্যবহার করা যায়। ফলে iteration-এর মাঝে bookmark রাখা সম্ভব:

```javascript
class IteratorWithBookmark {
  #items;
  #position = 0;

  constructor(items) {
    this.#items = [...items];
  }

  next() {
    if (this.#position >= this.#items.length) return { done: true };
    return { value: this.#items[this.#position++], done: false };
  }

  saveBookmark() {
    return new Memento(this.#position);
  }

  restoreBookmark(memento) {
    this.#position = memento.getState();
  }
}
```

### Prototype + Memento

Prototype প্যাটার্নের `clone()` method Memento তৈরিতে ব্যবহার করা যায়:

```php
<?php
// Prototype clone দিয়ে deep copy memento
class CloneableOriginator implements Cloneable
{
    public function save(): static
    {
        return clone $this; // পুরো অবজেক্টটাই memento!
    }

    public function restore(static $memento): void
    {
        // Clone থেকে state কপি করো
        foreach (get_object_vars($memento) as $prop => $value) {
            $this->$prop = $value;
        }
    }
}
```

---

## 📏 কখন ব্যবহার করবেন / করবেন না

### ✅ ব্যবহার করবেন যখন:

1. **Undo/Redo functionality** প্রয়োজন — টেক্সট এডিটর, ড্রয়িং অ্যাপ
2. **Snapshot/Checkpoint** দরকার — গেম সেভ, ফর্ম ড্রাফট
3. **Transaction Rollback** প্রয়োজন — ব্যর্থ অপারেশনে আগের অবস্থায় ফিরে যাওয়া
4. **Encapsulation** বজায় রাখতে চান — internal state expose করতে চান না
5. **State History** ট্র্যাক করতে চান — অডিট লগ, ডিবাগিং
6. **A/B State Comparison** — দুটি অবস্থা তুলনা করতে

### ❌ ব্যবহার করবেন না যখন:

1. **State অত্যন্ত বড়** — প্রতিটি snapshot-এ বিশাল মেমোরি খরচ (Delta Memento বিবেচনা করুন)
2. **State সরল** — শুধু একটি primitive value হলে overkill
3. **History প্রয়োজন নেই** — শুধু current state দরকার হলে
4. **Real-time system** — যেখানে snapshot-এর latency গ্রহণযোগ্য নয়
5. **Distributed system** — যেখানে state একাধিক node-এ ছড়িয়ে আছে (Event Sourcing ভালো)
6. **Immutable data** — যদি ডেটা কখনো পরিবর্তন না হয় তাহলে memento অপ্রয়োজনীয়

### সিদ্ধান্ত গ্রহণের ফ্লোচার্ট

```
State সংরক্ষণ দরকার?
├── না → Memento লাগবে না
└── হ্যাঁ → Encapsulation গুরুত্বপূর্ণ?
    ├── না → সরাসরি state কপি করুন
    └── হ্যাঁ → State কতটুকু বড়?
        ├── ছোট → Full Memento ✅
        ├── বড় → Delta/Incremental Memento ✅
        └── অত্যন্ত বড় → Compressed Memento বা Event Sourcing বিবেচনা করুন
```

---

## 📋 সারসংক্ষেপ

### মূল পয়েন্ট

```
┌─────────────────────────────────────────────────────────────┐
│                   Memento প্যাটার্ন সারাংশ                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  🎯 উদ্দেশ্য: Encapsulation ভঙ্গ না করে state             │
│              সংরক্ষণ ও পুনরুদ্ধার                          │
│                                                             │
│  👥 অংশগ্রহণকারী:                                          │
│     • Originator → state-এর মালিক, memento তৈরি করে       │
│     • Memento → state-এর immutable snapshot                │
│     • Caretaker → memento সংগ্রহ ও পরিচালনা               │
│                                                             │
│  ✅ কখন ব্যবহার:                                           │
│     • Undo/Redo, Checkpoint, Rollback                      │
│     • State history tracking                               │
│     • Draft/Auto-save systems                              │
│                                                             │
│  ⚠️ সতর্কতা:                                              │
│     • Memory management (history সীমিত রাখুন)             │
│     • Deep copy নিশ্চিত করুন                              │
│     • Redo stack নতুন edit-এ ক্লিয়ার করুন                │
│                                                             │
│  🔗 সম্পর্কিত: Command, Iterator, Prototype               │
│                                                             │
│  💡 টিপস:                                                  │
│     PHP → readonly class + serialize()                     │
│     JS  → #private fields + structuredClone()              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### চিট শিট

| প্রশ্ন | উত্তর |
|---|---|
| **কী করে?** | অবজেক্টের state snapshot নেয় |
| **কেন করে?** | পরে restore করার জন্য (undo) |
| **কে তৈরি করে?** | Originator |
| **কে সংরক্ষণ করে?** | Caretaker |
| **কী সংরক্ষণ হয়?** | Internal state (Memento-তে) |
| **Encapsulation?** | ✅ সম্পূর্ণ সুরক্ষিত |
| **PHP-তে key tool?** | `readonly class`, `serialize()` |
| **JS-এ key tool?** | `#private`, `structuredClone()`, `Object.freeze()` |
| **Memory সমস্যা?** | Delta memento বা compression ব্যবহার করুন |
| **Command-এর সাথে?** | Command execute-এর আগে memento সেভ = নিশ্চিত undo |

---

> **"প্রতিটি ভুল থেকে ফিরে আসার সুযোগ থাকা উচিত — সেটাই Memento প্যাটার্নের দর্শন।"** 💾
