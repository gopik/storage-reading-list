## WAL Protocol
The WAL protocol asserts that the log records representing changes to some data must already be on stable storage before the changeed data is allowed to replace the previous version of that data on the nonvolatile storage.

To enforce this, system using WAL method of recovery, store in every page the LSN of the log record that describes the most recent update performed on that page.

**Forward Processing** - Normal operation of the system where database is being updated due to client transactions.

**Partial Rollback** - Ability to set up savepoints during the execution of a transaction and later in the transaction, request the rolling back of the changes performed by the transaction since the establishment of a previous savepoint.

**Total rollback** - All updates of the transaction are undone (aborting the transaction)

**Nested Rollbacks** - Partial rollback followed by a total rollback or partial rollback followed by another partial rollback whose point of termination is earlier than the point of termination of previous rollback.

**Normal Undo** - Rollbacks executed during normal processing (transaction request to rollback or system initiated due to deadlocks or errors)

**Restart Undo** - Transaction rollback during restart recovery after system failure.

**Compensation Log Records (CLR)** - In many WAL based systems, the updated performed during a rollback are logged using what are called CLRs. Whether a CLRs update is undone, should that CLR be encountered during a rollback depends on the system. In ARIS, CLR updates are never undone, hence they are viewed as redo only log records.

**Page Oriented Redo(Undo)** - Log record whose update is redone describes which page of the database was originally modified during normal processing and if the same page is modified urding the redo process. No other descriptors/index need to be accessed to redo (or undo). This provides recovery independence amongst the objects.

**Logical redo(undo)** - In these systems, index changes are not logged separately but are redone using the log records for the data pages. Performing a redo requires accessing several descriptors and pages. The index tree need to be retraversed to determine the pages to be modified. Logical undo/redo allows uncommitted updates from one transaction to be moved to a different page, with physical redo/undo the 2nd transaction would have to wait for the previous transaction to commit, hence logical redo/undo increases concurrency.

**Latches vs Locks** - Latches are taken to maintain physical consistency whereas locks are for logical consistency. Logical consistency means application defined consistency. Physical consistency is to make consistent updates to the pages in a multi threaded execution. Latches are held for a small duration where as locks are held for entire transaction.

- Lock granularities
    - Shared
    - Exclusive
    - Intention exclusive
    - Intention shared
    - Shared intention exclusive