## 0. 目錄
1. [[#1. 求職策略與履歷]]
2. [[#2. 面試流程與溝通技巧]]
3. [[#3. C 語言（最重要）]]
4. [[#4. Coding 題型]]
5. [[#5. Data Structure]]
6. [[#6. OS / RTOS]]
7. [[#7. Multithread]]
8. [[#8. Computer Architecture]]
9. [[#9. Peripheral / 通訊協定 / 硬體]]
10. [[#10. 應用題 / Design 題]]
11. [[#11. Behavioral]]
12. [[#12. 資源]]
13. [[#13. 準備 Checklist]]

---

## 1. 求職策略與履歷

### 投遞策略
- **越早投越好**：別等履歷改到完美。別人可能已經到第 2 次 phone screen 或 panel。
- 海投是有效的（原作者 600+ 投遞、23 個 NG 面試，offer 全來自海投）；內推能拿就拿（公司有 referral bonus，是最強入口之一），但不要只等內推。
- LinkedIn 保持更新（Premium 可增加 recruiter 曝光）。
- Cisco NG 投了幾乎都會給 OA；Amazon 也常有 OA。

### 履歷寫法
- **一頁**，給多人（不同背景也可）看過 —— 自己看不出 bug。
- **開頭 2 句 Summary**：明確寫「Embedded Software Engineer with …（技能）」，讓 HR 分辨你不是一般 SDE。
- 每段經歷 **1–3 個 bullet**，每個 bullet **盡量寫滿一行**，挑最精華的。
- 用 **STAR**（Situation / Task / Action / Result）寫故事，**成果盡量量化**。
- 第一關通常是 HR（過 ATS 後）→ HR 看的是：「**大概**用了什麼技術、解決什麼問題、達成什麼（數字）」，不是技術細節。細節留到面試再聊。
- 檢驗標準：別人**一眼**看懂你的 project 在做什麼、強項是什麼 → 有記憶點。看完還不清楚 → 要改。
- 履歷上出現的東西（I2C / SPI / UART / RTOS…）**都要能被深挖**。

---

## 2. 面試流程與溝通技巧

### 面試官在看什麼
四個維度：**Problem solving / Coding / Verification / Communication**。
沉默很久後突然說不會，或直接丟出答案不解釋 → 面試官無法評估，還可能懷疑你背答案。

### Coding 面試 7 步驟
| # | 步驟 | 要做什麼 |
|---|---|---|
| 1 | **釐清題目** | 用自己的話重述題目；問 input/output、範圍、edge cases；自己舉例子確認 |
| 2 | **討論可能解法** | 一步步說出思路，不要直接跳結論；可同時提多個方向 |
| 3 | **比較解法** | 說出每個解法的 time/space trade-off，提出推薦方案，**取得面試官同意再寫** |
| 4 | **寫 code** | 邊寫邊講；變數命名有意義；先搭骨架再補 edge case |
| 5 | **自我測試** | 自己拿例子 dry run、找 bug 修掉，再宣布完成 |
| 6 | **複雜度分析** | 主動講 time / space complexity |
| 7 | **Follow-up** | 有時間就延伸（Apple 等公司 follow-up 答對才算 positive） |

### 實用句型
- 釐清：*"Let me rephrase the problem to make sure we are on the same page."*
- 確認假設：*"I'm guessing that means we can…"* / *"Sounds like we don't need to care about …?"*
- 比較：*"This one is better in time complexity, but that one is simpler."*
- 卡住時：*"Let me take a step back and think about another approach."*
- 沒把握最佳解：先提 *"Let me implement a simpler solution first, then optimize."*
- Debug 時說出來：*"This is weird, I'm expecting X but seeing Y."*
- Check in：*"It's taking longer than expected — would you like me to keep going?"* / *"Should we start wrapping up?"*
- **卡住就主動要 hint**，不要沉默掙扎。

### 其他
- 就算看過題目，也不要反射式直接寫，照流程釐清。
- **不要背 LeetCode 解答**：面試官問「為什麼這樣寫」會答不出來（例如何時需要 dummy node）。
- 講 code 時用正確術語：`*` 叫 **dereference operator**（宣告時是 pointer declarator），不要說 star。
- 練習白板 / CoderPad 手寫，並做 **mock interview**。

---

## 3. C 語言（最重要）

> [!important] 要熟到不行。幾乎每家都會問，Linked List 題也同時在考 pointer / malloc / struct。

### 3.1 Data type 大小
- `char` 1、`short` 2、`int` 通常 4、`long`：LP64（Linux 64-bit）8 / LLP64（Windows）4、`long long` 8、pointer 在 32-bit 為 4 / 64-bit 為 8。
- 實務用 `<stdint.h>`：`uint8_t`、`uint32_t`…。
- `signed` / `unsigned` 混用比較陷阱：`-1 < 0u` 為 **false**（-1 被轉成 unsigned）。
- `char` 是否 signed 是 implementation-defined。

### 3.2 關鍵字
| 關鍵字 | 重點 |
|---|---|
| `static` 區域變數 | 存在 `.data/.bss`，生命週期整個程式，只初始化一次 |
| `static` 全域變數 / 函式 | internal linkage，只在該檔案（translation unit）可見 |
| `static`（C++ 額外） | class 的 static member：屬於 class 而非 object；static member function 沒有 `this`（**Apple 問過 C vs C++ static 差別**） |
| `extern` | 宣告變數/函式定義在別的檔案；不配置記憶體 |
| `volatile` | 告訴 compiler 值可能在程式流程外被改變，**每次都從記憶體讀**、不可優化掉。用於：memory-mapped register、ISR 與主程式共用變數、multithread 共享旗標。**volatile ≠ atomic、不保證 thread safe** |
| `const` | 唯讀。`const int *p`（內容不可改）vs `int * const p`（指標不可改）vs `const int * const p` |
| `const volatile` | 例如唯讀的 status register |
| `union` | 所有成員共用同一塊記憶體，size = 最大成員（含對齊）。可用來做型別轉換 / endian 檢測 |
| `enum` | 整數常數集合，型別通常為 int |
| `inline` / `register` | 建議 compiler，不保證 |

### 3.3 Struct & Memory Alignment（常一起問）
- 每個成員對齊到自己的 size（或 alignment 要求），整個 struct 對齊到最大成員的倍數。
```c
struct A { char c; int i; char d; };   // 1+3(pad)+4+1+3(pad) = 12
struct B { int i; char c; char d; };   // 4+1+1+2(pad) = 8  → 成員順序影響 size
```
- `#pragma pack(1)` / `__attribute__((packed))` 可取消 padding，但 unaligned access 在部分架構（如某些 ARM）會變慢或 fault。
- 為什麼要對齊：CPU 一次讀一個 word，未對齊要讀兩次或觸發 exception。
- Bit field：`struct { unsigned a:3; unsigned b:5; }`，排列順序 implementation-defined。

### 3.4 Pointer
- **Pointer arithmetic**：`p + 1` 移動 `sizeof(*p)` bytes。
- **Wild pointer**：未初始化的指標。
- **Dangling pointer**：指向已 `free` 的記憶體或已離開 scope 的區域變數（例如 return 區域變數位址）。→ free 後設 `NULL`。
- **Double pointer**：在函式裡修改呼叫端的指標（如 linked list 的 `insert(Node **head)`）、2D 動態陣列。
- **Function pointer / Callback**：`int (*fp)(int, int);`、`void register_cb(void (*cb)(void));`、ISR vector table 就是 function pointer 陣列。
- Casting：`(uint32_t *)0x40000000` 存取暫存器；型別轉換可能違反 strict aliasing。
- 經典題：
```c
*(volatile uint32_t *)0x40021000 |= (1U << 5);   // set bit 5 of register
int *p[10];     // 10 個 int* 的陣列
int (*p)[10];   // 指向 10 個 int 陣列的指標
int (*fp)(int); // function pointer
```
- Array vs pointer：`sizeof(arr)` 是整個陣列大小；陣列傳入函式後退化為 pointer，`sizeof` 變成指標大小。

### 3.5 `sizeof` 易錯點
- `sizeof` 是 operator，編譯期求值（VLA 除外）；`sizeof(i++)` 不會執行 `i++`。
- `sizeof("abc")` = 4（含 `\0`）、`strlen("abc")` = 3。
- `char *s = "abc"; sizeof(s)` = 指標大小。

### 3.6 Call by value / reference
- C **只有 call by value**；「傳址」是把指標的值傳進去。
- C++ 才有真正的 reference（`int &x`）。

### 3.7 Macro
```c
#define SQUARE(x) ((x) * (x))          // 全部加括號
#define MIN(a,b) ((a) < (b) ? (a) : (b))  // 注意 MIN(i++, j) 會 side effect 兩次
#define SET_BIT(x,n)   ((x) |=  (1U << (n)))
#define CLR_BIT(x,n)   ((x) &= ~(1U << (n)))
#define TOG_BIT(x,n)   ((x) ^=  (1U << (n)))
#define CHK_BIT(x,n)   (((x) >> (n)) & 1U)
#define ARRAY_SIZE(a)  (sizeof(a) / sizeof((a)[0]))
```
- Macro vs inline function：macro 無型別檢查、可能多次求值；inline 有型別檢查。
- 多行 macro 用 `do { ... } while (0)` 包起來。

### 3.8 標準函式（要能手寫）
```c
size_t my_strlen(const char *s) { const char *p = s; while (*p) p++; return p - s; }

char *my_strcpy(char *dst, const char *src) { char *r = dst; while ((*dst++ = *src++)); return r; }

int my_strcmp(const char *a, const char *b) {
    while (*a && *a == *b) { a++; b++; }
    return (unsigned char)*a - (unsigned char)*b;
}
```
- **memcpy + 處理重疊（overlap）**：被考過 → 其實就是 `memmove`
```c
void *my_memmove(void *dst, const void *src, size_t n) {
    unsigned char *d = dst; const unsigned char *s = src;
    if (d == s || n == 0) return dst;
    if (d < s || d >= s + n) {          // 無重疊或 dst 在前 → 正向複製
        while (n--) *d++ = *s++;
    } else {                             // dst 在 src 區間內 → 反向複製
        d += n; s += n;
        while (n--) *--d = *--s;
    }
    return dst;
}
```
- memcpy **優化** follow-up：先逐 byte 複製到對齊邊界，再一次搬 word（4/8 bytes），最後補尾巴；或 loop unrolling、DMA。

### 3.9 malloc / free
- malloc 不初始化、calloc 初始化為 0、realloc 可能搬位置（要用暫存指標接，避免失敗時 leak）。
- Memory leak、double free、use-after-free。
- 嵌入式常避免動態配置（碎片化、不確定時間）→ 用 static pool。
- 手寫題：**aligned malloc / aligned free**
```c
void *aligned_malloc(size_t size, size_t align) {   // align 為 2 的冪
    void *raw = malloc(size + align - 1 + sizeof(void *));
    if (!raw) return NULL;
    uintptr_t addr = ((uintptr_t)raw + sizeof(void *) + align - 1) & ~(uintptr_t)(align - 1);
    ((void **)addr)[-1] = raw;       // 把原始指標藏在前面
    return (void *)addr;
}
void aligned_free(void *p) { if (p) free(((void **)p)[-1]); }
```

---

## 4. Coding 題型

> [!tip] 所有題目都要考慮 edge cases：NULL、空、單一元素、overflow、負數、重疊記憶體。

### 4.1 Linked List（常考到不行）
Screening 沒考，panel 某一關 8 成會考，甚至不只一次。可同時考 pointer、malloc、struct，面試官能依 level 追問。

**基本**：push、add（頭/尾/指定位置）、pop、delete（by value / by position）、print、free 整條
**應用（被考過）**：
- [ ] Reverse（考過 n 次；有人被要求**只能用 recursive**）
- [ ] Reverse doubly linked list（口述）
- [ ] Reverse a sublist（LC 92）
- [ ] Merge two sorted lists（LC 21）
- [ ] Merge k sorted lists（LC 23，O(nk) → heap / divide & conquer O(n log k)）
- [ ] Swap nodes in pairs（LC 24）
- [ ] Rotate list（LC 61）
- [ ] Detect cycle（LC 141/142，Floyd 快慢指標）
- [ ] Palindrome linked list（LC 234）
- [ ] Odd Even linked list（LC 328）
- [ ] Middle of the linked list（LC 876）
- [ ] Remove Nth node from end（LC 19）
- [ ] Sort list（merge sort，LC 148）

**Dummy node vs 只用 pointer**：當 head 可能被改（刪除 head、merge、插入到最前面）時用 dummy node 統一處理；也可用 `Node **pp` 間接指標技巧（Linus 的 "good taste"）。

```c
typedef struct Node { int val; struct Node *next; } Node;

Node *reverse(Node *head) {                // iterative
    Node *prev = NULL;
    while (head) { Node *nx = head->next; head->next = prev; prev = head; head = nx; }
    return prev;
}
Node *reverse_rec(Node *head) {            // recursive
    if (!head || !head->next) return head;
    Node *nh = reverse_rec(head->next);
    head->next->next = head; head->next = NULL;
    return nh;
}
void delete_val(Node **pp, int v) {        // pointer-to-pointer，不需特判 head
    while (*pp) {
        if ((*pp)->val == v) { Node *t = *pp; *pp = t->next; free(t); }
        else pp = &(*pp)->next;
    }
}
```

### 4.2 Bit Manipulation（底層必考）
**基本**：set / clear / toggle / check / mask / shift、AND / OR / XOR。

| 題目 | 解法重點 |
|---|---|
| Count set bits | ① loop 逐位 `n & 1` ② Brian Kernighan `n &= n - 1`（有人被要求給兩種解法）③ lookup table ④ `__builtin_popcount` |
| Reverse bits（LC 190） | 逐位搬，或分治 swap（16/8/4/2/1） |
| Swap 兩個 bit（位置 i, j） | 若不同才 `x ^= (1<<i) | (1<<j)` |
| 偶數位與奇數位交換 | `((x & 0xAAAAAAAA) >> 1) | ((x & 0x55555555) << 1)` |
| Print 32-bit binary | `for (i=31;i>=0;i--) putchar((x>>i)&1 ? '1':'0');` |
| 2's complement | `~x + 1` |
| 不用 + - * / 做加法（LC 371） | `while (b) { carry = (a & b) << 1; a ^= b; b = carry; }`（用 unsigned 避免 UB） |
| 判斷 2 的冪 | `x && !(x & (x - 1))` |
| 取最低位的 1 | `x & -x` |
| Endian swap | 見 [[#8. Computer Architecture]] |
| Gray code | `g = n ^ (n >> 1)` |
| 從 bit a 到 b 取值 / 寫入 field | `(x >> a) & ((1U << (b-a+1)) - 1)`；寫入先清 mask 再 OR |

```c
uint32_t reverse_bits(uint32_t n) {
    n = (n >> 16) | (n << 16);
    n = ((n & 0xFF00FF00) >> 8) | ((n & 0x00FF00FF) << 8);
    n = ((n & 0xF0F0F0F0) >> 4) | ((n & 0x0F0F0F0F) << 4);
    n = ((n & 0xCCCCCCCC) >> 2) | ((n & 0x33333333) << 2);
    n = ((n & 0xAAAAAAAA) >> 1) | ((n & 0x55555555) << 1);
    return n;
}
```
- 補充：float 的 IEEE 754 表示（sign 1 / exponent 8 / mantissa 23）、整數範圍（`INT_MIN` 取負會 overflow）、signed 右移是 implementation-defined（通常 arithmetic shift）、左移 signed 負數是 UB。
- 練習：**GeeksforGeeks bit manipulation 的 easy + medium** 看過一輪即可，hard 太 tricky。

### 4.3 String
核心技巧：two pointer、hash map（C 可用 `int count[256]`）、sliding window、大小寫轉換（`c ^ 0x20` 或 `tolower`）、熟練 `strlen/strcpy/strcmp`。

被考過：
- [ ] 輸出重複字元出現次數：`"abbbccddd"` → `b:3 c:2 d:3`
- [ ] 從 string1 移除所有出現在 string2 的字元（in-place）：`"this is a pencil"`, `"asc"` → `"thi i penil"`
- [ ] `atoi`（空白、正負號、非數字、**overflow**）、`itoa`（負數、`INT_MIN`、base）
- [ ] 字串反轉、反轉單字順序、搜尋子字串（`strstr`）、比較、複製
- [ ] UTF-8 編碼判斷 / 解析

```c
void remove_chars(char *s1, const char *s2) {
    bool del[256] = {0};
    for (; *s2; s2++) del[(unsigned char)*s2] = true;
    char *w = s1;
    for (char *r = s1; *r; r++) if (!del[(unsigned char)*r]) *w++ = *r;
    *w = '\0';
}
```

### 4.4 Array / Matrix
- [ ] 寫一個 function 動態 create 2D matrix 並回傳（`int **` 或一塊連續記憶體），記得 free
- [ ] Matmul（矩陣乘法）、矩陣內積
- [ ] 2D → 1D flatten：row-major `idx = r * cols + c`；column-major `idx = c * rows + r`（兩種都要會）
- [ ] Binary search、two pointer、sliding window、Kadane（最大子陣列和）
- [ ] 找出相加（或相乘）最大 / 等於特定值的組合（Two Sum 系列）
- [ ] Valid Sudoku / matrix 驗證
- [ ] QuickSort、MergeSort、Binary Search —— **最好 Array 和 Linked List 版本都會**

### 4.5 其他
- Tree：幾乎沒被問，但有人被問 BST，學基本 insert / search / traversal 即可。
- Graph：幾乎不考（例外：被轉去面 SDE 時考 topological sort，C 沒有 dict/STL 很吃虧）。

---

## 5. Data Structure

| 結構 | 適用場景 |
|---|---|
| Stack（LIFO） | 括號匹配（LC 20）、function call、undo、DFS |
| Queue（FIFO） | 任務排程、BFS、訊息傳遞 |
| Circular Buffer / Ring Buffer | UART RX/TX、audio stream、log、ISR ↔ main 傳資料、producer-consumer；固定大小不需 malloc |

被考過：
- [ ] 用 **linked list** 實作 Stack / Queue / Circular buffer
- [ ] Stack 搭配 LC 20（Valid Parentheses）一起考
- [ ] 用兩個 stack 實作 queue（LC 232）
- [ ] Circular queue follow-up：**producer / consumer 在不同 thread** → 需要 mutex / semaphore

```c
#define BUF_SIZE 16   // 2 的冪可用 & (BUF_SIZE-1) 取代 %
typedef struct { uint8_t data[BUF_SIZE]; volatile uint32_t head, tail; } ring_t;

bool rb_push(ring_t *r, uint8_t v) {
    uint32_t next = (r->head + 1) % BUF_SIZE;
    if (next == r->tail) return false;      // full（犧牲一格區分 full/empty）
    r->data[r->head] = v; r->head = next; return true;
}
bool rb_pop(ring_t *r, uint8_t *v) {
    if (r->head == r->tail) return false;   // empty
    *v = r->data[r->tail]; r->tail = (r->tail + 1) % BUF_SIZE; return true;
}
```
- 區分 full / empty：犧牲一格、或另存 `count`、或 head/tail 不取模只用 `head - tail`。
- **單一 producer + 單一 consumer**（例如 ISR 寫、main 讀）且 index 更新為 atomic 時可 lock-free；多 producer 就要鎖。

---

## 6. OS / RTOS

> [!important] Interrupt、Mutex/Semaphore/Spinlock 是最高頻；每次都會 follow-up 問細節。

### 6.1 基本觀念
- **Process vs Thread**：process 有獨立 address space；同 process 的 thread 共享 code/data/heap，各自有 stack 與 registers。thread 建立與切換較便宜，但共享資料需同步。
- **Kernel**：OS 核心，管理 CPU 排程、記憶體、I/O、中斷、system call；kernel mode vs user mode。
- **Atomic**：操作不可分割，不會被中斷或其他 thread 看到中間狀態。`i++` 不是 atomic（read-modify-write）。實作：關中斷、LDREX/STREX（ARM）、C11 `<stdatomic.h>`。

### 6.2 Memory Layout
| 區段        | 內容                                                   |
| --------- | ---------------------------------------------------- |
| `.text`   | 程式碼（通常在 Flash/ROM）                                   |
| `.rodata` | 常數、字串常量                                              |
| `.data`   | **有初始值**的 global / static 變數（初始值存 Flash，開機時複製到 RAM）  |
| `.bss`    | **未初始化或初始為 0** 的 global / static 變數（開機時清 0）          |
| Heap      | `malloc` 動態配置，往高位址長                                  |
| Stack     | 區域變數、參數、return address，往低位址長；stack overflow 會撞到 heap |

### 6.3 Embedded Boot Process
Power on / Reset → CPU 從 reset vector 取初始 SP 與 Reset_Handler（Cortex-M）→ 時脈 / 基本硬體初始化 → 把 `.data` 從 Flash 複製到 RAM、清 `.bss` → （C++ constructors）→ 呼叫 `main()`。
較複雜系統：ROM code → Bootloader（如 U-Boot，可做 firmware update / 驗簽 secure boot）→ Kernel → rootfs / app。

### 6.4 Interrupt & ISR（真的很重要）
**Interrupt 流程（越細越好，以 Cortex-M 為例）**：
1. 周邊產生 interrupt request → NVIC 判斷是否 enable 與優先權
2. CPU 完成當前指令（或可中斷的 multi-cycle 指令）
3. 硬體自動 **stacking**：把 R0–R3、R12、LR、PC、xPSR push 到目前 stack
4. 從 **vector table** 取 ISR 位址，LR 設為 EXC_RETURN，進入 Handler mode
5. 執行 ISR（清除 interrupt flag）
6. 返回：硬體 **unstacking**，恢復 registers，回到原程式
- 優化：tail-chaining、late arrival、nested interrupt（高優先權可搶佔低優先權）。

**ISR 設計原則**：
- **越短越快**：只做必要事（清 flag、讀資料進 buffer、設旗標 / 發 semaphore），重活丟給 task / bottom half（deferred processing）
- **不要放**：`printf`、`malloc/free`、blocking 呼叫（mutex lock、sleep、delay）、浮點大量運算、長 loop
- ISR 沒有參數也沒有回傳值
- 共享變數要 `volatile`，並保護 critical section
- **Reentrant？** 一般 ISR 不應依賴 reentrancy；同一 interrupt 通常在執行中不會再進（除非重新 enable），但 ISR 呼叫的函式必須 reentrant
- 用途：事件即時回應（按鍵、UART 收資料、timer tick、DMA 完成）而不用 polling

**Interrupt Latency**：從 interrupt 發生到 ISR 第一條指令執行的時間。
降低方法：縮短關中斷的時間 / critical section、提高 ISR 優先權、ISR 精簡、使用 tail-chaining 的硬體、把 ISR / vector table 放在快速記憶體（TCM/RAM）、避免長的不可中斷指令、關閉不必要的 cache miss 來源。

**Linux interrupt**：hardware IRQ vs softirq / tasklet / workqueue（top half / bottom half）。

### 6.5 Context Switch（不要只說「換個 thread」）
- **誰觸發**：timer tick（SysTick）時間片到、task 主動 yield / block（等 semaphore、delay）、高優先權 task 被喚醒。
- **怎麼做**：保存當前 task 的 CPU context（registers、PC、SP、status）到它的 **TCB / task stack** → scheduler 選下一個 task → 從它的 stack 恢復 context。
- **Cortex-M RTOS（如 FreeRTOS）**：
  - SysTick ISR 決定需要切換 → 設 **PendSV** pending
  - PendSV 設為**最低優先權**，確保在所有其他 ISR 結束後才切換
  - 進入 PendSV 時硬體已自動 push R0–R3, R12, LR, PC, xPSR；PendSV handler 軟體 push **R4–R11**（有 FPU 還有 S16–S31）到 PSP，把 PSP 存到 TCB → 載入新 task 的 PSP → pop R4–R11 → exception return 時硬體 pop 其餘
  - SVC 常用於啟動第一個 task
- Process switch 比 thread switch 多了：切換 page table / 地址空間、可能 flush TLB。

### 6.6 同步機制：Mutex / Semaphore / Spinlock
| | Mutex | Semaphore | Spinlock |
|---|---|---|---|
| 本質 | 互斥鎖，有 **ownership**（誰 lock 誰 unlock） | 計數器（binary / counting），**無 ownership** | busy-wait 的鎖 |
| 等待方式 | sleep（block） | sleep（block） | 忙等（spin） |
| 適用 | 保護 shared resource | 事件通知 / 同步（ISR → task）、管理 N 個資源 | 臨界區**極短**、多核心、不能 sleep 的情境（如 Linux interrupt context） |
| ISR 可用？ | ❌ | ✅ give（post）可以 | 多核可用（通常搭配關中斷 `spin_lock_irqsave`） |
| Priority inheritance | 通常有 | 通常無 | — |
- 單核心上 spinlock 沒意義（持鎖者無法同時執行）→ 用關中斷 / 關搶佔。
- Binary semaphore vs mutex：binary semaphore 可由別人 give（用於 signaling），mutex 必須由 owner 釋放。

### 6.7 Critical Section
- Task 之間：mutex / 關搶佔（scheduler suspend）。
- **ISR 與 task 共用變數**：mutex 不能在 ISR 用 → 在 task 端**短暫關中斷**（`taskENTER_CRITICAL` / `__disable_irq()`）存取，或用 atomic 操作、lock-free ring buffer、只由單一方寫入。
- 關中斷要**越短越好**（會增加 interrupt latency）。

### 6.8 Priority Inversion
- 情境：低優先權 L 持有 mutex → 高優先權 H 等這把鎖 → 中優先權 M 搶佔 L → H 被 M 間接卡住（著名案例：Mars Pathfinder）。
- 解法：**Priority Inheritance**（L 暫時繼承 H 的優先權）、**Priority Ceiling**（持鎖時提升到鎖的天花板優先權）、縮短 critical section、避免共享資源。

### 6.9 Deadlock
**四條件（同時成立才會發生）**：Mutual exclusion、Hold and wait、No preemption、Circular wait。
避免：破壞任一條件 —— **固定取鎖順序**（最常用）、一次取得所有資源、`trylock` + timeout 失敗就釋放、Banker's algorithm（避免）。
**Starvation**：某 task 長期得不到資源 → aging（等越久優先權越高）、公平排程、FIFO 鎖。
**Livelock**：大家都在讓步卻沒進展。

### 6.10 Reentrant vs Thread-safe
- **Reentrant**：可以在執行途中被中斷再進入（包括 ISR），結果仍正確。條件：不使用 static/global 變數（或只讀）、不回傳 static 資料的指標、不呼叫 non-reentrant 函式（如 `malloc`、`printf`、`strtok`）、只用參數與區域變數。
- **Thread-safe**：多 thread 同時呼叫結果正確，可以用鎖達成。
- 用 mutex 的函式是 thread-safe 但**不是 reentrant**（在 ISR 裡再進入會 deadlock）。`strtok` 兩者皆非；`strtok_r` 是 reentrant 版本。

### 6.11 Inter-Thread / Inter-Process Communication
- **ITC（同 process / RTOS task）**：shared memory + mutex、message queue、semaphore / event flags / event group、task notification、condition variable、mailbox、ring buffer。
- **IPC**：pipe / FIFO（named pipe）、message queue、shared memory（最快，需自行同步）、socket、signal、semaphore、memory-mapped file。
- 情境題選擇：
  - 大量資料、低延遲 → shared memory + 同步
  - 需要解耦、有序傳遞 → message queue
  - 只要通知事件 → semaphore / event flag / task notification
  - ISR 傳資料給 task → queue（FromISR API）或 ring buffer + semaphore

### 6.12 General-purpose OS vs RTOS
| | GPOS（Linux / Windows） | RTOS（FreeRTOS / Zephyr / ThreadX） |
|---|---|---|
| 目標 | 平均 throughput、公平 | **Deterministic**、可預測的最壞時間（deadline） |
| 排程 | 公平排程（CFS 等） | Priority-based preemptive |
| 記憶體 | MMU、virtual memory | 通常無 MMU（MPU）、footprint 小 |
| Latency | 不保證 | 有界 |
- **Hard vs soft real-time**：錯過 deadline 是系統失效 vs 只是品質下降。

### 6.13 RTOS 細節
- **Scheduler**：priority-based **preemptive**（高優先權 ready 立即搶佔）vs **cooperative**（task 主動 yield 才切換）；同優先權 Round Robin time slicing。
- 其他排程：FIFO、Round Robin、Priority、Rate Monotonic、EDF。
- **SysTick**：週期性 timer 中斷（如 1 ms），提供 tick、驅動時間片與 delay 計時。
- **誰做 context switch**：scheduler 決定，Cortex-M 上由 **PendSV** handler 執行。
- Task states：Running / Ready / Blocked / Suspended。
- Idle task、stack overflow 偵測、tickless idle（省電）、watchdog。

### 6.14 何時用 RTOS vs Bare-metal
- **Bare-metal（super loop + ISR）**：功能簡單、資源極小、時序單純、需要最低 overhead / 最小 footprint。
- **RTOS**：多個獨立且有不同時間要求的任務、需要優先權與 deadline 保證、網路 / USB / 檔案系統等 stack、團隊開發需要模組化。

### 6.15 Segmentation Fault
- 原因：dereference NULL / wild / dangling pointer、陣列越界、stack overflow（無限遞迴）、寫入唯讀記憶體（修改字串常量）、double free / use-after-free。
- Debug：gdb（`bt` 看 backtrace）、core dump、AddressSanitizer（`-fsanitize=address`）、Valgrind、加 log / assert。
- MCU 上的對應：**HardFault** → 讀 fault status registers（CFSR / HFSR / MMFAR / BFAR）與 stacked PC 找出錯位置。

### 6.16 其他可能被問
Virtual memory / paging / MMU / TLB、cache coherency、memory leak、shared memory、watchdog、DMA、power management。

---

## 7. Multithread

> [!note] 大廠越來越常考 multithread **coding**，要能**手寫 pthread 程式**。

### 7.1 pthread API
```c
#include <pthread.h>
int pthread_create(pthread_t *th, const pthread_attr_t *attr, void *(*fn)(void *), void *arg);
int pthread_join(pthread_t th, void **retval);
pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_lock(&m);  pthread_mutex_unlock(&m);
pthread_cond_t c = PTHREAD_COND_INITIALIZER;
pthread_cond_wait(&c, &m);   // 會自動 unlock，被喚醒後重新 lock；要放在 while 裡（spurious wakeup）
pthread_cond_signal(&c);  pthread_cond_broadcast(&c);
// semaphore: sem_init / sem_wait / sem_post
```

### 7.2 經典題：兩個 thread 交替印 1–50（奇 / 偶）
```c
#include <pthread.h>
#include <stdio.h>
#define MAX 50
static int cur = 1;
static pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
static pthread_cond_t  c = PTHREAD_COND_INITIALIZER;

static void *worker(void *arg) {
    int parity = *(int *)arg;           // 1 = odd, 0 = even
    while (1) {
        pthread_mutex_lock(&m);
        while (cur <= MAX && cur % 2 != parity) pthread_cond_wait(&c, &m);
        if (cur > MAX) { pthread_mutex_unlock(&m); pthread_cond_broadcast(&c); break; }
        printf("thread %s: %d\n", parity ? "odd" : "even", cur++);
        pthread_cond_broadcast(&c);
        pthread_mutex_unlock(&m);
    }
    return NULL;
}
int main(void) {
    pthread_t t1, t2; int odd = 1, even = 0;
    pthread_create(&t1, NULL, worker, &odd);
    pthread_create(&t2, NULL, worker, &even);
    pthread_join(t1, NULL); pthread_join(t2, NULL);
    return 0;
}
```
- 也要會「兩個不同 worker function」的版本，以及用兩個 semaphore 互相 post 的版本。

### 7.3 Producer–Consumer（bounded buffer）
- 條件：buffer 滿時 producer 等、空時 consumer 等；存取 buffer 要互斥。
- 解法 A：mutex + 2 個 condition variable（`not_full`、`not_empty`）
- 解法 B：mutex + 2 個 counting semaphore（`empty_slots` 初值 N、`full_slots` 初值 0）
- 注意：wait 放在 `while` 迴圈、避免 lost wakeup、取鎖順序避免 deadlock。

### 7.4 常見觀念題
- **只有一個 core 還需要 multithread 嗎？** 需要。I/O-bound 時可在等待 I/O 時切去做別的、提升 responsiveness、模組化設計；但 CPU-bound 不會變快，反而有 context switch overhead。
- **只有一個 thread 需要 mutex 嗎？** 通常不需要；但若有 **ISR** 或 signal handler 會存取同一資料，仍需保護（此時用關中斷而非 mutex）。
- Race condition、data race、false sharing、memory barrier / memory ordering。
- 可以請 ChatGPT / Claude 出幾個小題目自己實作（bank account、thread pool、reader-writer lock）。

---

## 8. Computer Architecture

### 8.1 Endianness（最重要）
- Big-endian：高位 byte 放低位址（network byte order）；Little-endian：低位 byte 放低位址（x86、多數 ARM）。
```c
int is_little_endian(void) { uint32_t x = 1; return *(uint8_t *)&x == 1; }
// 或用 union { uint32_t i; uint8_t c[4]; }

uint32_t swap32(uint32_t x) {
    return ((x & 0x000000FF) << 24) | ((x & 0x0000FF00) << 8) |
           ((x & 0x00FF0000) >> 8)  | ((x & 0xFF000000) >> 24);
}
uint16_t swap16(uint16_t x) { return (x << 8) | (x >> 8); }
```
- 相關：`htonl / ntohl`、通訊協定封包解析時的 byte order。

### 8.2 基本觀念
- **PC（Program Counter）**：下一條要執行的指令位址。另有 SP（stack pointer）、LR（link register，return address）。
- **RAM vs ROM**：RAM 揮發、可讀寫、放執行期資料（SRAM 快、DRAM 需 refresh）；ROM/Flash 非揮發、放程式與常數，Flash 寫入需先 erase、有寫入次數限制。
- **CISC vs RISC**：CISC（x86）指令長度可變、指令複雜、可直接操作記憶體；RISC（ARM、RISC-V）指令定長、簡單、load/store 架構、暫存器多、易 pipeline。
- **Harvard vs Von Neumann**：指令與資料分開匯流排 vs 共用。

### 8.3 5-Stage Pipeline
| Stage | 做什麼 |
|---|---|
| IF（Instruction Fetch） | 依 PC 從 instruction memory 取指令，PC += 4 |
| ID（Decode） | 解碼、讀 register file、sign-extend immediate |
| EX（Execute） | ALU 運算 / 計算記憶體位址 / branch 判斷 |
| MEM（Memory） | load / store 存取 data memory |
| WB（Write Back） | 寫回 register file |
- **`lw rt, offset(rs)`**：IF 取指令 → ID 讀 `rs`、sign-extend offset → EX 計算 `rs + offset` → MEM 從該位址讀資料 → WB 寫入 `rt`。
- **`sw rt, offset(rs)`**：EX 算位址 → MEM 把 `rt` 寫入記憶體 → 無 WB。
- **Hazards**：
  - Structural（硬體資源衝突）
  - **Data hazard 三種**：**RAW**（Read After Write，真相依，5-stage 唯一會真的發生的）、**WAR**（Write After Read）、**WAW**（Write After Write）—— 後兩者在 out-of-order 才會出現
  - Control（branch）
- 解法：forwarding / bypassing、stall（bubble，load-use hazard 必須 stall 1 cycle）、指令重排、branch prediction、delayed branch。

### 8.4 Cache（沒被問過，但要理解）
- **Mapping 三種**：Direct-mapped、Fully associative、N-way set associative；位址切成 tag / index / offset。
- **Miss 三種（3C）**：Compulsory（冷啟動）、Capacity（容量不足）、Conflict（映射衝突）。
- **Write policy**：Write-through（同時寫 memory，簡單但慢，常配 write buffer）vs Write-back（只寫 cache，dirty bit，被替換時才寫回）；write-allocate vs no-write-allocate。
- 替換策略：LRU、FIFO、Random。
- 嵌入式重點：**DMA 與 cache 一致性**（DMA 前 clean、DMA 後 invalidate）、多核 cache coherency（MESI）、`volatile` 不會繞過 cache。

### 8.5 MMU / TLB / Virtual Memory
- Virtual memory：每個 process 有獨立虛擬位址空間，由 page table 轉成實體位址；提供隔離與保護、可超過實體記憶體。
- MMU：硬體做位址轉換與權限檢查；Page fault 由 OS 處理。
- TLB：page table 的 cache，加速轉換；context switch 時可能需要 flush（或用 ASID）。
- MPU（MCU 上）：只做區域權限保護，不做轉換。

---

## 9. Peripheral / 通訊協定 / 硬體

> [!warning] 履歷上有寫 I2C / SPI / UART，就要準備被深挖。

### 9.1 比較
|      | UART                                | I2C                                          | SPI                               |
| ---- | ----------------------------------- | -------------------------------------------- | --------------------------------- |
| 線數   | TX、RX（+GND）                         | SDA、SCL（2 線）                                 | SCLK、MOSI、MISO、CS（每個 slave 一條 CS） |
| 同步   | **非同步**（雙方約定 baud rate）             | 同步                                           | 同步                                |
| 雙工   | Full duplex                         | Half duplex                                  | Full duplex                       |
| 拓撲   | 點對點                                 | Multi-master / multi-slave，用 **7/10-bit 位址** | 單 master、多 slave（靠 CS 選）          |
| 速度   | 較慢（常見 9600–115200 bps，可到 Mbps）      | 100k / 400k / 1M / 3.4 Mbps                  | **最快**（數十 MHz）                    |
| ACK  | 無（可有 parity）                        | 有 ACK / NACK                                 | 無                                 |
| 優點   | 簡單、長距離（搭配 RS-232/485）、debug console | 線少、可掛很多裝置、有位址與 ACK                           | 快、簡單、全雙工、無位址開銷                    |
| 缺點   | 只能點對點、需準確時脈                         | 慢、需上拉電阻、匯流排電容限制距離                            | 線多（CS 隨 slave 數增加）、無 ACK、無標準流量控制  |
| 典型用途 | Debug log、GPS、BT module             | Sensor、EEPROM、PMIC                           | Flash、Display、ADC、SD card         |

### 9.2 細節
- **UART frame**：Start bit（0）+ 5–9 data bits（LSB first）+ 可選 parity + 1–2 stop bits（1）。常見 8N1。錯誤：framing error、parity error、overrun。
- **I2C**：
  - Open-drain + **pull-up 電阻**（wired-AND）；電阻太大 → 上升時間慢，太小 → 耗電 / 驅動不了
  - START（SCL high 時 SDA 由高變低）、STOP（SCL high 時 SDA 由低變高）、Repeated START
  - 傳輸：START → 7-bit address + R/W → ACK → data bytes（每 byte 後 ACK）→ STOP
  - **Arbitration**：多個 master 同時傳，每個 master 邊送邊讀 SDA；送 1 卻讀到 0 的一方失去仲裁並退出（因為 open-drain，0 贏）。位址較小者勝，不會破壞資料。
  - **Clock stretching**：slave 還沒準備好時把 **SCL 拉低**，master 必須等 SCL 被放開才繼續。
- **SPI**：4 種 mode 由 **CPOL**（idle 時 clock 電位）與 **CPHA**（在第一或第二個邊緣取樣）決定，master/slave 必須一致；可 daisy chain。
- **Debug 某協定不 work**：
  1. 確認硬體：接線、GND 共地、電壓準位、pull-up（I2C）、CS 腳位
  2. 確認設定：baud rate / clock 頻率、SPI mode、I2C 位址（7-bit 是否左移）、data 格式、endianness
  3. 用 **Logic analyzer / 示波器** 看波形（有沒有 ACK、時序、雜訊、上升時間）
  4. Loopback 測試（UART TX 接 RX）
  5. 讀 status / error register、檢查 interrupt / DMA 設定、GPIO alternate function、clock enable
  6. 對照 datasheet 的 timing 需求，降速測試

### 9.3 其他硬體 / 系統題
- **GPIO**：input / output、push-pull vs open-drain、pull-up / pull-down、alternate function、debounce（按鍵去彈跳）。
- **JTAG / SWD**：debug 與燒錄介面；JTAG 4–5 線（TCK、TMS、TDI、TDO、TRST），SWD 2 線（SWDIO、SWCLK）。可做 boundary scan。
- **Hardware vs Software breakpoint**：HW 用 debug 暫存器比對位址（數量有限、可用於 Flash）；SW 把指令替換成 BKPT 指令（需可寫記憶體）。
- **DMA**：不經 CPU 在記憶體與周邊間搬資料，完成時發 interrupt；注意 cache coherency 與 buffer 對齊。
- **Watchdog**：定時要 kick，否則 reset，用於從當機恢復。
- **Firmware update（OTA / DFU）**：A/B 雙分區、bootloader 驗證簽章（hash + digital signature）、失敗 rollback、斷電保護。
- **基本電子**：電晶體（開關）、二極體、電容（去耦、濾波）、電阻分壓、馬達驅動（H-bridge、PWM）。
- **ADC / DAC / PWM / Timer**：解析度、取樣率、duty cycle。
- 其他：CAN、USB、PCIe、Ethernet / TCP-IP 基礎、安全（hash、encryption、digital signing、secure boot）。

---

## 10. 應用題 / Design 題

依公司與職缺 domain 不同，不一定比較難，但需要 domain knowhow 或思考轉個彎。

| 類型                    | 題目 / 重點                                                                                                                                                   |
| --------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 字串處理                  | 搜尋、比較、複製、`atoi`/`itoa`、**state machine** 解析                                                                                                               |
| 訊號處理                  | 用 **3×3 filter（kernel）** 對整張影像做 convolution 模糊化 / 銳利化 / 去雜訊（注意邊界）；moving average、去除特定頻率聲音                                                                 |
| Circular Buffer Queue | 基本不難，follow-up 是 producer / consumer 為不同 thread → mutex / semaphore                                                                                       |
| **封包處理**              | 例如 `[START][CMD][LEN][PARAM × n][CRC][END]`，模擬 BT / I2C / UART；可當字串題寫，但要考慮 **API 設計、容錯（長度錯、CRC 錯、缺 END、資料被截斷）、state machine 逐 byte 解析**、checksum / CRC 計算 |
| 控制系統                  | 給 `heat()` / `cool()` / `get_temp()` 三個 API，寫恆溫系統 → bang-bang + **hysteresis** → **PID**（P/I/D 各自作用、積分飽和 anti-windup）→ fuzzy                              |
| 排序 / 搜尋               | QuickSort、MergeSort、Binary Search（Array 與 Linked List 都要會）                                                                                                |
| memcpy 優化             | 對齊後 word copy、loop unrolling、DMA                                                                                                                          |
| Array 組合              | 找相加 / 相乘最大、或等於目標值的組合                                                                                                                                      |
| 矩陣                    | 內積、乘法、flatten                                                                                                                                             |
| 浮點                    | IEEE 754、實作浮點運算 / 定點數（fixed-point）運算                                                                                                                      |
| 系統設計                  | Traffic light（state machine）、智慧燈泡 IoT、keyboard matrix driver（掃描 + debounce）、device driver 介面設計、Gray code 系統                                               |

**API 設計原則**（可搜尋 Google / Facebook 的 API design 影片）：
- 清楚的命名與參數、回傳 error code、input validation（NULL、長度）
- 呼叫者配置記憶體 vs 函式內配置、ownership 清楚
- 可重入 / thread safety、避免 global state
- 例：`int parse_packet(const uint8_t *buf, size_t len, packet_t *out);` 回傳 0 / 錯誤碼

---

## 11. Behavioral

> [!note] 答案很個人化，公司期待也不同：有的希望你對未知問題**大膽推測**，有的希望「**知之為知之，不知為不知**」。先觀察面試官風格。

### 常見題目（準備故事）
- [ ] Self-introduction（1–2 分鐘）
- [ ] Why this company / role？Why job change？
- [ ] 最大成就 / 解決過最難的問題（技術深度）
- [ ] 失敗經驗 / code 出大 bug 怎麼解決
- [ ] 跟組員 / 主管意見不合怎麼處理
- [ ] 舉例證明團隊合作能力
- [ ] 客戶 deadline 突然提前怎麼辦
- [ ] 如何處理壓力 / 做決策 / 領導
- [ ] 接到新專案會怎麼開始
- [ ] 組裡有人一直拖後腿怎麼處理
- [ ] 需要改變做法的情境
- [ ] 把複雜技術解釋給非技術人員
- [ ] Why should we hire you?

### 準備方法
- 用 **STAR / SAR**（Situation → Action → Result），結果盡量量化。
- 準備 5–8 個萬用故事，對應不同題目。
- **寫下來 → 練習 → 錄影/錄音**檢查清晰度與速度；做 mock behavioral。
- 說話**慢、清楚、大聲**。

---

## 12. 資源

### 書
- *The C Programming Language*（K&R）—— 重點 Ch.1–6
- *Cracking the Coding Interview*（McDowell）—— firmware 可跳過 system design 與 graph

### 線上
- **jserv「你所不知道的 C 語言」系列講座**
- GeeksforGeeks：Bit manipulation（easy + medium）、OS fundamentals
- LeetCode：Linked List、Bit Manipulation、String、Array 標籤
- GitHub 上的 embedded interview / tutorial repo
- Google / Facebook 的 API design、mock interview 影片

### 原文連結
- [Cracking the Firmware / Embedded Systems Engineer Interview（Akash Agrawal, Apple FW）](https://medium.com/@akashagrawal_33749/cracking-the-firmware-embedded-systems-engineer-interview-d73a37da95bd)
- [程式面試的 7 個階段與 2 個關鍵能力（holyisland）](https://holyisland.blog/coding-interview-steps/)
- [Coding Interview 怎麼跟面試官全程保持互動（Yu-Chien）](https://yuchien.medium.com/coding-interview-%E6%80%8E%E9%BA%BC%E8%B7%9F%E9%9D%A2%E8%A9%A6%E5%AE%98%E5%85%A8%E7%A8%8B%E4%BF%9D%E6%8C%81%E4%BA%92%E5%8B%95-d178afa265c6)

---

## 13. 準備 Checklist

### 優先順序（依出現頻率）
1. **C 語言關鍵字與觀念**（static / volatile / const / struct alignment / pointer / sizeof / macro）
2. **Linked List** 全套（含 recursive reverse）
3. **Bit Manipulation**
4. **OS**：Interrupt / ISR、Mutex vs Semaphore vs Spinlock、Priority inversion、Deadlock、Context switch、Memory layout
5. **String / Array** 手寫（strlen/strcpy/strcmp/memmove/atoi/itoa）
6. **Multithread coding**（pthread 奇偶印、producer-consumer）
7. **Data structure**：Stack / Queue / Ring buffer（用 linked list 實作）
8. **Peripheral**：UART / I2C / SPI 比較與 debug
9. **Computer Architecture**：Endian、pipeline、hazard、cache
10. **應用題**：封包解析、PID、image filter、state machine
11. **Behavioral** 故事

### 練習習慣
- [ ] 每天 2–3 題，完整寫完（不看答案）＋ 邊寫邊講
- [ ] 每題都列 edge cases、自己 dry run
- [ ] 白板 / CoderPad / 純文字編輯器練習（沒有 autocomplete）
- [ ] 定期 mock interview（技術 + behavioral）
- [ ] 履歷上每個 project 準備「30 秒版本」與「深挖版本」
- [ ] 每場面試後記錄題目與不會的地方，回頭補
