# Write-Ahead-Log-WAL-WAL-Archiving
When the client performs some changes insert, update, and delete operations.
These changes are made in the shared buffer and wal buffer before it is written to disk.
Before changes are written to the actual database files wal_writter process writes these committed records from the wal_buffer to the pg_wal directory. 
Ensures that no data is lost during crashes or failures, if the database crashes, WAL allows recovery to that last consistent state.

- Wal segment file size is 16MB default and we can configure during cluster creation.( - - wal-segsize=32) and it is internally divided into pages.
- We can not change size of it after cluster creation.
- Each WAL segment file name is a 24-digit hexadecimal number.
    ```
		timelineId + (LSN - 1)
		-000000010000000000000001 to 0000000100000000000000FF
    ```
-------------------------------------------------------------------------------------------------------------------------------
-  Provide various configuration parameters to manage.

- wal_level: - Controls the amount of information written to WAL.
  ```
    *minimal: - logs only the minimal information needed for crash recovery Best for non -replication systems.
	            - Only the essential information is written to the WAL for crash recovery purposes. No support for point-in-time recovery (PITR), logical replication, or streaming replication.
    *replica:-  (most common for streaming replication):
              • Use Case: Required for physical (streaming) replication and PITR (Point-in-Time Recovery).
              • Data: Includes more information than minimal, which allows replicas to be kept in sync with the primary server.
    *logical: -
              • Use Case: Required for logical replication (e.g., using pglogical or built-in logical replication in PostgreSQL).
              • Data: Writes additional information that enables changes to individual tables or rows to be replicated between servers. Logical replication allows replication of a subset of the database and even replication between different PostgreSQL versions.
  ```
---------------------------------------------------------------------------------------------------------------------
- fsync
    - Ensures data is written to disk to prevent corruption in case of a crash.
    ```
      Using fsync on:
      • Pros: Guarantees data integrity, safe from crashes.
      • Cons: Can slow down performance due to the overhead of frequent disk writes.
      Using async off:
      • Pros: Improves performance by reducing disk I/O.
      • Cons: Risks data loss and corruption during crashes.
    ```
---------------------------------------------------------------------------------------------------------------------
- synchronous_commit
    Transaction Commit Process:
    - When a transaction is committed, PostgreSQL writes the transaction log (WAL) to disk.
    - Depending on the synchronous_commit setting, the behavior of this process changes.
 
```
        ◦ on: Ensures commits are fully synchronized.
        ◦ off: Can lead to better performance, but with the risk of losing transactions in a crash.
        ◦ local: Synchronizes only the local server, not the standby.
    • on:
        ◦ PostgreSQL waits for the WAL to be written to both the primary and the standby server before acknowledging the commit to the client.
        ◦ This guarantees that if the primary server crashes immediately after a commit, the changes will still be available on the standby server.
    • off:
        ◦ The primary server acknowledges the commit as soon as the WAL is written to its own disk.
        ◦ This may lead to better performance since the application does not wait for confirmation from the standby server.
        ◦ However, if the primary crashes before the standby server receives the WAL, the transaction can be lost.
    • local:
        ◦ The primary server acknowledges the commit only after the WAL is written to its own disk, similar to off.
        ◦ If the standby is not available or if it cannot be reached, the commit will still be acknowledged.
        ◦ This is useful for ensuring fast commits while still allowing for replication when the standby server is reachable.
```
---------------------------------------------------------------------------------------------------------------------
- wal_sync_method
```
  - The wal_sync_method parameter in PostgreSQL specifies how the Write-Ahead Logging (WAL) system performs synchronization operations for ensuring data durability.
  - This setting directly affects how WAL data is written to disk and the methods used to guarantee that the data is safely stored before PostgreSQL acknowledges a transaction as committed
```
- open_datasync:
  ```
    • Uses the open system call with the O_DSYNC flag.
    • Guarantees that data is written to disk before returning control to the application, but it may allow metadata (like file attributes) to be cached.
    • This method is typically faster than fsync because it does not require full synchronization of metadata, but it still ensures data integrity.
  ```
- fdatasync:
  ```
    • This is the default method on Linux systems.
    • It flushes the data to disk and ensures that any data modifications are saved, but it may not synchronize file metadata.
    • This method is efficient for databases, as it avoids unnecessary overhead related to metadata updates, making it a good balance between performance and durability.
  ```

- fsync:
  ```
    • Forces the system to write all changes to disk, including both data and metadata.
    • It ensures that the data is fully committed to the storage device before any write operation is acknowledged.
    • This method offers strong data integrity guarantees but may incur more overhead due to the increased number of disk I/O operations.
    - fsync_writethrough:
    • Similar to fsync, but with the O_SYNC flag, it writes data to disk and ensures that it is fully committed, including metadata.
    • The difference is that it may bypass the cache for write operations, enforcing that all writes are immediately visible to other processes.
    • This method provides strong durability guarantees but may also lead to performance degradation due to its synchronous nature.
  ```
- open_sync:
  ```
    • Uses the open system call with the O_SYNC flag.
    • It guarantees that all writes are flushed to disk immediately, including both data and metadata.
    • This method is very strict about data integrity and ensures that any modifications are fully committed before returning control to PostgreSQL, but it can significantly slow down performance.
  ```
-   Performance: Methods like fdatasync and open_datasync can improve performance by reducing the overhead of disk writes, making them suitable for high-throughput scenarios.
-   Durability: Methods like fsync and open_sync prioritize data integrity but can slow down write operations, as they require more comprehensive synchronization before acknowledging transactions.
- full_page_writes
  ```
    • Helps recover from partial page writes during a crash by ensuring full pages are written to WAL.
    • The full_page_writes setting in PostgreSQL is an important feature that ensures data integrity in the event of a crash.
     It specifically addresses the issue of partial page writes, which can occur when a transaction modifies only a portion of a data page.
    • Options:
        ◦ on: Protects against partial page corruption.
        ◦ off: May improve performance but risks incomplete data during a crash.
     When full_page_writes is enabled (set to on), PostgreSQL takes the following approach:
      1. Logging Full Pages: When a data page is modified (e.g., an update to a row), PostgreSQL writes the entire contents of the page to the Write-Ahead Log (WAL) before applying the changes to the actual data file. This is done regardless of whether the entire page has changed or just a portion of it.
      2. Partial Page Writes: If a crash occurs before the modified page is fully written to the data file, and only a portion of that page is written, the page in the data file may become corrupted. By logging the entire page in the WAL, PostgreSQL can reconstruct the page from the log if needed during recovery.
      3. Recovery Process: During recovery, if PostgreSQL detects a crash, it reads the WAL to ensure that the data is restored correctly. If only part of a page was written to the data file and a full page write was logged, PostgreSQL uses the full version from the WAL to restore data integrity.
  ```
- Workflow of full_page_writes
  ```
    1. Transaction Begins: A transaction begins and makes changes to one or more data pages in the database.
    2. Page Modification: The transaction modifies data on a specific page (for example, updating a few rows).
    3. WAL Logging:
        ◦ If full_page_writes is set to on, PostgreSQL writes the entire page to the WAL before writing any changes to the actual data file.
        ◦ If full_page_writes is set to off, only the changes (the specific row updates) are logged.
    4. Data File Update: The changes are then applied to the actual data file (data page).
    5. Checkpointing: Periodically, PostgreSQL performs a checkpoint, where it writes all modified pages to disk, ensuring data durability.
    6. Crash Occurs: If a crash occurs at any point after the data page is partially written but before it has been fully written, the following happens:
        ◦ If full_page_writes is on, the full version of the page from the WAL can be used to recover the data during startup.
        ◦ If full_page_writes is off, only the changes may be recorded in the WAL, leading to potential data loss or corruption if the data page is incomplete.
    7. Recovery Process:
        ◦ Upon recovery, PostgreSQL scans the WAL for the logged pages.
        ◦ If the full page was logged, PostgreSQL restores the complete page from the WAL.
        ◦ If not, the system may end up with a corrupted page.
Summary
    • full_page_writes = on: Ensures data integrity by logging full pages, protecting against partial page corruption.
    • full_page_writes = off: May improve performance but carries a risk of data loss or corruption in the event of a crash.
  ```
- wal_buffers
  ```
      • Defines the amount of shared memory allocated to WAL data before it’s written to disk.
      • Default: -1 automatically calculates based on the shared_buffers setting.
      • Increasing this may improve performance in write-heavy environments.
  ```
- Commit Parameters
```
    1. commit_delay
        ◦ Description: This setting specifies the amount of time (in microseconds) that PostgreSQL will wait before flushing committed transactions to disk. It can be set to a value between 0 and 100000 microseconds (0 to 100 milliseconds).
        ◦ Workflow:
            ▪ When a transaction is committed, if commit_delay is greater than 0, PostgreSQL waits for that duration to allow other transactions to commit as well. This can group multiple transactions into fewer writes, which can improve performance at the cost of durability in certain failure scenarios.
    2. commit_siblings
        ◦ Description: This parameter defines the number of concurrent connections required to trigger the commit_delay. The default value is 5, meaning that if there are at least 5 concurrent transactions, the database will use the commit_delay.
        ◦ Workflow:
            ▪ If the number of active transactions exceeds this threshold, PostgreSQL will apply the commit_delay for new transactions, potentially allowing them to group their writes, reducing the number of disk operations.
```
