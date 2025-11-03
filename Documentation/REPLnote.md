InputBuffer* input_buffer = new_input_buffer();
```

This calls `new_input_buffer()` which:
- Allocates memory for an `InputBuffer` structure (let's say at memory address `0x1000`)
- Sets `buffer = NULL` (no memory allocated yet for actual text)
- Sets `buffer_length = 0` (no space allocated)
- Sets `input_length = 0` (no text received)
- Returns the pointer `0x1000`

**Visual representation:**
```
input_buffer (at 0x1000):
├─ buffer: NULL
├─ buffer_length: 0
└─ input_length: 0

Infinity loop starts - 
while (true) {
    print_prompt();

----------------
ssize_t bytes_read = getline(&(input_buffer->buffer), 
                              &(input_buffer->buffer_length), 
                              stdin);
```

**What `getline()` does (this is crucial):**

- `getline()` is a special function that **automatically manages memory**
- You pass it:
  - `&(input_buffer->buffer)` - **address of the pointer** (so getline can modify where buffer points)
  - `&(input_buffer->buffer_length)` - **address of the size variable** (so getline can update it)
  - `stdin` - read from standard input

**Step-by-step process:**

a) `getline()` sees that `buffer` is `NULL` and `buffer_length` is `0`

b) `getline()` says: "I need to allocate memory!" 
   - Allocates, say, 120 bytes at address `0x2000`
   - **Updates** `input_buffer->buffer = 0x2000`
   - **Updates** `input_buffer->buffer_length = 120`

c) `getline()` reads characters from stdin: `'h' 'e' 'l' 'l' 'o' '\n'` (6 bytes)

d) `getline()` stores them in the buffer:
```
   buffer (at 0x2000): "hello\n"
```

e) `getline()` returns `6` (number of bytes read including the newline)

**Now our structure looks like:**
```
input_buffer (at 0x1000):
├─ buffer: 0x2000 → "hello\n"
├─ buffer_length: 120
└─ input_length: 0 (not updated yet)

----------------
Next Line - 
----------------
input_buffer->input_length = bytes_read - 1;  // 6 - 1 = 5
input_buffer->buffer[bytes_read - 1] = 0;     // Replace '\n' with '\0'
```

**After this:**
```
input_buffer (at 0x1000):
├─ buffer: 0x2000 → "hello\0"
├─ buffer_length: 120
└─ input_length: 5

-----------------
Goes back to the main
----------------------
if (strcmp(input_buffer->buffer, ".exit") == 0)
--------------------
printf("Unrecognized command '%s'.\n", input_buffer->buffer);

-------------------
Scenario 2: You Type "world" and Press Enter
7. read_input(input_buffer) is called again:

ssize_t bytes_read = getline(&(input_buffer->buffer), 
                              &(input_buffer->buffer_length), 
                              stdin);
```

**THIS TIME IT'S DIFFERENT!**

a) `getline()` sees:
   - `buffer` is NOT NULL (it's `0x2000`)
   - `buffer_length` is `120`
   - **So it REUSES the existing buffer!**

b) `getline()` reads: `'w' 'o' 'r' 'l' 'd' '\n'` (6 bytes)

c) `getline()` **OVERWRITES** the old content:
```
   Before: buffer (at 0x2000): "hello\0"
   After:  buffer (at 0x2000): "world\n"
```

d) Returns `6`

**After processing:**
```
input_buffer (at 0x1000):
├─ buffer: 0x2000 → "world\0"  (OVERWROTE "hello")
├─ buffer_length: 120 (still same)
└─ input_length: 5
```

**Key Point:** The old string "hello" is **GONE**. It was overwritten. There's only **ONE** buffer that gets reused.

## Scenario 3: You Type ".exit" and Press Enter

**8. `read_input(input_buffer)` is called:**

Same process:
- `getline()` reuses the buffer at `0x2000`
- Overwrites "world" with ".exit\n"
- Returns `6`

**After processing:**
```
input_buffer (at 0x1000):
├─ buffer: 0x2000 → ".exit\0"
├─ buffer_length: 120
└─ input_length: 5

if (strcmp(input_buffer->buffer, ".exit") == 0)  // TRUE!

close_input_buffer(input_buffer);
exit(EXIT_SUCCESS);

close_input_buffer()

free(input_buffer->buffer);  // Frees the memory at 0x2000
free(input_buffer);          // Frees the structure at 0x1000
```

**11. Program exits successfully**

## What If You Type Something REALLY Long?

Let's say you type 200 characters:

**`getline()` is smart:**

1. Sees buffer is 120 bytes (not enough!)
2. Calls `realloc()` to **resize** the buffer to, say, 256 bytes
3. Might move it to a new address like `0x3000`
4. **Updates** `input_buffer->buffer = 0x3000`
5. **Updates** `input_buffer->buffer_length = 256`
6. Copies your 200 characters there

## Summary: Memory Management

**Where is data saved?**
- In **heap memory** (dynamic memory allocated by `malloc`/`getline`)
- **NOT** on disk - only in RAM
- The buffer address (like `0x2000`) stays the same unless resized

**How many copies exist?**
- Only **ONE** buffer at a time
- Each new input **overwrites** the previous one
- Previous inputs are **lost forever** (unless you copy them elsewhere)

**The buffer lifecycle:**
```
Start → NULL
First input → Allocated (120 bytes)
Second input → Reused (same 120 bytes, content overwritten)
Third input → Reused again
...
.exit → Freed (memory returned to system)


------------------------------------
-----------------------------------
what happen inside the getline() - 
-----------------------------------
----------------------------------
ssize_t getline(char **lineptr, size_t *n, FILE *stream) {
    char *buffer = *lineptr;
    size_t capacity = *n;
    
    // If buffer is NULL, allocate initial memory
    if (buffer == NULL) {
        capacity = 120;  // or some default size
        buffer = malloc(capacity);
        *lineptr = buffer;
        *n = capacity;
    }
    
    size_t pos = 0;
    int c;
    
    while ((c = fgetc(stream)) != EOF && c != '\n') {
        // If we're running out of space, RESIZE!
        if (pos >= capacity - 1) {
            capacity *= 2;  // double the size
            buffer = realloc(buffer, capacity);  // ← HERE!
            *lineptr = buffer;  // update the pointer
            *n = capacity;      // update the size
        }
        
        buffer[pos++] = c;
    }
    
    if (c == '\n') {
        buffer[pos++] = '\n';
    }
    
    buffer[pos] = '\0';
    return pos;
}
```

**Key points:**

1. **`getline()` calls `realloc()` for you** - it's hidden inside the function
2. When the input is too long, `getline()` automatically:
   - Calls `realloc()` to grow the buffer
   - Updates `*lineptr` to point to the new (possibly moved) buffer
   - Updates `*n` to reflect the new size
3. **You pass pointers to pointers** (`&(input_buffer->buffer)`) so `getline()` can modify where your buffer pointer points

**Example with a long input:**
```
Initial state:
input_buffer->buffer = NULL
input_buffer->buffer_length = 0

After typing 200 characters:

getline() internally:
1. malloc(120)           → buffer at 0x2000
2. Read 120 chars        → buffer full!
3. realloc(buffer, 240)  → might move to 0x3000
4. Read 80 more chars    → done!
5. Update your pointers:
   *lineptr = 0x3000
   *n = 240

Result:
input_buffer->buffer = 0x3000 (updated by getline!)
input_buffer->buffer_length = 240 (updated by getline!)
---------------------------------------------
--------------------------------------------

IN C -

input_buffer->buffer[5] = 0;      // These three lines
input_buffer->buffer[5] = '\0';   // do EXACTLY
input_buffer->buffer[5] = (char)0; // the same thing!
-------------------------------------------------