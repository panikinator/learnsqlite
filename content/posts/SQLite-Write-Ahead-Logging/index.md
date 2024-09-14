+++
title = 'SQLite Write Ahead Logging'
date = 2024-09-12T16:09:02+05:30
draft = false
+++
# Mastering SQLite Write-Ahead Logging (WAL): Use Cases, Best Practices, and Performance Benchmarks

---

By default, SQLite uses a rollback journal mode to ensure atomicity, consistency, isolation, and durability (ACID) compliance. However, SQLite offers an alternative journaling mode called Write-Ahead Logging (WAL), which dramatically improves write performance and allows concurrent read and write operations.

In this post, we’ll dive deep into SQLite’s WAL mode, explaining how it works, when to use it, and its pros and cons. We’ll also explore benchmarks to see how it performs in different scenarios.


---

## What is SQLite Write-Ahead Logging (WAL)?

The Write-Ahead Logging (WAL) mode in SQLite is an advanced journaling mechanism designed to improve performance and concurrency. In the default rollback journal mode, every write transaction locks the entire database, preventing concurrent reads and writes. WAL mode, on the other hand, writes changes to a separate WAL file, allowing reads and writes to occur simultaneously.

In WAL mode, the database works as follows:

- Write Operations: Instead of writing directly to the main database file, changes are written to a separate WAL file.

- Checkpointing: When the WAL file grows beyond a certain size (default: 1,000 pages), SQLite merges changes from the WAL file back into the main database file in a process called "checkpointing."

- Read Operations: Readers continue accessing the original database file while changes accumulate in the WAL file with minimal performance degradation compared to rollback mode.


This separation between readers and writers removes the bottleneck caused by exclusive locking, improving write performance and enabling more parallelism.


---

## How to Enable WAL Mode in SQLite

Activating WAL mode in SQLite is straightforward. You can enable it for a specific database by running the following SQL command:

```SQL
PRAGMA journal_mode = WAL;
```
This command switches the database to WAL mode and creates a WAL file next to the main database file.

You can also check the current journaling mode with:

```SQL
PRAGMA journal_mode;
```

To return to the default rollback journal mode, you can execute:

```SQL
PRAGMA journal_mode = DELETE;
```


---

## How Does WAL Mode Work?

WAL mode fundamentally changes how SQLite manages database changes. Instead of writing directly to the database file, SQLite writes transactions into the WAL file. Let’s break down the key components of WAL mode:

1. WAL File

Temporary File: The WAL file stores changes until they are merged back into the main database file during a checkpoint.

Efficient Writes: Since SQLite appends data to the WAL file, the write process is faster, avoiding costly file-locking operations that occur in rollback journal mode.


2. Checkpointing

Periodic Merging: During a checkpoint, SQLite applies the changes from the WAL file to the main database. Checkpointing can happen automatically or manually.

Optimization: You can tune when checkpointing occurs (e.g., by adjusting the size of the WAL file) to balance performance and durability.


3. Readers and Writers

Concurrent Reads/Writes: Unlike rollback journal mode, where writes block reads, WAL mode allows multiple readers and one writer to work concurrently.

Read Consistency: Readers in WAL mode access the unchanged database file, ensuring they get a consistent view of the data.



---

## When to Use WAL Mode

WAL mode offers significant benefits in various scenarios, but it’s not suitable for every use case. Here are some ideal situations where WAL mode excels:

1. High Write Performance

WAL mode significantly improves write performance by avoiding the need to lock the entire database during each transaction. If your application is write-heavy, WAL mode is a good fit.

2. Concurrent Reads and Writes

In applications that require concurrent reads and writes, such as web applications or real-time data processing systems, WAL mode allows for more efficient concurrency without sacrificing read performance.

3. Embedded Systems and Mobile Applications

Many mobile apps and embedded systems (e.g., IoT devices) use SQLite for data storage. These applications often benefit from WAL mode's better concurrency and faster writes, especially when dealing with high-frequency data collection.

4. Long-Running Reads

In some applications, you may have long-running read transactions that block other operations. WAL mode allows long-running reads to continue without affecting write performance.


---

## When Not to Use WAL Mode

While WAL mode offers great advantages, there are some scenarios where it may not be the best option:

1. Limited Disk Space

WAL mode requires additional disk space because it creates a separate WAL file for write transactions. If your application is running in an environment with limited disk space, such as embedded devices, this might be a concern. However, the WAL file is truncated periodically, so this is usually a manageable issue.

2. Single-Threaded Environments

If your application is single-threaded and doesn’t require concurrent reads and writes, the performance improvements of WAL mode may be negligible. In such cases, the default rollback journal mode may suffice.

3. External Storage (e.g., Network Filesystems)

WAL mode might not work efficiently or not at all on network-mounted or cloud-based filesystems such as SMB or NFS. SQLite recommends caution when using WAL mode in environments where the file system might not provide fast access or low-latency storage.

4. Frequent Checkpoints

WAL mode works best when checkpoints are infrequent, allowing the WAL file to grow before merging changes into the main database. Applications that trigger frequent checkpoints (due to low checkpoint thresholds or explicit requests) may see diminished benefits, as frequent disk I/O will negate WAL’s write performance gains.


---

## Performance Benchmarks
The benchmarks were performed on my laptops, the figures are only meant to be seen relatively.
I used the following python script to benchmark the database
```python
import concurrent.futures
import os
import random
import sqlite3
import time


def create_table(conn):
    cursor = conn.cursor()
    cursor.execute(
        """CREATE TABLE IF NOT EXISTS test_table
                      (id INTEGER PRIMARY KEY, value TEXT)"""
    )
    conn.commit()


def insert_data(db_file, num_records):
    conn = sqlite3.connect(db_file)
    cursor = conn.cursor()
    for i in range(num_records):
        cursor.execute("INSERT INTO test_table (value) VALUES (?)", (f"Value {i}",))
    conn.commit()
    conn.close()


def read_data(db_file, num_reads):
    conn = sqlite3.connect(db_file)
    cursor = conn.cursor()
    for _ in range(num_reads):
        cursor.execute(
            "SELECT * FROM test_table WHERE id = ?", (random.randint(1, 10000),)
        )
        cursor.fetchone()
    conn.close()


def concurrent_operations(
    db_file, num_writers, num_readers, records_per_writer, reads_per_reader
):
    with concurrent.futures.ThreadPoolExecutor(
        max_workers=num_writers + num_readers
    ) as executor:
        # Start write operations
        write_futures = [
            executor.submit(insert_data, db_file, records_per_writer)
            for _ in range(num_writers)
        ]

        # Start read operations
        read_futures = [
            executor.submit(read_data, db_file, reads_per_reader)
            for _ in range(num_readers)
        ]

        # Wait for all operations to complete
        concurrent.futures.wait(write_futures + read_futures)


def run_benchmark(
    db_file, enable_wal, num_writers, num_readers, records_per_writer, reads_per_reader
):
    if os.path.exists(db_file):
        os.remove(db_file)

    conn = sqlite3.connect(db_file)

    if enable_wal:
        conn.execute("PRAGMA journal_mode=WAL")

    create_table(conn)
    conn.close()

    start_time = time.time()
    concurrent_operations(
        db_file, num_writers, num_readers, records_per_writer, reads_per_reader
    )
    end_time = time.time()

    total_time = end_time - start_time
    total_writes = num_writers * records_per_writer
    total_reads = num_readers * reads_per_reader

    return total_time, total_writes / total_time, total_reads / total_time


def main():
    num_writers = 12
    num_readers = 40
    records_per_writer = 2000
    reads_per_reader = 1000

    # Run benchmark without WAL
    time_no_wal, writes_no_wal, reads_no_wal = run_benchmark(
        "test_no_wal.db",
        False,
        num_writers,
        num_readers,
        records_per_writer,
        reads_per_reader,
    )

    # Run benchmark with WAL
    time_wal, writes_wal, reads_wal = run_benchmark(
        "test_wal.db",
        True,
        num_writers,
        num_readers,
        records_per_writer,
        reads_per_reader,
    )

    print(f"Without WAL:")
    print(f"  Total time: {time_no_wal:.2f} seconds")
    print(f"  Writes per second: {writes_no_wal:.2f}")
    print(f"  Reads per second: {reads_no_wal:.2f}")
    print(f"\nWith WAL:")
    print(f"  Total time: {time_wal:.2f} seconds")
    print(f"  Writes per second: {writes_wal:.2f}")
    print(f"  Reads per second: {reads_wal:.2f}")

    print(f"\nWAL vs No WAL comparison:")
    print(f"  Total time improvement: {(time_no_wal / time_wal - 1) * 100:.2f}%")
    print(f"  Write speed improvement: {(writes_wal / writes_no_wal - 1) * 100:.2f}%")
    print(f"  Read speed improvement: {(reads_wal / reads_no_wal - 1) * 100:.2f}%")


if __name__ == "__main__":
    main()

```

And here are the results:
|Mode|Writes/s|Reads/s|
|---|---|---|
|Normal|8408|14014|
|WAL|9768|16281|

---


## Conclusion:
If you run a non fault tolerant app then continue using the default otherwise WAL is a no brainer for most people.
