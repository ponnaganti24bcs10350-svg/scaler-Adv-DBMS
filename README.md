# Advanced DBMS - Lab Assignment

**Name:** ponnaganti srividya  
**Role Number:** 10352  

---

### Part 1: SQLite3 Testing

To start the experiment, I created a new SQLite database (`sample.db`) and used a short Python script to populate it with 1 million mock user records. 

**Checking File Size:**
Command: `ls -lh sample.db`
Result: The file size came out to exactly 43 MB.

**Page Size and Page Count:**
I used the PRAGMA commands to check the internal structure:
- `sqlite3 sample.db "PRAGMA page_size;"` returned `4096` (4 KB blocks).
- `sqlite3 sample.db "PRAGMA page_count;"` returned `10894` total pages.

**Testing `mmap_size`:**
By default, memory mapping was turned off (`0`):
`sqlite3 sample.db "PRAGMA mmap_size;"`

I increased it to 256 MB to see what would happen:
`sqlite3 sample.db "PRAGMA mmap_size=268435456; PRAGMA mmap_size;"`

**Query Speed Results:**
I ran a standard `SELECT * FROM users;` and dumped the output to `/dev/null` while timing it.
- **Normal execution (mmap off):** `0.339 seconds`
- **With mmap enabled:** `0.322 seconds`
*(I monitored this by running `ps aux | grep sqlite` in a separate terminal).*
Conclusion: Memory mapping provided a slight speed improvement.

---

### Part 2: PostgreSQL Setup

For the second part of the lab, I installed Postgres using Homebrew, started the server, and created a `sampledb` database. To ensure an apples-to-apples comparison, I also inserted 1 million rows into Postgres, this time utilizing the built-in `generate_series()` function.

**Page Info:**
- Command: `psql -d sampledb -c "SHOW block_size;"`
- Result: `8192` (Postgres uses 8 KB blocks).
- Command: `psql -d sampledb -c "VACUUM ANALYZE users;"` followed by `psql -d sampledb -c "SELECT relpages FROM pg_class WHERE relname = 'users';"`
- Result: `14286` pages.

**Query Speed:**
Command: `time psql -d sampledb -c "SELECT * FROM users;" > /dev/null`
Result: `3.572 seconds`

---

### Part 3: Final Comparison & Analysis

| Feature | SQLite3 | PostgreSQL |
| :--- | :--- | :--- |
| **Page Size** | 4 KB | 8 KB |
| **Total Pages** | 10,894 | 14,286 |
| **Storage Size** | ~43 MB | ~111 MB |
| **Query Time** | 0.339s | 3.572s |

**Key Takeaways:**

1. **Storage Differences:** Even though both databases contain exactly 1 million rows, PostgreSQL is significantly larger in file size (about 111 MB compared to SQLite's 43 MB). This happens because PostgreSQL implements MVCC (Multi-Version Concurrency Control), which requires it to store hidden metadata columns (like `xmin` and `xmax` transaction IDs) for every single row.
2. **Speed Differences:** The `SELECT *` query executed much faster on SQLite. This is expected because SQLite is an embedded database running in the same process space. Postgres operates on a client-server model, meaning the data had to be formatted and sent over a local socket to the terminal, adding massive overhead to the read time.
3. **mmap Behavior:** Setting a custom `mmap_size` in SQLite improved performance because it allowed the engine to fetch pages directly from the operating system's memory cache, skipping traditional `read()` operations. PostgreSQL handles memory differently using a shared buffer pool, so it doesn't use `mmap` in the same direct way.
