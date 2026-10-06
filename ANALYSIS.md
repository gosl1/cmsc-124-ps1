## Q1 SIAN

Pick 3 of the 10 categories. For each, pick a language that gives it to you for free and say what that
language pays for it. “Python has dictionaries” isn’t an answer. What does Python’s dictionary cost
in memory or in speed compared to what you built, and where would you notice?

### Strings in JavaScript

JavaScript provides built-in strings with Unicode support, automatic garbage collection, and immutable sequence operations. Like our `dt_str`, length checks are O(1) field reads rather than O(n) null-byte scans.

Every JavaScript string is an engine object carrying tens of bytes of metadata headers. Our `dt_str` maintains raw bytes alongside a minimal 24-byte header (`bytes` pointer, `length`, `capacity`).

JavaScript string concatenation creates new heap objects. Our `dt_str_append` resizes geometrically and appends in place in O(1) amortized time, paying for that speed with spare allocated capacity.

The difference can be noticed in high-throughput buffer operations where JavaScript's object overhead and garbage collection trigger memory bloat and latency pauses.

---

### Integers in Python

Python provides arbitrary-precision integers that automatically expand in memory to prevent numeric overflow.

Before arithmetic operations (`+`, `-`, `*`), our `dt_int` manually checks operands against `LLONG_MAX` and `LLONG_MIN` to return `DT_ERR_OVERFLOW` if a value exceeds 64 bits. A Python `int` is a dynamic heap `PyObject` consuming ~28 to 32+ bytes compared to our raw 8-byte `long long`.

Python integer addition requires interpreter dispatch and dynamic object allocations whose execution time scales with the number of digits. Our `dt_int` executes pre-checks and a single hardware instruction in strictly constant time O(1).

You would notice the difference in tight numerical loops where Python's dynamic allocation overhead degrades performance.

---

### Records in Python

In our `dt_record`, fields are fixed at creation in `dt_record_new`, and attempting to access or set an undeclared field returns `DT_ERR_FIELD`. In Python, writing `person.salary = 1` dynamically attaches a new attribute to the object at runtime rather than throwing an error.

Python objects maintain dynamic attribute mappings, carrying significant per-instance memory overhead. Our `dt_record` uses a static fixed-size layout capped at `DT_RECORD_MAX_FIELDS` (8) that stores field names and values in sequential array slots.

Looking up a field in `dt_record` performs a linear array scan (`strcmp`) over up to 8 slots. Python uses hashtable lookups.

You would notice the difference when a typo like `person.agee = 30` silently creates a redundant attribute in Python instead of failing.


## Q4 SIAN

Compare access after release with an allocation that remains unreleased at the driver’s final check. What damage can each cause in a long-running server? How does that answer change for a command-line tool that exits in a second?

- A memory leak occurs when memory is allocated and we lose a pointer to that memory or we keep memory we no longer need. Because of this, your program's memory usage (Resident Set Size / RSS) continuously climbs over time. Ultimately, the Operating System's Out-Of-Memory (OOM) Killer steps in and forcibly terminates your process. A Use-After-Free (UAF) is reading or writing memory that is already freed. This can cause undefined behavior because the memory allocator may have already reallocated. This could lead to unpredictable data corruption, sudden crashes, and severe security vulnerabilities (such as attackers exploiting reused memory blocks to steal session data or overwrite control flow pointers). For a short-lived CLI tool that runs in 1 second, if it leaks any memory, the memory stays trapped only while the program is executing. However, UAF is still critical. Running for only one second doesn't stop a program from outputting corrupted data, throwing a segmentation fault, or being exploited by bad inputs before it exits.