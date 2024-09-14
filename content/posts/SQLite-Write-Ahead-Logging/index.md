+++
title = 'SQLite Write Ahead Logging'
date = 2024-09-12T16:09:02+05:30
draft = false
author = ['panikinator']
tags = ['sqlite', 'performance']
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
The benchmarks were performed on a proxmox LXC with 1 CPU and 512MB of RAM, the figures are only meant to be judged relatively.
I used the following go script to benchmark the database as go is a good choice for concurrent programs 
```go
package main

import (
	"database/sql"
	"fmt"
	"math/rand"
	"os"
	"sync"
	"time"

	_ "github.com/mattn/go-sqlite3"
)

const (
	dbPath        = "./test.db"
	numOperations = 1000
	numWorkers    = 10
)

func main() {
	// Test without WAL
	err := os.Remove(dbPath)
	if err != nil {
		fmt.Println(err)
	}
	fmt.Println("Testing without WAL:")
	runTest(false)

	if err != nil {
		fmt.Println(err)
	}
	// Test with WAL
	fmt.Println("\nTesting with WAL:")
	runTest(true)
}

func runTest(useWAL bool) {
	db, err := sql.Open("sqlite3", dbPath)
	if err != nil {
		panic(err)
	}
	defer db.Close()

	if useWAL {
		_, err = db.Exec("PRAGMA journal_mode=WAL;")
		if err != nil {
			panic(err)
		}
	}

	// Create table
	_, err = db.Exec(`CREATE TABLE IF NOT EXISTS test_table (id INTEGER PRIMARY KEY, value TEXT);`)
	if err != nil {
		panic(err)
	}

	// Clear table
	_, err = db.Exec(`DELETE FROM test_table;`)
	if err != nil {
		panic(err)
	}

	start := time.Now()

	var wg sync.WaitGroup
	for i := 0; i < numWorkers; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := 0; j < numOperations/numWorkers; j++ {
				if rand.Float32() < 0.5 {
					// Write operation
					_, err := db.Exec("INSERT INTO test_table (value) VALUES (?)",
						randomString(10))
					if err != nil {
						fmt.Printf("Write error: %v\n", err)
					}
				} else {
					// Read operation
					var value string
					q := "SELECT value FROM test_table ORDER BY RANDOM() LIMIT 1"
					err := db.QueryRow(q).Scan(&value)
					if err != nil && err != sql.ErrNoRows {
						fmt.Printf("Read error: %v\n", err)
					}
				}
			}
		}()
	}

	wg.Wait()

	duration := time.Since(start)
	fmt.Printf("Time taken: %v\n", duration)
	fmt.Printf("Operations per second: %.2f\n", float64(numOperations)/duration.Seconds())
}

func randomString(length int) string {
	const charset = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
	b := make([]byte, length)
	for i := range b {
		b[i] = charset[rand.Intn(len(charset))]
	}
	return string(b)
}

```

And here are the results:
|Mode|IO/s|Time Elapsed|
|---|---|---|
|Normal|56.54|17.68s|
|WAL|17377.66|57.54ms|

Yeah I know

---


## Conclusion:
If you run a non fault tolerant app then continue using the default otherwise WAL is a no brainer for most people.
