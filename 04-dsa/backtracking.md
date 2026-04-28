# 🔙 ব্যাকট্র্যাকিং (Backtracking)

> _"যখন একটা পথ বন্ধ, পেছনে ফিরে এসে আরেকটা পথ চেষ্টা করো — হাল ছেড়ো না।"_ 🚪

---

## 📖 সংজ্ঞা

**ব্যাকট্র্যাকিং** হলো একটি **অ্যালগরিদমিক কৌশল** যেখানে সমস্যার সমাধান ধাপে ধাপে তৈরি করা হয়। কোনো ধাপে যদি বোঝা যায় যে বর্তমান পথে সমাধান সম্ভব নয়, তখন **পেছনে ফিরে** (backtrack) আগের ধাপে যাওয়া হয় এবং **অন্য পথ** চেষ্টা করা হয়।

এটি মূলত **Brute Force**-এর একটি **স্মার্ট সংস্করণ** — যেসব পথে সমাধান নেই, সেগুলো আগেই বাদ দেওয়া হয় (pruning)।

## 🏠 বাস্তব উদাহরণ

| বাস্তব জগৎ | ব্যাকট্র্যাকিং-এর সাথে তুলনা |
|---|---|
| গোলকধাঁধা (Maze) | ভুল পথে গেলে ফিরে এসে আরেকটা পথ ধরো |
| সুডোকু সমাধান | ভুল সংখ্যা বসালে মুছে নতুন সংখ্যা চেষ্টা |
| পাসওয়ার্ড ক্র্যাক | সব কম্বিনেশন চেষ্টা, ভুল হলে পরবর্তীটা |
| পোশাক বাছাই | শার্ট-প্যান্ট ম্যাচ না হলে অন্য শার্ট চেষ্টা |
| রুট পরিকল্পনা | একটা রাস্তা বন্ধ থাকলে বিকল্প রাস্তা |

---

## ১. ব্যাকট্র্যাকিং-এর মৌলিক ধারণা

### 🔑 মূল প্যাটার্ন

```
function backtrack(candidate):
    if candidate হলো সমাধান:
        সমাধান সংরক্ষণ করো
        return

    for প্রতিটি সম্ভাব্য পরবর্তী ধাপ:
        if ধাপটি valid:
            ধাপ নাও (choose)
            backtrack(updated candidate)
            ধাপ বাতিল করো (unchoose) ← এটাই backtrack!
```

### 🆚 Backtracking vs Brute Force vs DP

| বৈশিষ্ট্য | Brute Force | Backtracking | DP |
|---|---|---|---|
| কৌশল | সব চেষ্টা | ভুল পথ আগেই বাদ | সাব-প্রবলেম cache |
| কার্যকারিতা | সবচেয়ে ধীর | মাঝামাঝি | সবচেয়ে দ্রুত |
| মেমোরি | কম | কম | বেশি |
| কখন ব্যবহার | সবসময় চলে | Constraint satisfaction | Overlapping subproblems |

---

## ২. N-Queens Problem — N রানী সমস্যা

**সমস্যা**: N×N দাবার বোর্ডে N টি রানী এমনভাবে বসাও যেন কোনো রানী অন্য কোনো রানীকে আক্রমণ করতে না পারে।

```
4-Queens এর একটি সমাধান:

  . Q . .       কলাম ১-এ রানী → সারি ০
  . . . Q       কলাম ৩-এ রানী → সারি ১
  Q . . .       কলাম ০-এ রানী → সারি ২
  . . Q .       কলাম ২-এ রানী → সারি ৩

কোনো রানী একই সারি, কলাম বা কর্ণে নেই ✓
```

### PHP ইমপ্লিমেন্টেশন

```php
<?php
function solveNQueens(int $n): array {
    $solutions = [];
    $board = array_fill(0, $n, array_fill(0, $n, '.'));
    placeQueen($board, 0, $n, $solutions);
    return $solutions;
}

function placeQueen(array &$board, int $row, int $n, array &$solutions): void {
    if ($row === $n) {
        $solutions[] = array_map(fn($r) => implode('', $r), $board);
        return;
    }

    for ($col = 0; $col < $n; $col++) {
        if (isSafe($board, $row, $col, $n)) {
            $board[$row][$col] = 'Q';       // choose
            placeQueen($board, $row + 1, $n, $solutions);
            $board[$row][$col] = '.';       // unchoose (backtrack!)
        }
    }
}

function isSafe(array $board, int $row, int $col, int $n): bool {
    // একই কলামে চেক
    for ($i = 0; $i < $row; $i++) {
        if ($board[$i][$col] === 'Q') return false;
    }

    // ওপরের বাম কর্ণ চেক
    for ($i = $row - 1, $j = $col - 1; $i >= 0 && $j >= 0; $i--, $j--) {
        if ($board[$i][$j] === 'Q') return false;
    }

    // ওপরের ডান কর্ণ চেক
    for ($i = $row - 1, $j = $col + 1; $i >= 0 && $j < $n; $i--, $j++) {
        if ($board[$i][$j] === 'Q') return false;
    }

    return true;
}

$solutions = solveNQueens(4);
echo count($solutions) . " টি সমাধান পাওয়া গেছে\n"; // 2
foreach ($solutions as $sol) {
    foreach ($sol as $row) echo $row . "\n";
    echo "---\n";
}
```

### JavaScript ইমপ্লিমেন্টেশন

```javascript
function solveNQueens(n) {
    const solutions = [];
    const board = Array.from({ length: n }, () => Array(n).fill("."));
    placeQueen(board, 0, n, solutions);
    return solutions;
}

function placeQueen(board, row, n, solutions) {
    if (row === n) {
        solutions.push(board.map(r => r.join("")));
        return;
    }

    for (let col = 0; col < n; col++) {
        if (isSafe(board, row, col, n)) {
            board[row][col] = "Q";
            placeQueen(board, row + 1, n, solutions);
            board[row][col] = "."; // backtrack!
        }
    }
}

function isSafe(board, row, col, n) {
    for (let i = 0; i < row; i++) {
        if (board[i][col] === "Q") return false;
    }
    for (let i = row - 1, j = col - 1; i >= 0 && j >= 0; i--, j--) {
        if (board[i][j] === "Q") return false;
    }
    for (let i = row - 1, j = col + 1; i >= 0 && j < n; i--, j++) {
        if (board[i][j] === "Q") return false;
    }
    return true;
}

const solutions = solveNQueens(4);
console.log(`${solutions.length} টি সমাধান`); // 2
solutions.forEach(sol => { sol.forEach(r => console.log(r)); console.log("---"); });
```

---

## ৩. Sudoku Solver — সুডোকু সমাধানকারী

```php
<?php
function solveSudoku(array &$board): bool {
    for ($row = 0; $row < 9; $row++) {
        for ($col = 0; $col < 9; $col++) {
            if ($board[$row][$col] === 0) {
                for ($num = 1; $num <= 9; $num++) {
                    if (isValidPlacement($board, $row, $col, $num)) {
                        $board[$row][$col] = $num;      // choose

                        if (solveSudoku($board)) return true;

                        $board[$row][$col] = 0;          // backtrack!
                    }
                }
                return false; // কোনো সংখ্যা কাজ করেনি
            }
        }
    }
    return true; // সব ঘর পূরণ হয়ে গেছে
}

function isValidPlacement(array $board, int $row, int $col, int $num): bool {
    // একই সারিতে চেক
    if (in_array($num, $board[$row])) return false;

    // একই কলামে চেক
    for ($i = 0; $i < 9; $i++) {
        if ($board[$i][$col] === $num) return false;
    }

    // 3×3 বক্সে চেক
    $boxRow = intdiv($row, 3) * 3;
    $boxCol = intdiv($col, 3) * 3;
    for ($i = $boxRow; $i < $boxRow + 3; $i++) {
        for ($j = $boxCol; $j < $boxCol + 3; $j++) {
            if ($board[$i][$j] === $num) return false;
        }
    }

    return true;
}
```

```javascript
function solveSudoku(board) {
    for (let row = 0; row < 9; row++) {
        for (let col = 0; col < 9; col++) {
            if (board[row][col] === 0) {
                for (let num = 1; num <= 9; num++) {
                    if (isValid(board, row, col, num)) {
                        board[row][col] = num;
                        if (solveSudoku(board)) return true;
                        board[row][col] = 0; // backtrack!
                    }
                }
                return false;
            }
        }
    }
    return true;
}

function isValid(board, row, col, num) {
    for (let i = 0; i < 9; i++) {
        if (board[row][i] === num) return false;
        if (board[i][col] === num) return false;
    }
    const boxRow = Math.floor(row / 3) * 3;
    const boxCol = Math.floor(col / 3) * 3;
    for (let i = boxRow; i < boxRow + 3; i++) {
        for (let j = boxCol; j < boxCol + 3; j++) {
            if (board[i][j] === num) return false;
        }
    }
    return true;
}
```

---

## ৪. Subset Sum — সাবসেট যোগফল

**সমস্যা**: একটি অ্যারে থেকে এমন সাবসেট খোঁজো যার যোগফল একটি নির্দিষ্ট মানের সমান।

```php
<?php
function subsetSum(array $nums, int $target): array {
    $results = [];
    findSubsets($nums, $target, 0, [], $results);
    return $results;
}

function findSubsets(array $nums, int $remaining, int $start, array $current, array &$results): void {
    if ($remaining === 0) {
        $results[] = $current;
        return;
    }
    if ($remaining < 0) return; // pruning!

    for ($i = $start; $i < count($nums); $i++) {
        $current[] = $nums[$i];                                // choose
        findSubsets($nums, $remaining - $nums[$i], $i + 1, $current, $results);
        array_pop($current);                                   // backtrack!
    }
}

$result = subsetSum([2, 3, 6, 7], 7);
print_r($result); // [[7], [2, 3, ...]] ইত্যাদি
```

```javascript
function subsetSum(nums, target) {
    const results = [];
    findSubsets(nums, target, 0, [], results);
    return results;
}

function findSubsets(nums, remaining, start, current, results) {
    if (remaining === 0) {
        results.push([...current]);
        return;
    }
    if (remaining < 0) return;

    for (let i = start; i < nums.length; i++) {
        current.push(nums[i]);
        findSubsets(nums, remaining - nums[i], i + 1, current, results);
        current.pop(); // backtrack!
    }
}

console.log(subsetSum([2, 3, 6, 7], 7));
// [[2, 3, ...], [7]]
```

---

## ✅ কখন ব্যবহার করবেন

| পরিস্থিতি | কেন ব্যাকট্র্যাকিং |
|---|---|
| N-Queens, Sudoku | Constraint satisfaction problem |
| সব কম্বিনেশন/পারমুটেশন বের করা | সম্পূর্ণ অনুসন্ধান দরকার |
| গ্রাফে পথ খোঁজা | সব পথ বের করতে হলে |
| গোলকধাঁধা সমাধান | পথ খুঁজে বের করা |
| পাজল সমাধান | Crossword, Word Search |

## ❌ কখন ব্যবহার করবেন না

| পরিস্থিতি | কেন না |
|---|---|
| Overlapping subproblems আছে | DP ব্যবহার করো |
| গ্রিডি কাজ করে | গ্রিডি দ্রুততর |
| খুব বড় সার্চ স্পেস | টাইম আউট হতে পারে |
| অপটিমাল সমাধান একটাই দরকার | BFS/DFS যথেষ্ট হতে পারে |

---

## 🧠 মনে রাখার টিপস

```
ব্যাকট্র্যাকিং = রিকার্শন + choose/unchoose + pruning
তিনটি ধাপ: ১. বেছে নাও, ২. এগিয়ে যাও, ৩. বাতিল করো
Pruning = ভুল পথ আগেই চিহ্নিত করে বাদ দেওয়া
ফর্মুলা: if (!valid) return; ← এটাই pruning
```
