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
- If WAL segments grow beyond max_wal_size, PostgreSQL initiates a forced checkpoint to clean up WAL files. and its default size is 1GB.
- PostgreSQL automatically removes WAL (Write-Ahead Logging) files from the pg_wal directory when they are no longer needed.
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
      postgresql waits until the data is fully written and Acknowladgement to disk before allowing nxt process. (ensure data safty but may slow down performance. it's need to waits for Acknowladgement.)
      • Cons: Can slow down performance due to the overhead of frequent disk writes.
    ```
- async off:
      ```
      Does not wait for confirmation tha the data is written to disk.
  	(It moves on to the next perocess immediately, which improves speed but risk losing recent tramsaction if the system crash).
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

- full_page_writes = on :-
  ```
   when page is modified the first time aftera checkpoint, postgresql writes the entire page in wal file. after the first modification, only row level changes are 
    logged until the next checkpoint. (in case of cresh, can restore full p[age from wal).

   -Recovery Process:
        ◦ Upon recovery, PostgreSQL scans the WAL for the logged pages.
        ◦ If the full page was logged, PostgreSQL restores the complete page from the WAL.
        ◦ If not, the system may end up with a corrupted page.
  ```
- full_page_writes = off:
  ```
  May improve performance but carries a risk of data loss or corruption in the event of a crash.
  ```
  ```
- wal_buffers
  ```
      • Defines the amount of shared memory allocated to WAL data before it’s written to disk.
      • Default: -1 automatically calculates based on the shared_buffers setting.
      • Increasing this may improve performance in write-heavy environments.
  ```
#### Commit Parameters
```
    1. commit_delay
        ◦ Description: This setting specifies the amount of time (in microseconds) that PostgreSQL will wait before flushing committed transactions to disk. It can be set to a value between 0 and 100000 microseconds (0 to 100 milliseconds).
    2. commit_siblings
        ◦ Description: This parameter defines the number of concurrent connections required to trigger the commit_delay. The default value is 5, meaning that if there are at least 5 concurrent transactions, the database will use the commit_delay.
            ▪ If the number of active transactions exceeds this threshold, PostgreSQL will apply the commit_delay for new transactions, potentially allowing them to group their writes, reducing the number of disk operations.
```
