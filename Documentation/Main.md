Example: SELECT statement becomes bytecode like:
-------------------------------------------------
OpenRead table=users
Rewind cursor
Next row
Column 0 → register
Column 1 → register
ResultRow
Close cursor
```

---

## **SQL Compiler (Gray Box)**

This converts SQL text into executable bytecode.

### **4. Tokenizer (Lexer)**
- **What it is:** Breaks SQL into tokens
- **What it does:**
  - Takes raw SQL string: `"SELECT name FROM users WHERE id = 5"`
  - Splits into tokens: `[SELECT] [name] [FROM] [users] [WHERE] [id] [=] [5]`
  - Identifies keywords, identifiers, operators, literals
  - Removes whitespace and comments

**Example:**
```
Input:  "SELECT * FROM users;"
Output: [SELECT] [*] [FROM] [users] [;]
```

### **5. Parser**
- **What it is:** Builds a syntax tree (AST - Abstract Syntax Tree)
- **What it does:**
  - Takes tokens and checks grammar rules
  - Creates a tree structure representing the query
  - Validates syntax (ensures SQL is well-formed)
  - Catches syntax errors

**Example:**
```
SELECT name FROM users WHERE id = 5

Becomes a tree:
     SELECT
      /  \
   name   FROM
           |
         users
           |
         WHERE
          / \
        id   =
              |
              5
```

### **6. Code Generator**
- **What it is:** Converts the parse tree into bytecode
- **What it does:**
  - Takes the AST and produces VM instructions
  - Optimizes the query plan
  - Generates efficient bytecode for the VM to execute
  - Decides how to access data (full scan vs index)

**Example:**
```
Parse tree → Bytecode:
Init
OpenRead 0 2 0 users     # Open table 'users'
Rewind 0 10              # Go to first row
Column 0 0 1             # Read 'id' column
Ne 5 9 1                 # If id != 5, jump to label 9
Column 0 1 2             # Read 'name' column
ResultRow 2 1            # Return result
Next 0 3                 # Go to next row
Close 0                  # Close cursor
Halt                     # Done
```

---

## **Backend (Blue Box)**

This is where data is actually stored and retrieved.

### **7. B-Tree**
- **What it is:** The core data structure for storing data
- **What it does:**
  - Organizes data in a balanced tree structure
  - Allows fast searching, inserting, deleting (O(log n))
  - Each table is stored as a B-Tree
  - Each index is also stored as a B-Tree
  - Handles both data pages and index pages

**Why B-Tree?**
- Keeps data sorted
- Efficient for range queries
- Minimizes disk reads (important for large databases)
- Self-balancing (stays efficient as data grows)

**Structure:**
```
        [Root Node]
         /       \
   [Internal]   [Internal]
    /    \       /    \
[Leaf] [Leaf] [Leaf] [Leaf]  ← Actual data here
```

### **8. Pager**
- **What it is:** The memory/disk management layer
- **What it does:**
  - Manages **pages** (fixed-size chunks of data, typically 4KB)
  - Implements a **page cache** (keeps frequently used pages in RAM)
  - Handles reading pages from disk
  - Handles writing pages to disk
  - Implements **lazy writing** (batch writes for performance)
  - Manages the **buffer pool**

**Example:**
```
VM needs row 1000:
1. Pager checks cache: Is page containing row 1000 in memory?
2. If YES: Return from cache (fast!)
3. If NO: Read page from disk → cache it → return it
```

### **9. OS Interface**
- **What it is:** Abstraction layer for file system operations
- **What it does:**
  - Provides platform-independent file I/O
  - Handles `open()`, `read()`, `write()`, `close()`, `fsync()`
  - Manages file locking (for concurrent access)
  - Handles journal files (for transactions/crash recovery)
  - Works across different operating systems (Linux, Windows, Mac)

**Why separate this?**
- Different OSes have different file APIs
- Easier to port database to new platforms
- Can optimize for each OS's peculiarities

---

## **Accessories (Bottom Right)**

### **10. Utilities**
- **What it is:** Helper functions and tools
- **What it does:**
  - Memory allocation wrappers
  - Hash functions
  - Date/time functions
  - String manipulation
  - Random number generation
  - Encoding/decoding (UTF-8, etc.)

### **11. Test Code**
- **What it is:** Testing framework
- **What it does:**
  - Unit tests for each component
  - Integration tests
  - Regression tests
  - Performance benchmarks
  - Ensures code quality and correctness

---

## **Complete Flow Example: `INSERT INTO users VALUES (1, 'Alice');`**

Let me trace this through the entire system:

**1. Interface** receives the input string

**2. SQL Command Processor** sees it's SQL (not a meta-command like `.exit`)

**3. Tokenizer** breaks it down:
```
[INSERT] [INTO] [users] [VALUES] [(] [1] [,] ['Alice'] [)] [;]
```

**4. Parser** creates AST:
```
    INSERT
      |
    users
      |
   VALUES
    /  \
   1   'Alice'
```

**5. Code Generator** produces bytecode:
```
Transaction Begin
OpenWrite 0 users
NewRowId → register 1
String 'Alice' → register 2
MakeRecord registers 1,2 → register 3
Insert table=0, key=register 3
Close 0
Transaction Commit