# 📐 Template Method প্যাটার্ন

## 📌 সংজ্ঞা

**Template Method** হলো একটি **Behavioral Design Pattern** যা একটি base class-এ অ্যালগরিদমের **skeleton (কাঠামো)** সংজ্ঞায়িত করে এবং subclass-গুলোকে সেই অ্যালগরিদমের **নির্দিষ্ট step-গুলো override** করতে দেয়, কিন্তু সামগ্রিক কাঠামো পরিবর্তন করতে দেয় না।

> **GoF সংজ্ঞা:** "Define the skeleton of an algorithm in an operation, deferring some steps to subclasses. Template Method lets subclasses redefine certain steps of an algorithm without changing the algorithm's structure."

### মূল ধারণা

Template Method প্যাটার্নের মূল শক্তি হলো **Inversion of Control (IoC)**। সাধারণত আমরা library/framework-কে call করি, কিন্তু এখানে **framework আমাদের code-কে call করে**। এটাই বিখ্যাত **Hollywood Principle** — "Don't call us, we'll call you."

কল্পনা করুন, বাংলাদেশের যেকোনো ব্যাংকের **লোন প্রসেসিং** সিস্টেম। প্রতিটি লোনের ধাপ একই:

```
আবেদন গ্রহণ → যাচাই → অনুমোদন → বিতরণ
```

কিন্তু **হোম লোন**, **পার্সোনাল লোন**, **এডুকেশন লোন** — প্রতিটির যাচাই ও অনুমোদন প্রক্রিয়া আলাদা। Template Method ঠিক এই সমস্যাটি সমাধান করে।

### প্যাটার্নের তিনটি মূল উপাদান

| উপাদান | বর্ণনা |
|---------|--------|
| **Abstract Class** | অ্যালগরিদমের কাঠামো ধারণ করে, `templateMethod()` define করে |
| **Concrete Steps** | Subclass-এ implement হওয়া abstract methods |
| **Hook Methods** | Optional methods যা subclass override করতে পারে (কিন্তু বাধ্যতামূলক নয়) |

---

## 🏠 বাস্তব উদাহরণ

### রান্নার রেসিপি (Recipe Cooking)

ধরুন, আপনি **বিরিয়ানি** এবং **খিচুড়ি** রান্না করছেন। দুটোরই মূল ধাপ একই:

```
১. উপকরণ প্রস্তুত করা (prepareIngredients)
২. তেল/ঘি গরম করা (heatOil)
৩. মশলা ভাজা (frySpices)
৪. প্রধান উপাদান যোগ করা (addMainIngredient)
৫. রান্না করা (cook)
৬. পরিবেশন করা (serve)
```

কিন্তু প্রতিটি ধাপের **বিস্তারিত** আলাদা — বিরিয়ানিতে দম দিতে হয়, খিচুড়িতে দিতে হয় না। Template Method এই common structure রাখে base class-এ, আর variation-গুলো subclass-এ।

### বাংলাদেশের পেমেন্ট গেটওয়ে

bKash, Nagad, Rocket — সবগুলোর payment flow একই:

```
OTP পাঠাও → যাচাই করো → টাকা কাটো → রশিদ দাও → নোটিফিকেশন পাঠাও
```

কিন্তু প্রতিটি সার্ভিসের implementation আলাদা। এটি Template Method-এর perfect use case।

---

## 📊 UML ডায়াগ্রাম

```
┌─────────────────────────────────────────┐
│        <<abstract>>                     │
│        AbstractClass                    │
├─────────────────────────────────────────┤
│ + templateMethod(): void    [final]     │
│ # primitiveOperation1(): void [abstract]│
│ # primitiveOperation2(): void [abstract]│
│ # hook(): bool              [virtual]   │
├─────────────────────────────────────────┤
│ templateMethod() {                      │
│   primitiveOperation1();                │
│   if (hook()) {                         │
│     primitiveOperation2();              │
│   }                                     │
│   commonOperation();                    │
│ }                                       │
└─────────────┬───────────────────────────┘
              │ extends
     ┌────────┴────────┐
     │                 │
┌────▼─────┐    ┌─────▼────┐
│ConcreteA  │    │ConcreteB │
├──────────┤    ├──────────┤
│#op1()    │    │#op1()    │
│#op2()    │    │#op2()    │
│#hook()   │    │          │
└──────────┘    └──────────┘
```

### ফ্লো ডায়াগ্রাম

```
Client
  │
  ▼
templateMethod() ─── [FINAL - পরিবর্তনযোগ্য নয়]
  │
  ├──► step1()  ──── [Abstract - subclass অবশ্যই implement করবে]
  │
  ├──► step2()  ──── [Abstract - subclass অবশ্যই implement করবে]
  │
  ├──► hook()   ──── [Optional - default আছে, override করা যায়]
  │     │
  │     ├── true  ──► step3()
  │     └── false ──► skip
  │
  └──► step4()  ──── [Concrete - base class-এ fixed]
```

---

## 💻 ইমপ্লিমেন্টেশন

### ১. Basic Template Method

#### PHP 8.3

```php
<?php

declare(strict_types=1);

abstract class DocumentGenerator
{
    // Template Method — final তাই subclass override করতে পারবে না
    final public function generate(string $title): string
    {
        $content = $this->createHeader($title);
        $content .= $this->createBody();

        if ($this->hasFooter()) { // Hook method
            $content .= $this->createFooter();
        }

        $this->log("Document generated: {$title}");

        return $content;
    }

    abstract protected function createHeader(string $title): string;
    abstract protected function createBody(): string;

    // Hook method — default implementation আছে, override optional
    protected function hasFooter(): bool
    {
        return true;
    }

    protected function createFooter(): string
    {
        return "\n--- Generated at " . date('Y-m-d H:i:s') . " ---";
    }

    // Concrete method — সব subclass-এ একই
    private function log(string $message): void
    {
        echo "[LOG] {$message}\n";
    }
}

class HtmlDocument extends DocumentGenerator
{
    protected function createHeader(string $title): string
    {
        return "<html><head><title>{$title}</title></head><body>\n";
    }

    protected function createBody(): string
    {
        return "<h1>HTML Document Content</h1>\n<p>This is an HTML document.</p>\n";
    }

    protected function createFooter(): string
    {
        return "</body></html>";
    }
}

class MarkdownDocument extends DocumentGenerator
{
    protected function createHeader(string $title): string
    {
        return "# {$title}\n\n";
    }

    protected function createBody(): string
    {
        return "## Content\n\nThis is a **Markdown** document.\n";
    }

    // Footer hook override — Markdown-এ footer চাই না
    protected function hasFooter(): bool
    {
        return false;
    }
}

// ব্যবহার
$html = new HtmlDocument();
echo $html->generate("আমার ডকুমেন্ট");

$md = new MarkdownDocument();
echo $md->generate("README");
```

#### JavaScript (ES2022+)

```javascript
class DocumentGenerator {
    // Template Method
    generate(title) {
        // Private method simulation — subclass override করতে পারবে না
        if (new.target === DocumentGenerator) {
            throw new Error('Abstract class cannot be instantiated');
        }

        let content = this.createHeader(title);
        content += this.createBody();

        if (this.hasFooter()) {
            content += this.createFooter();
        }

        this.#log(`Document generated: ${title}`);
        return content;
    }

    createHeader(title) {
        throw new Error('Subclass must implement createHeader()');
    }

    createBody() {
        throw new Error('Subclass must implement createBody()');
    }

    // Hook method
    hasFooter() {
        return true;
    }

    createFooter() {
        return `\n--- Generated at ${new Date().toISOString()} ---`;
    }

    // Private method — ES2022 private field
    #log(message) {
        console.log(`[LOG] ${message}`);
    }
}

class HtmlDocument extends DocumentGenerator {
    createHeader(title) {
        return `<html><head><title>${title}</title></head><body>\n`;
    }

    createBody() {
        return `<h1>HTML Document Content</h1>\n<p>This is an HTML document.</p>\n`;
    }

    createFooter() {
        return `</body></html>`;
    }
}

class MarkdownDocument extends DocumentGenerator {
    createHeader(title) {
        return `# ${title}\n\n`;
    }

    createBody() {
        return `## Content\n\nThis is a **Markdown** document.\n`;
    }

    hasFooter() {
        return false;
    }
}

// ব্যবহার
const html = new HtmlDocument();
console.log(html.generate('আমার ডকুমেন্ট'));

const md = new MarkdownDocument();
console.log(md.generate('README'));
```

---

### ২. Data Import Pipeline (CSV / JSON / XML)

এটি একটি অত্যন্ত practical use case। বাংলাদেশের অনেক প্রতিষ্ঠানে বিভিন্ন ফরম্যাটে ডেটা আসে — CSV, JSON, XML। প্রতিটির import pipeline একই কাঠামো অনুসরণ করে: **parse → validate → transform → save**।

#### PHP 8.3

```php
<?php

declare(strict_types=1);

// DTO for imported records
readonly class ImportRecord
{
    public function __construct(
        public string $id,
        public array $data,
        public bool $isValid = true,
        public string $error = '',
    ) {}
}

// Enum for import status
enum ImportStatus: string
{
    case Success = 'success';
    case PartialSuccess = 'partial_success';
    case Failed = 'failed';
}

readonly class ImportResult
{
    public function __construct(
        public ImportStatus $status,
        public int $totalRecords,
        public int $importedRecords,
        public int $failedRecords,
        public array $errors = [],
    ) {}
}

abstract class DataImporter
{
    private array $errors = [];

    final public function import(string $source): ImportResult
    {
        $this->beforeImport($source); // Hook

        $rawData = $this->parse($source);
        $validatedData = $this->validate($rawData);
        $transformedData = $this->transform($validatedData);
        $savedCount = $this->save($transformedData);

        $this->afterImport($savedCount); // Hook

        $total = count($rawData);
        $failed = $total - $savedCount;

        return new ImportResult(
            status: match(true) {
                $savedCount === $total => ImportStatus::Success,
                $savedCount > 0 => ImportStatus::PartialSuccess,
                default => ImportStatus::Failed,
            },
            totalRecords: $total,
            importedRecords: $savedCount,
            failedRecords: $failed,
            errors: $this->errors,
        );
    }

    // Abstract steps — subclass অবশ্যই implement করবে
    abstract protected function parse(string $source): array;
    abstract protected function validate(array $records): array;
    abstract protected function transform(array $records): array;
    abstract protected function save(array $records): int;

    // Hook methods — optional override
    protected function beforeImport(string $source): void
    {
        echo "Starting import from: {$source}\n";
    }

    protected function afterImport(int $count): void
    {
        echo "Import completed. {$count} records saved.\n";
    }

    protected function addError(string $error): void
    {
        $this->errors[] = $error;
    }
}

class CsvImporter extends DataImporter
{
    protected function parse(string $source): array
    {
        $records = [];
        $lines = explode("\n", trim($source));
        $headers = str_getcsv(array_shift($lines));

        foreach ($lines as $index => $line) {
            if (empty(trim($line))) continue;
            $values = str_getcsv($line);
            $records[] = new ImportRecord(
                id: (string)($index + 1),
                data: array_combine($headers, $values),
            );
        }

        return $records;
    }

    protected function validate(array $records): array
    {
        return array_filter($records, function (ImportRecord $record) {
            if (empty($record->data['name'] ?? '')) {
                $this->addError("Record {$record->id}: name is required");
                return false;
            }
            return true;
        });
    }

    protected function transform(array $records): array
    {
        return array_map(fn(ImportRecord $r) => new ImportRecord(
            id: $r->id,
            data: [
                ...$r->data,
                'name' => mb_convert_case($r->data['name'], MB_CASE_TITLE, 'UTF-8'),
                'imported_at' => date('Y-m-d H:i:s'),
            ],
        ), $records);
    }

    protected function save(array $records): int
    {
        // বাস্তবে এখানে database save হবে
        foreach ($records as $record) {
            echo "  Saved: {$record->data['name']}\n";
        }
        return count($records);
    }
}

class JsonImporter extends DataImporter
{
    protected function parse(string $source): array
    {
        $data = json_decode($source, true, 512, JSON_THROW_ON_ERROR);
        return array_map(
            fn(array $item, int $i) => new ImportRecord(
                id: $item['id'] ?? (string)($i + 1),
                data: $item,
            ),
            $data,
            array_keys($data),
        );
    }

    protected function validate(array $records): array
    {
        return array_filter($records, function (ImportRecord $record) {
            $requiredFields = ['name', 'email'];
            foreach ($requiredFields as $field) {
                if (empty($record->data[$field] ?? '')) {
                    $this->addError("Record {$record->id}: {$field} is required");
                    return false;
                }
            }
            return true;
        });
    }

    protected function transform(array $records): array
    {
        return array_map(fn(ImportRecord $r) => new ImportRecord(
            id: $r->id,
            data: [
                ...$r->data,
                'email' => strtolower($r->data['email']),
                'source' => 'json_import',
            ],
        ), $records);
    }

    protected function save(array $records): int
    {
        foreach ($records as $record) {
            echo "  Saved JSON record: {$record->data['name']}\n";
        }
        return count($records);
    }

    // Hook override — JSON-এ আগে schema validate করি
    protected function beforeImport(string $source): void
    {
        parent::beforeImport($source);
        echo "  Validating JSON schema...\n";
    }
}

// ব্যবহার
$csvData = "name,email,phone\nরহিম,rahim@example.com,01711111111\nকরিম,karim@example.com,01722222222";
$csvImporter = new CsvImporter();
$result = $csvImporter->import($csvData);
echo "Status: {$result->status->value}, Imported: {$result->importedRecords}/{$result->totalRecords}\n\n";

$jsonData = json_encode([
    ['id' => '1', 'name' => 'ফারহানা', 'email' => 'FARHANA@test.com'],
    ['id' => '2', 'name' => 'আরিফ', 'email' => 'arif@test.com'],
]);
$jsonImporter = new JsonImporter();
$result = $jsonImporter->import($jsonData);
echo "Status: {$result->status->value}\n";
```

#### JavaScript (ES2022+)

```javascript
class DataImporter {
    #errors = [];

    async import(source) {
        if (new.target === DataImporter) {
            throw new Error('Cannot instantiate abstract DataImporter');
        }

        await this.beforeImport(source);

        const rawData = await this.parse(source);
        const validated = this.validate(rawData);
        const transformed = this.transform(validated);
        const savedCount = await this.save(transformed);

        await this.afterImport(savedCount);

        const total = rawData.length;
        const status = savedCount === total ? 'success'
            : savedCount > 0 ? 'partial_success'
            : 'failed';

        return {
            status,
            totalRecords: total,
            importedRecords: savedCount,
            failedRecords: total - savedCount,
            errors: [...this.#errors],
        };
    }

    // Abstract methods
    async parse(source) { throw new Error('Must implement parse()'); }
    validate(records) { throw new Error('Must implement validate()'); }
    transform(records) { throw new Error('Must implement transform()'); }
    async save(records) { throw new Error('Must implement save()'); }

    // Hook methods
    async beforeImport(source) {
        console.log(`Starting import from source...`);
    }

    async afterImport(count) {
        console.log(`Import completed. ${count} records saved.`);
    }

    addError(error) {
        this.#errors.push(error);
    }
}

class CsvImporter extends DataImporter {
    async parse(source) {
        const lines = source.trim().split('\n');
        const headers = lines.shift().split(',');

        return lines.filter(l => l.trim()).map((line, i) => {
            const values = line.split(',');
            const data = Object.fromEntries(
                headers.map((h, idx) => [h.trim(), values[idx]?.trim()])
            );
            return { id: String(i + 1), data };
        });
    }

    validate(records) {
        return records.filter(record => {
            if (!record.data.name) {
                this.addError(`Record ${record.id}: name is required`);
                return false;
            }
            return true;
        });
    }

    transform(records) {
        return records.map(r => ({
            ...r,
            data: {
                ...r.data,
                name: r.data.name.charAt(0).toUpperCase() + r.data.name.slice(1),
                imported_at: new Date().toISOString(),
            },
        }));
    }

    async save(records) {
        for (const record of records) {
            console.log(`  Saved: ${record.data.name}`);
        }
        return records.length;
    }
}

class JsonImporter extends DataImporter {
    async parse(source) {
        const data = JSON.parse(source);
        return data.map((item, i) => ({
            id: item.id ?? String(i + 1),
            data: item,
        }));
    }

    validate(records) {
        return records.filter(record => {
            for (const field of ['name', 'email']) {
                if (!record.data[field]) {
                    this.addError(`Record ${record.id}: ${field} is required`);
                    return false;
                }
            }
            return true;
        });
    }

    transform(records) {
        return records.map(r => ({
            ...r,
            data: { ...r.data, email: r.data.email.toLowerCase(), source: 'json_import' },
        }));
    }

    async save(records) {
        for (const r of records) console.log(`  Saved JSON: ${r.data.name}`);
        return records.length;
    }

    async beforeImport(source) {
        await super.beforeImport(source);
        console.log('  Validating JSON schema...');
    }
}

// ব্যবহার
const csvImporter = new CsvImporter();
const csvResult = await csvImporter.import(
    'name,email,phone\nরহিম,rahim@test.com,01711111111\nকরিম,karim@test.com,01722222222'
);
console.log('CSV Result:', csvResult);
```

---

### ৩. Report Generation System

বাংলাদেশের যেকোনো এন্টারপ্রাইজ সফটওয়্যারে রিপোর্ট জেনারেশন অত্যন্ত গুরুত্বপূর্ণ। বিক্রয় রিপোর্ট, আর্থিক রিপোর্ট, স্টক রিপোর্ট — সবার কাঠামো একই: **fetchData → processData → formatReport → export**।

#### PHP 8.3

```php
<?php

declare(strict_types=1);

enum ReportFormat: string
{
    case PDF = 'pdf';
    case Excel = 'excel';
    case HTML = 'html';
}

abstract class ReportGenerator
{
    final public function generateReport(
        \DateTimeImmutable $startDate,
        \DateTimeImmutable $endDate,
    ): string {
        echo "📊 Generating report: {$this->getReportName()}\n";

        $rawData = $this->fetchData($startDate, $endDate);

        if (empty($rawData)) {
            return $this->handleEmptyReport(); // Hook
        }

        $processedData = $this->processData($rawData);
        $aggregated = $this->aggregate($processedData); // Hook — default empty
        $formatted = $this->formatReport(
            empty($aggregated) ? $processedData : $aggregated
        );

        if ($this->shouldExport()) { // Hook
            $this->export($formatted);
        }

        return $formatted;
    }

    // Abstract methods
    abstract protected function getReportName(): string;
    abstract protected function fetchData(
        \DateTimeImmutable $start,
        \DateTimeImmutable $end,
    ): array;
    abstract protected function processData(array $rawData): array;
    abstract protected function formatReport(array $data): string;

    // Hook methods
    protected function aggregate(array $data): array
    {
        return []; // Default: কোনো aggregation নেই
    }

    protected function shouldExport(): bool
    {
        return false;
    }

    protected function export(string $content): void
    {
        // Default implementation
    }

    protected function handleEmptyReport(): string
    {
        return "No data found for the specified period.";
    }
}

class SalesReport extends ReportGenerator
{
    protected function getReportName(): string
    {
        return 'মাসিক বিক্রয় রিপোর্ট';
    }

    protected function fetchData(
        \DateTimeImmutable $start,
        \DateTimeImmutable $end,
    ): array {
        // বাস্তবে DB query হবে
        return [
            ['product' => 'ল্যাপটপ', 'quantity' => 45, 'revenue' => 4500000],
            ['product' => 'মোবাইল', 'quantity' => 230, 'revenue' => 6900000],
            ['product' => 'ট্যাবলেট', 'quantity' => 80, 'revenue' => 2400000],
        ];
    }

    protected function processData(array $rawData): array
    {
        return array_map(fn(array $row) => [
            ...$row,
            'avg_price' => $row['revenue'] / $row['quantity'],
            'revenue_formatted' => number_format($row['revenue']) . ' ৳',
        ], $rawData);
    }

    protected function aggregate(array $data): array
    {
        $totalRevenue = array_sum(array_column($data, 'revenue'));
        $totalQty = array_sum(array_column($data, 'quantity'));

        return [
            'items' => $data,
            'summary' => [
                'total_revenue' => number_format($totalRevenue) . ' ৳',
                'total_quantity' => $totalQty,
                'avg_order_value' => number_format($totalRevenue / $totalQty) . ' ৳',
            ],
        ];
    }

    protected function formatReport(array $data): string
    {
        $output = "═══ মাসিক বিক্রয় রিপোর্ট ═══\n\n";

        foreach ($data['items'] as $item) {
            $output .= "পণ্য: {$item['product']} | পরিমাণ: {$item['quantity']} | আয়: {$item['revenue_formatted']}\n";
        }

        $output .= "\n── সারসংক্ষেপ ──\n";
        $output .= "মোট আয়: {$data['summary']['total_revenue']}\n";
        $output .= "মোট বিক্রি: {$data['summary']['total_quantity']}\n";

        return $output;
    }

    protected function shouldExport(): bool
    {
        return true;
    }

    protected function export(string $content): void
    {
        echo "📁 Exported to: sales_report_" . date('Y_m') . ".pdf\n";
    }
}

class FinancialReport extends ReportGenerator
{
    protected function getReportName(): string
    {
        return 'আর্থিক প্রতিবেদন';
    }

    protected function fetchData(
        \DateTimeImmutable $start,
        \DateTimeImmutable $end,
    ): array {
        return [
            ['category' => 'আয়', 'amount' => 15000000],
            ['category' => 'ব্যয়', 'amount' => 8500000],
            ['category' => 'কর', 'amount' => 1950000],
        ];
    }

    protected function processData(array $rawData): array
    {
        $totalIncome = $rawData[0]['amount'];
        $totalExpense = $rawData[1]['amount'];
        $tax = $rawData[2]['amount'];

        return [
            'income' => number_format($totalIncome) . ' ৳',
            'expense' => number_format($totalExpense) . ' ৳',
            'tax' => number_format($tax) . ' ৳',
            'net_profit' => number_format($totalIncome - $totalExpense - $tax) . ' ৳',
        ];
    }

    protected function formatReport(array $data): string
    {
        return "═══ আর্থিক প্রতিবেদন ═══\n"
            . "আয়: {$data['income']}\n"
            . "ব্যয়: {$data['expense']}\n"
            . "কর: {$data['tax']}\n"
            . "নিট লাভ: {$data['net_profit']}\n";
    }
}

// ব্যবহার
$salesReport = new SalesReport();
echo $salesReport->generateReport(
    new \DateTimeImmutable('2024-01-01'),
    new \DateTimeImmutable('2024-01-31'),
);
```

---

### ৪. Authentication Flow

#### PHP 8.3

```php
<?php

declare(strict_types=1);

readonly class AuthResult
{
    public function __construct(
        public bool $success,
        public ?string $token = null,
        public ?string $redirectUrl = null,
        public string $error = '',
    ) {}
}

abstract class AuthenticationFlow
{
    final public function authenticate(array $request): AuthResult
    {
        try {
            $credentials = $this->extractCredentials($request);

            if (!$this->validateCredentials($credentials)) {
                $this->onAuthFailure($credentials); // Hook
                return new AuthResult(success: false, error: 'Invalid credentials');
            }

            if ($this->requiresTwoFactor()) { // Hook
                if (!$this->verifyTwoFactor($request)) {
                    return new AuthResult(success: false, error: '2FA verification failed');
                }
            }

            $session = $this->createSession($credentials);
            $this->onAuthSuccess($session); // Hook
            $redirectUrl = $this->getRedirectUrl($credentials);

            return new AuthResult(
                success: true,
                token: $session['token'],
                redirectUrl: $redirectUrl,
            );
        } catch (\Throwable $e) {
            return new AuthResult(success: false, error: $e->getMessage());
        }
    }

    // Abstract methods
    abstract protected function extractCredentials(array $request): array;
    abstract protected function validateCredentials(array $credentials): bool;
    abstract protected function createSession(array $credentials): array;

    // Hook methods
    protected function requiresTwoFactor(): bool
    {
        return false;
    }

    protected function verifyTwoFactor(array $request): bool
    {
        return true;
    }

    protected function getRedirectUrl(array $credentials): string
    {
        return '/dashboard';
    }

    protected function onAuthSuccess(array $session): void {}
    protected function onAuthFailure(array $credentials): void {}
}

class EmailPasswordAuth extends AuthenticationFlow
{
    protected function extractCredentials(array $request): array
    {
        return [
            'email' => $request['email'] ?? '',
            'password' => $request['password'] ?? '',
        ];
    }

    protected function validateCredentials(array $credentials): bool
    {
        // বাস্তবে database check হবে
        return !empty($credentials['email']) && !empty($credentials['password']);
    }

    protected function createSession(array $credentials): array
    {
        return [
            'token' => bin2hex(random_bytes(32)),
            'user_email' => $credentials['email'],
            'created_at' => time(),
        ];
    }

    protected function requiresTwoFactor(): bool
    {
        return true; // Email auth-এ 2FA বাধ্যতামূলক
    }

    protected function verifyTwoFactor(array $request): bool
    {
        $otp = $request['otp'] ?? '';
        return strlen($otp) === 6 && is_numeric($otp);
    }
}

class OAuthAuthentication extends AuthenticationFlow
{
    public function __construct(
        private readonly string $provider, // 'google', 'facebook', 'github'
    ) {}

    protected function extractCredentials(array $request): array
    {
        return [
            'provider' => $this->provider,
            'code' => $request['code'] ?? '',
            'state' => $request['state'] ?? '',
        ];
    }

    protected function validateCredentials(array $credentials): bool
    {
        return !empty($credentials['code']) && !empty($credentials['state']);
    }

    protected function createSession(array $credentials): array
    {
        return [
            'token' => bin2hex(random_bytes(32)),
            'provider' => $credentials['provider'],
            'created_at' => time(),
        ];
    }

    protected function getRedirectUrl(array $credentials): string
    {
        return "/welcome?provider={$credentials['provider']}";
    }
}

// ব্যবহার
$emailAuth = new EmailPasswordAuth();
$result = $emailAuth->authenticate([
    'email' => 'user@example.com',
    'password' => 'secure123',
    'otp' => '123456',
]);
echo $result->success ? "✅ Token: {$result->token}\n" : "❌ Error: {$result->error}\n";

$oAuth = new OAuthAuthentication('google');
$result = $oAuth->authenticate(['code' => 'abc123', 'state' => 'xyz789']);
echo "Redirect to: {$result->redirectUrl}\n";
```

---

### ৫. Game AI Turn System

#### JavaScript (ES2022+)

```javascript
class GameAI {
    #turnLog = [];

    executeTurn(gameState) {
        if (new.target === GameAI) {
            throw new Error('Cannot instantiate abstract GameAI');
        }

        const analysis = this.analyzeBoard(gameState);
        const plan = this.planStrategy(analysis);
        const actions = this.executeActions(plan, gameState);
        const evaluation = this.evaluateOutcome(actions, gameState);

        this.#turnLog.push({ analysis, plan, actions, evaluation });

        if (this.shouldAdaptStrategy()) {
            this.adaptStrategy(evaluation);
        }

        return { actions, evaluation };
    }

    // Abstract methods
    analyzeBoard(state) { throw new Error('Must implement analyzeBoard'); }
    planStrategy(analysis) { throw new Error('Must implement planStrategy'); }
    executeActions(plan, state) { throw new Error('Must implement executeActions'); }
    evaluateOutcome(actions, state) { throw new Error('Must implement evaluateOutcome'); }

    // Hook methods
    shouldAdaptStrategy() { return false; }
    adaptStrategy(evaluation) {}

    get turnHistory() { return [...this.#turnLog]; }
}

class AggressiveAI extends GameAI {
    analyzeBoard(state) {
        return {
            enemyPositions: state.enemies?.map(e => e.position) ?? [],
            weakestEnemy: state.enemies?.sort((a, b) => a.health - b.health)[0],
            myStrength: state.player?.attack ?? 10,
        };
    }

    planStrategy(analysis) {
        if (!analysis.weakestEnemy) return { type: 'patrol', target: null };
        return {
            type: 'attack',
            target: analysis.weakestEnemy,
            estimatedDamage: analysis.myStrength * 1.5,
        };
    }

    executeActions(plan, state) {
        if (plan.type === 'attack' && plan.target) {
            return [{
                action: 'attack',
                target: plan.target.id,
                damage: plan.estimatedDamage,
                position: plan.target.position,
            }];
        }
        return [{ action: 'patrol', direction: 'forward' }];
    }

    evaluateOutcome(actions, state) {
        const totalDamage = actions.reduce((sum, a) => sum + (a.damage ?? 0), 0);
        return { success: totalDamage > 0, totalDamage, rating: totalDamage > 20 ? 'excellent' : 'good' };
    }
}

class DefensiveAI extends GameAI {
    #adaptationCount = 0;

    analyzeBoard(state) {
        return {
            threats: state.enemies?.filter(e => e.distance < 5) ?? [],
            safeZones: state.safeZones ?? [],
            health: state.player?.health ?? 100,
        };
    }

    planStrategy(analysis) {
        if (analysis.health < 30) {
            return { type: 'retreat', destination: analysis.safeZones[0] };
        }
        if (analysis.threats.length > 2) {
            return { type: 'defend', formation: 'turtle' };
        }
        return { type: 'hold', formation: 'standard' };
    }

    executeActions(plan, state) {
        return [{ action: plan.type, details: plan }];
    }

    evaluateOutcome(actions, state) {
        return {
            survived: state.player?.health > 0,
            healthRemaining: state.player?.health ?? 0,
        };
    }

    shouldAdaptStrategy() { return true; }

    adaptStrategy(evaluation) {
        this.#adaptationCount++;
        if (!evaluation.survived) {
            console.log(`🔄 Adapting strategy (attempt ${this.#adaptationCount})`);
        }
    }
}

// ব্যবহার
const aggressive = new AggressiveAI();
const result = aggressive.executeTurn({
    player: { attack: 25, health: 80 },
    enemies: [
        { id: 'e1', position: [3, 5], health: 15, distance: 3 },
        { id: 'e2', position: [7, 2], health: 40, distance: 6 },
    ],
});
console.log('Turn result:', JSON.stringify(result, null, 2));
```

---

## 🌍 Real-World Applicable Areas

### Laravel: FormRequest

Laravel-এর `FormRequest` হলো Template Method-এর চমৎকার উদাহরণ:

```php
// Laravel FormRequest — framework এর template method pattern
abstract class FormRequest extends Request
{
    // Template Method (simplified)
    public function validateResolved()
    {
        $this->prepareForValidation();     // Hook
        
        if (!$this->passesAuthorization()) {
            $this->failedAuthorization();
        }
        
        $validator = $this->getValidatorInstance();
        
        if ($validator->fails()) {
            $this->failedValidation($validator);
        }
        
        $this->passedValidation();         // Hook
    }
}

// আপনার কোড — শুধু steps implement করেন
class StoreProductRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('create-product');
    }

    public function rules(): array
    {
        return [
            'name' => 'required|string|max:255',
            'price' => 'required|numeric|min:0',
            'category_id' => 'required|exists:categories,id',
        ];
    }

    public function messages(): array
    {
        return [
            'name.required' => 'পণ্যের নাম আবশ্যক',
            'price.min' => 'মূল্য ০ এর কম হতে পারে না',
        ];
    }

    // Hook override
    protected function prepareForValidation(): void
    {
        $this->merge([
            'slug' => Str::slug($this->name),
        ]);
    }
}
```

### PHPUnit: setUp → test → tearDown

```php
// PHPUnit-এর TestCase হলো Template Method
abstract class TestCase
{
    // Template Method — framework control করে
    final public function runTest(): void
    {
        $this->setUp();              // Hook — আপনি override করেন
        $this->setUpBeforeClass();   // Hook
        
        try {
            $this->{$this->name}();  // আপনার test method
        } finally {
            $this->tearDown();       // Hook — cleanup
        }
    }
}
```

### React Component Lifecycle

```javascript
// React class component lifecycle = Template Method
// React (framework) নিয়ন্ত্রণ করে কখন কোন method call হবে
class ProductList extends React.Component {
    // Step 1: mount হওয়ার পর
    componentDidMount() {
        this.fetchProducts();
    }

    // Step 2: update হলে
    componentDidUpdate(prevProps) {
        if (prevProps.categoryId !== this.props.categoryId) {
            this.fetchProducts();
        }
    }

    // Step 3: unmount হওয়ার আগে cleanup
    componentWillUnmount() {
        this.controller?.abort();
    }

    // Step 4: render — must implement
    render() {
        return <div>{/* UI */}</div>;
    }
}
```

### Express Middleware Pattern

```javascript
// Express middleware chain — একধরনের Template Method
class RequestPipeline {
    #middlewares = [];

    use(middleware) {
        this.#middlewares.push(middleware);
        return this;
    }

    async execute(req, res) {
        // Template: authenticate → authorize → validate → handle → respond
        for (const middleware of this.#middlewares) {
            const result = await middleware(req, res);
            if (result?.stop) break;
        }
    }
}

const pipeline = new RequestPipeline();
pipeline
    .use(async (req) => { /* authenticate */ })
    .use(async (req) => { /* authorize */ })
    .use(async (req) => { /* validate */ })
    .use(async (req, res) => { /* handle & respond */ });
```

### ETL Pipeline

```php
// ETL = Extract, Transform, Load — natural Template Method
abstract class ETLPipeline
{
    final public function run(string $sourceConfig): array
    {
        $extracted = $this->extract($sourceConfig);
        $transformed = $this->transform($extracted);
        $loadResult = $this->load($transformed);

        $this->notify($loadResult); // Hook

        return $loadResult;
    }

    abstract protected function extract(string $config): array;
    abstract protected function transform(array $data): array;
    abstract protected function load(array $data): array;

    protected function notify(array $result): void {}
}
```

### Build Process

```javascript
class BuildPipeline {
    async build(config) {
        await this.clean(config);
        await this.compile(config);
        
        if (this.shouldRunTests()) {
            await this.test(config);
        }
        
        await this.package(config);
        
        if (this.shouldDeploy()) {
            await this.deploy(config);
        }
    }

    async clean(config) { throw new Error('Implement clean()'); }
    async compile(config) { throw new Error('Implement compile()'); }
    async test(config) { throw new Error('Implement test()'); }
    async package(config) { throw new Error('Implement package()'); }
    async deploy(config) { throw new Error('Implement deploy()'); }

    shouldRunTests() { return true; }
    shouldDeploy() { return false; }
}
```

---

## 🔥 Advanced Deep Dive

### Template Method vs Strategy Pattern

এই দুটি প্যাটার্নের মধ্যে তুলনা অত্যন্ত গুরুত্বপূর্ণ কারণ উভয়ই অ্যালগরিদমের variation handle করে, কিন্তু সম্পূর্ণ ভিন্ন উপায়ে:

```
┌──────────────────────┬───────────────────────────────────┐
│    Template Method    │          Strategy                 │
├──────────────────────┼───────────────────────────────────┤
│ Inheritance ভিত্তিক  │ Composition ভিত্তিক              │
│ Compile-time binding │ Runtime binding                   │
│ "IS-A" সম্পর্ক      │ "HAS-A" সম্পর্ক                  │
│ পুরো algo কাঠামো fix│ পুরো algo বদলানো যায়             │
│ Parent control করে   │ Client control করে                │
│ Step-level variation │ Algorithm-level variation          │
│ Tight coupling       │ Loose coupling                    │
└──────────────────────┴───────────────────────────────────┘
```

#### তুলনামূলক কোড

```php
<?php

// Template Method approach — inheritance
abstract class Sorter
{
    final public function sort(array &$data): void
    {
        $this->prepare($data);
        $this->doSort($data);
        $this->postProcess($data);
    }

    abstract protected function doSort(array &$data): void;

    protected function prepare(array &$data): void {}
    protected function postProcess(array &$data): void {}
}

class QuickSorter extends Sorter
{
    protected function doSort(array &$data): void
    {
        sort($data); // Quick sort implementation
    }
}

// Strategy approach — composition
class SortableCollection
{
    public function __construct(
        private SortStrategy $strategy,
    ) {}

    public function setStrategy(SortStrategy $strategy): void
    {
        $this->strategy = $strategy; // Runtime-এ বদলানো যায়!
    }

    public function sort(array &$data): void
    {
        $this->strategy->sort($data);
    }
}

interface SortStrategy
{
    public function sort(array &$data): void;
}
```

**কখন কোনটি ব্যবহার করবেন:**
- **Template Method:** যখন অ্যালগরিদমের কাঠামো fixed এবং শুধু কিছু step vary করে
- **Strategy:** যখন পুরো অ্যালগরিদম runtime-এ বদলাতে চান

### Hook Method vs Abstract Method

```
┌────────────────────────┬──────────────────────────┐
│    Abstract Method     │     Hook Method           │
├────────────────────────┼──────────────────────────┤
│ MUST override          │ CAN override              │
│ কোনো default নেই       │ Default implementation আছে│
│ Subclass বাধ্য         │ Subclass-এর ইচ্ছা        │
│ Algorithm-এর মূল step │ Optional customization    │
│ abstract keyword       │ virtual/protected method  │
└────────────────────────┴──────────────────────────┘
```

```php
abstract class PaymentProcessor
{
    final public function process(float $amount): bool
    {
        $this->validateAmount($amount);     // Abstract — must implement
        $this->deductBalance($amount);      // Abstract — must implement

        if ($this->shouldSendReceipt()) {   // Hook — optional
            $this->sendReceipt($amount);    // Hook — has default
        }

        $this->logTransaction($amount);     // Concrete — fixed

        return true;
    }

    // Abstract — subclass অবশ্যই implement করবে
    abstract protected function validateAmount(float $amount): void;
    abstract protected function deductBalance(float $amount): void;

    // Hook — default আছে, override optional
    protected function shouldSendReceipt(): bool
    {
        return true;
    }

    protected function sendReceipt(float $amount): void
    {
        echo "Receipt sent for ৳{$amount}\n";
    }

    // Concrete — সব subclass-এ একই
    private function logTransaction(float $amount): void
    {
        echo "[LOG] Transaction: ৳{$amount}\n";
    }
}
```

### Hollywood Principle — "Don't call us, we'll call you"

এটি Template Method-এর সবচেয়ে গুরুত্বপূর্ণ নীতি। সাধারণ প্রোগ্রামিংয়ে আমরা library-কে call করি। কিন্তু Template Method-এ **framework আমাদের code-কে call করে**।

```
সাধারণ পদ্ধতি (আপনি control করেন):
┌──────────┐         ┌───────────┐
│ Your Code│ ──call──► │  Library  │
└──────────┘         └───────────┘

Hollywood Principle (Framework control করে):
┌───────────┐         ┌──────────┐
│ Framework │ ──call──► │Your Code │
└───────────┘         └──────────┘

Template Method-এ:
┌─────────────────┐     ┌────────────────┐
│ Abstract Class   │     │ Your Subclass   │
│ templateMethod() │────►│ step1()         │
│   calls step1()  │     │ step2()         │
│   calls step2()  │     │ hook()          │
└─────────────────┘     └────────────────┘
```

Laravel-এ এটি সবসময় ঘটে:
- আপনি `Controller`-এ method লেখেন, Laravel call করে
- আপনি `FormRequest`-এ rules লেখেন, Laravel validate করে
- আপনি `Migration`-এ `up()` লেখেন, Artisan execute করে

### Template Method in Frameworks

প্রতিটি framework আসলে একটি বিশাল Template Method:

```
Framework = Template Method
Your Code = Concrete Steps

Laravel Request Lifecycle:
┌─────────────────────────────────────────────┐
│ index.php (Framework controls)              │
│  ├── Bootstrap Application                  │
│  ├── Load Service Providers                 │
│  ├── Boot Service Providers                 │
│  ├── Handle Request                         │
│  │   ├── Middleware Pipeline [YOUR CODE]    │
│  │   ├── Route Resolution                   │
│  │   ├── Controller Method  [YOUR CODE]    │
│  │   └── Response                           │
│  └── Terminate                              │
└─────────────────────────────────────────────┘
```

### Template Method + Factory Method

এই দুটি প্যাটার্ন একসাথে অত্যন্ত শক্তিশালী। Template Method অ্যালগরিদমের কাঠামো define করে, Factory Method সেই অ্যালগরিদমে ব্যবহৃত object তৈরি করে:

```php
<?php

abstract class NotificationSender
{
    // Template Method
    final public function send(string $to, string $message): bool
    {
        $channel = $this->createChannel();       // Factory Method
        $formatted = $this->formatMessage($message);
        
        return $channel->deliver($to, $formatted);
    }

    // Factory Method — subclass decide করবে কোন channel তৈরি হবে
    abstract protected function createChannel(): NotificationChannel;

    // Template step
    abstract protected function formatMessage(string $message): string;
}

interface NotificationChannel
{
    public function deliver(string $to, string $message): bool;
}

class SmsNotificationSender extends NotificationSender
{
    protected function createChannel(): NotificationChannel
    {
        return new SmsChannel(); // Factory Method — SMS channel তৈরি
    }

    protected function formatMessage(string $message): string
    {
        return mb_substr($message, 0, 160); // SMS 160 char limit
    }
}

class EmailNotificationSender extends NotificationSender
{
    protected function createChannel(): NotificationChannel
    {
        return new EmailChannel();
    }

    protected function formatMessage(string $message): string
    {
        return "<html><body><p>{$message}</p></body></html>";
    }
}
```

---

## ✅ Pros ও ❌ Cons

### ✅ সুবিধাসমূহ

| সুবিধা | বর্ণনা |
|---------|--------|
| **Code Reuse** | Common logic base class-এ একবার লেখা হয়, সব subclass share করে |
| **DRY Principle** | Duplicate code এড়ানো যায় — কাঠামো একবারই লিখতে হয় |
| **Open/Closed Principle** | নতুন variation যোগ করতে base class পরিবর্তন করতে হয় না |
| **Controlled Extension** | `final` keyword দিয়ে অ্যালগরিদমের কাঠামো protect করা যায় |
| **Consistency** | সব subclass একই কাঠামো follow করতে বাধ্য |
| **Hook Methods** | Optional customization points দেওয়া যায় |

### ❌ অসুবিধাসমূহ

| অসুবিধা | বর্ণনা |
|----------|--------|
| **Tight Coupling** | Subclass base class-এর উপর strongly dependent |
| **Liskov Substitution Risk** | Subclass ভুলভাবে override করলে behavior ভেঙে যেতে পারে |
| **Debugging Difficulty** | Execution flow base ↔ subclass-এর মধ্যে jump করে, trace করা কঠিন |
| **Inheritance Limitation** | Single inheritance ভাষায় (Java/PHP) শুধু একটি template follow করতে পারে |
| **Rigidity** | অ্যালগরিদমের কাঠামো পরিবর্তন করা কঠিন — সব subclass affected হয় |
| **Fragile Base Class** | Base class পরিবর্তন করলে সব subclass break হতে পারে |

---

## ⚠️ Common Mistakes

### ১. Template Method-কে `final` না করা

```php
// ❌ ভুল — subclass পুরো algorithm override করে ফেলতে পারে
abstract class BadTemplate
{
    public function process(): void // final নেই!
    {
        $this->step1();
        $this->step2();
    }
}

// ✅ সঠিক — কাঠামো protect করা হয়েছে
abstract class GoodTemplate
{
    final public function process(): void // final!
    {
        $this->step1();
        $this->step2();
    }
}
```

### ২. অতিরিক্ত Abstract Methods

```php
// ❌ ভুল — ১০+ abstract methods = subclass-এর কষ্ট
abstract class TooManySteps
{
    abstract protected function step1(): void;
    abstract protected function step2(): void;
    abstract protected function step3(): void;
    abstract protected function step4(): void;
    abstract protected function step5(): void;
    abstract protected function step6(): void;
    abstract protected function step7(): void;
    // ... এটি একটি design smell!
}

// ✅ সঠিক — ৩-৫টি meaningful step
abstract class JustRight
{
    abstract protected function parse(string $input): array;
    abstract protected function validate(array $data): array;
    abstract protected function process(array $data): mixed;
}
```

### ৩. Hook Method-এ Side Effects

```php
// ❌ ভুল — Hook-এ heavy logic
abstract class BadHook
{
    protected function shouldProcess(): bool
    {
        $this->connectToDatabase();    // Side effect!
        $this->loadConfiguration();    // Side effect!
        return $this->checkPermission();
    }
}

// ✅ সঠিক — Hook শুধু decision নেয়
abstract class GoodHook
{
    protected function shouldProcess(): bool
    {
        return true; // Simple boolean decision
    }
}
```

### ৪. Base Class-এ Concrete State রাখা

```php
// ❌ ভুল — base class-এ mutable state
abstract class StatefulTemplate
{
    protected array $results = []; // Subclass এটি corrupt করতে পারে!
    protected int $count = 0;
}

// ✅ সঠিক — state encapsulate করুন
abstract class StatelessTemplate
{
    final public function process(): ProcessResult
    {
        $data = $this->fetch();
        $processed = $this->transform($data);
        return new ProcessResult($processed); // Immutable result
    }
}
```

### ৫. Strategy ব্যবহার করা উচিত যেখানে Template Method ব্যবহার করা

```php
// ❌ ভুল — পুরো algorithm vary করে, কাঠামো fix নয়
abstract class WrongUseCase
{
    final public function calculate(float $amount): float
    {
        return $this->doCalculation($amount); // শুধু একটি step!
    }
    abstract protected function doCalculation(float $amount): float;
}
// এটি Strategy pattern হওয়া উচিত, Template Method নয়

// ✅ সঠিক — Template Method ব্যবহার করুন যখন multiple steps আছে
abstract class RightUseCase
{
    final public function calculate(float $amount): float
    {
        $validated = $this->validate($amount);
        $adjusted = $this->applyDiscount($validated);
        $taxed = $this->applyTax($adjusted);
        return $this->round($taxed);
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

// Test-এর জন্য concrete implementation
class TestableImporter extends DataImporter
{
    public array $parsedData = [];
    public array $savedData = [];
    public bool $beforeImportCalled = false;
    public bool $afterImportCalled = false;

    protected function parse(string $source): array
    {
        return $this->parsedData;
    }

    protected function validate(array $records): array
    {
        return array_filter($records, fn($r) => $r->isValid);
    }

    protected function transform(array $records): array
    {
        return $records;
    }

    protected function save(array $records): int
    {
        $this->savedData = $records;
        return count($records);
    }

    protected function beforeImport(string $source): void
    {
        $this->beforeImportCalled = true;
    }

    protected function afterImport(int $count): void
    {
        $this->afterImportCalled = true;
    }
}

class DataImporterTest extends TestCase
{
    private TestableImporter $importer;

    protected function setUp(): void
    {
        $this->importer = new TestableImporter();
    }

    public function test_template_method_executes_all_steps_in_order(): void
    {
        $this->importer->parsedData = [
            new ImportRecord('1', ['name' => 'Test'], true),
            new ImportRecord('2', ['name' => 'Test2'], true),
        ];

        $result = $this->importer->import('test_source');

        $this->assertTrue($this->importer->beforeImportCalled);
        $this->assertTrue($this->importer->afterImportCalled);
        $this->assertCount(2, $this->importer->savedData);
        $this->assertSame(ImportStatus::Success, $result->status);
    }

    public function test_invalid_records_are_filtered_during_validation(): void
    {
        $this->importer->parsedData = [
            new ImportRecord('1', ['name' => 'Valid'], true),
            new ImportRecord('2', ['name' => 'Invalid'], false), // isValid = false
        ];

        $result = $this->importer->import('test_source');

        $this->assertSame(1, $result->importedRecords);
        $this->assertSame(1, $result->failedRecords);
        $this->assertSame(ImportStatus::PartialSuccess, $result->status);
    }

    public function test_empty_data_returns_failed_status(): void
    {
        $this->importer->parsedData = [];

        $result = $this->importer->import('test_source');

        $this->assertSame(ImportStatus::Failed, $result->status);
        $this->assertSame(0, $result->totalRecords);
    }

    public function test_hooks_are_called_in_correct_order(): void
    {
        $callOrder = [];

        $importer = new class extends DataImporter {
            public array $callOrder = [];

            protected function parse(string $source): array
            {
                $this->callOrder[] = 'parse';
                return [new ImportRecord('1', ['x' => 'y'], true)];
            }
            protected function validate(array $records): array
            {
                $this->callOrder[] = 'validate';
                return $records;
            }
            protected function transform(array $records): array
            {
                $this->callOrder[] = 'transform';
                return $records;
            }
            protected function save(array $records): int
            {
                $this->callOrder[] = 'save';
                return count($records);
            }
            protected function beforeImport(string $source): void
            {
                $this->callOrder[] = 'beforeImport';
            }
            protected function afterImport(int $count): void
            {
                $this->callOrder[] = 'afterImport';
            }
        };

        $importer->import('test');

        $this->assertSame(
            ['beforeImport', 'parse', 'validate', 'transform', 'save', 'afterImport'],
            $importer->callOrder,
        );
    }

    public function test_csv_importer_parses_correctly(): void
    {
        $csvImporter = new CsvImporter();
        $csv = "name,email\nরহিম,rahim@test.com\nকরিম,karim@test.com";

        $result = $csvImporter->import($csv);

        $this->assertSame(ImportStatus::Success, $result->status);
        $this->assertSame(2, $result->importedRecords);
    }
}
```

### Jest (JavaScript)

```javascript
import { describe, it, expect, jest, beforeEach } from '@jest/globals';

// টেস্টের জন্য concrete class
class TestableImporter extends DataImporter {
    parsedData = [];
    savedRecords = [];
    hooksCalled = [];

    async parse(source) {
        this.hooksCalled.push('parse');
        return this.parsedData;
    }

    validate(records) {
        this.hooksCalled.push('validate');
        return records.filter(r => r.isValid !== false);
    }

    transform(records) {
        this.hooksCalled.push('transform');
        return records;
    }

    async save(records) {
        this.hooksCalled.push('save');
        this.savedRecords = records;
        return records.length;
    }

    async beforeImport(source) {
        this.hooksCalled.push('beforeImport');
    }

    async afterImport(count) {
        this.hooksCalled.push('afterImport');
    }
}

describe('DataImporter Template Method', () => {
    let importer;

    beforeEach(() => {
        importer = new TestableImporter();
    });

    it('should execute all steps in correct order', async () => {
        importer.parsedData = [
            { id: '1', data: { name: 'Test' } },
        ];

        await importer.import('source');

        expect(importer.hooksCalled).toEqual([
            'beforeImport', 'parse', 'validate', 'transform', 'save', 'afterImport',
        ]);
    });

    it('should return success when all records import', async () => {
        importer.parsedData = [
            { id: '1', data: { name: 'A' } },
            { id: '2', data: { name: 'B' } },
        ];

        const result = await importer.import('source');

        expect(result.status).toBe('success');
        expect(result.importedRecords).toBe(2);
        expect(result.failedRecords).toBe(0);
    });

    it('should return partial_success when some records fail validation', async () => {
        importer.parsedData = [
            { id: '1', data: { name: 'Valid' } },
            { id: '2', data: { name: 'Invalid' }, isValid: false },
        ];

        const result = await importer.import('source');

        expect(result.status).toBe('partial_success');
        expect(result.importedRecords).toBe(1);
        expect(result.failedRecords).toBe(1);
    });

    it('should return failed when no records exist', async () => {
        importer.parsedData = [];

        const result = await importer.import('source');

        expect(result.status).toBe('failed');
        expect(result.totalRecords).toBe(0);
    });

    it('should not allow direct instantiation of abstract class', () => {
        expect(() => new DataImporter()).toThrow('Cannot instantiate abstract DataImporter');
    });
});

describe('CsvImporter', () => {
    it('should parse and import CSV data correctly', async () => {
        const importer = new CsvImporter();
        const csv = 'name,email,phone\nরহিম,rahim@test.com,017111\nকরিম,karim@test.com,017222';

        const result = await importer.import(csv);

        expect(result.status).toBe('success');
        expect(result.importedRecords).toBe(2);
    });
});

describe('Hook Methods', () => {
    it('should allow hook override without breaking template', async () => {
        class CustomImporter extends TestableImporter {
            customHookCalled = false;

            async beforeImport(source) {
                this.customHookCalled = true;
                await super.beforeImport(source);
            }
        }

        const custom = new CustomImporter();
        custom.parsedData = [{ id: '1', data: {} }];

        await custom.import('src');

        expect(custom.customHookCalled).toBe(true);
    });
});
```

---

## 🔗 সম্পর্কিত প্যাটার্ন

### Strategy Pattern

| বিষয় | Template Method | Strategy |
|-------|----------------|----------|
| মেকানিজম | Inheritance | Composition |
| Variation level | Step-level | Algorithm-level |
| Binding time | Compile-time | Runtime |
| Coupling | Tight (base-sub) | Loose (interface) |
| SOLID | Open/Closed ✅ | Open/Closed ✅, DIP ✅ |

**নিয়ম:** যদি শুধু ১-২টি step vary করে → Template Method। যদি পুরো algorithm vary করে → Strategy।

### Factory Method

Template Method এবং Factory Method প্রায়ই একসাথে ব্যবহার হয়। Template Method অ্যালগরিদমের **কাঠামো** define করে, Factory Method সেই কাঠামোর ভিতরে ব্যবহৃত **object** তৈরি করে। Factory Method আসলে Template Method-এর একটি **specialization** — যেখানে "step" হলো object creation।

### Observer Pattern

Template Method-এর hook method-গুলো Observer-এর মতো কাজ করে। `beforeImport()`, `afterImport()` — এগুলো essentially event notification। বড় সিস্টেমে hook method-এর বদলে Observer ব্যবহার করলে আরো flexible হয়:

```php
// Template Method + Observer hybrid
abstract class EnhancedImporter
{
    private array $listeners = [];

    public function on(string $event, callable $listener): void
    {
        $this->listeners[$event][] = $listener;
    }

    final public function import(string $source): ImportResult
    {
        $this->emit('before_import', $source);
        // ... steps ...
        $this->emit('after_import', $result);
        return $result;
    }

    private function emit(string $event, mixed ...$args): void
    {
        foreach ($this->listeners[$event] ?? [] as $listener) {
            $listener(...$args);
        }
    }
}
```

---

## 📏 কখন ব্যবহার করবেন / করবেন না

### ✅ ব্যবহার করবেন যখন:

1. **একই অ্যালগরিদমের কাঠামো, ভিন্ন ভিন্ন step implementation** — যেমন বিভিন্ন ফরম্যাটে ডেটা import
2. **Framework/Library তৈরি করছেন** — ব্যবহারকারীকে নির্দিষ্ট extension point দিতে চান
3. **Duplicate code eliminate করতে চান** — একই কাঠামো বারবার লেখা হচ্ছে, শুধু details আলাদা
4. **Step execution order গুরুত্বপূর্ণ** — steps অবশ্যই নির্দিষ্ট ক্রমে চলতে হবে
5. **Optional behavior দিতে চান** — Hook methods দিয়ে optional customization
6. **Bangladesh payment gateway integration** — bKash/Nagad/Rocket এর flow একই কিন্তু details আলাদা
7. **Report generation** — একই কাঠামোতে বিভিন্ন ধরনের report

### ❌ ব্যবহার করবেন না যখন:

1. **শুধু একটি step vary করে** — Strategy pattern ভালো হবে
2. **Runtime-এ algorithm বদলাতে হবে** — Strategy ব্যবহার করুন
3. **Steps-এর মধ্যে কোনো dependency নেই** — তাহলে কাঠামোর প্রয়োজন নেই
4. **অতিরিক্ত inheritance hierarchy তৈরি হচ্ছে** — Composition prefer করুন
5. **Base class ক্রমাগত পরিবর্তন হচ্ছে** — Fragile base class problem হবে
6. **Algorithm-এর step সংখ্যা ২-৩টির কম** — Overkill হবে

### সিদ্ধান্ত নেওয়ার ফ্লোচার্ট

```
অ্যালগরিদমের কাঠামো কি fixed?
├── হ্যাঁ → Steps কি vary করে?
│          ├── হ্যাঁ → কতগুলো step vary করে?
│          │          ├── ১-২টি → Template Method ✅
│          │          └── সবগুলো → Strategy ✅
│          └── না → সাধারণ class-ই যথেষ্ট
└── না → Strategy অথবা Command pattern ✅
```

---

## 📋 সারসংক্ষেপ

### মূল বিষয়গুলো এক নজরে

```
┌─────────────────────────────────────────────────┐
│           Template Method Pattern                │
├─────────────────────────────────────────────────┤
│ 🎯 উদ্দেশ্য: অ্যালগরিদমের কাঠামো fix রেখে     │
│    নির্দিষ্ট step-গুলো subclass-এ vary করা      │
│                                                  │
│ 🔑 মূলনীতি:                                     │
│    • Hollywood Principle (IoC)                   │
│    • Open/Closed Principle                       │
│    • DRY (Don't Repeat Yourself)                │
│                                                  │
│ 🧩 তিন ধরনের method:                            │
│    1. Template Method (final) — কাঠামো          │
│    2. Abstract Method — must implement           │
│    3. Hook Method — optional override            │
│                                                  │
│ ⚡ মনে রাখবেন:                                   │
│    • templateMethod() অবশ্যই final হবে          │
│    • ৩-৫টি step ideal                           │
│    • Hook = optional, Abstract = mandatory       │
│    • Strategy prefer করুন যদি runtime vary করে  │
│    • Framework তৈরিতে অপরিহার্য                 │
└─────────────────────────────────────────────────┘
```

### বাংলাদেশ প্রেক্ষাপটে ব্যবহার

- **পেমেন্ট প্রসেসিং:** bKash, Nagad, Rocket — একই flow, ভিন্ন implementation
- **সরকারি ফর্ম:** NID verification, Tax filing — steps fixed, details vary
- **ই-কমার্স:** Order processing pipeline — validate → payment → shipping → notification
- **ব্যাংকিং:** Loan processing, Account opening — নির্দিষ্ট ধাপ follow করতে হয়
- **রিপোর্ট জেনারেশন:** বিক্রয়, আর্থিক, ইনভেন্টরি — একই কাঠামো, ভিন্ন ডেটা

### Quick Reference

```php
// PHP — Template Method মনে রাখার সূত্র
abstract class Template
{
    final public function algorithm(): void  // ① final template
    {
        $this->requiredStep();               // ② abstract = must
        if ($this->optionalHook()) {         // ③ hook = optional
            $this->anotherStep();
        }
        $this->fixedStep();                  // ④ concrete = fixed
    }

    abstract protected function requiredStep(): void;
    abstract protected function anotherStep(): void;
    protected function optionalHook(): bool { return true; }
    private function fixedStep(): void { /* ... */ }
}
```

```javascript
// JavaScript — Template Method মনে রাখার সূত্র
class Template {
    algorithm() {                            // ① template method
        this.requiredStep();                 // ② abstract = must
        if (this.optionalHook()) {           // ③ hook = optional
            this.anotherStep();
        }
        this.#fixedStep();                   // ④ private = fixed
    }

    requiredStep() { throw new Error('Abstract'); }
    anotherStep() { throw new Error('Abstract'); }
    optionalHook() { return true; }
    #fixedStep() { /* ... */ }
}
```

> **"Template Method হলো framework-এর হৃদপিণ্ড। আপনি যখন কোনো framework ব্যবহার করেন, আপনি আসলে Template Method pattern-এর ভিতরেই থাকেন।"**

---

*এই ডকুমেন্ট তৈরি করা হয়েছে বাংলাদেশের সফটওয়্যার ডেভেলপারদের জন্য, বাংলা ভাষায়, বাস্তব প্রেক্ষাপটে।*
