---
status: accepted
---

# Run v1 as one Bun process with embedded SQLite

The first release will serve one Workspace per Installation from one Bun process and one local SQLite database, with WAL, short write transactions, and a single-writer gate. This minimizes operational burden for small teams and fits the Effect/Bun stack, at the cost of no horizontal write scaling, HA, point-in-time recovery, or network-filesystem deployment; PostgreSQL is a future migration only if measured requirements justify it.
