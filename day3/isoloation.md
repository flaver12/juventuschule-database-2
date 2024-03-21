### 1. Abfrage des aktuellen Isolationslevels einer Transaktion

Um den aktuellen Isolationslevel einer Transaktion in SQL abzufragen, können Sie die systemeigene Funktion oder Abfrage verwenden, die von Ihrem SQL-Datenbankmanagementsystem (DBMS) bereitgestellt wird. Für Microsoft SQL Server sieht die Abfrage wie folgt aus:

```sql
SELECT CASE transaction_isolation_level
       WHEN 0 THEN 'Unspecified'
       WHEN 1 THEN 'ReadUncommitted'
       WHEN 2 THEN 'ReadCommitted'
       WHEN 3 THEN 'RepeatableRead'
       WHEN 4 THEN 'Serializable'
       WHEN 5 THEN 'Snapshot'
       END AS IsolationLevel
FROM sys.dm_exec_sessions
WHERE session_id = @@SPID;
```

Diese Abfrage gibt den Isolationslevel der aktuellen Sitzung zurück, indem sie die `sys.dm_exec_sessions`-Dynamikverwaltungssicht verwendet und nach der aktuellen `session_id` (`@@SPID`) sucht.

### 2. Setzen der Isolationslevels in SQL

Isolationslevel in SQL können in der Regel mit dem `SET TRANSACTION ISOLATION LEVEL`-Befehl gesetzt werden. Hier sind die Befehle für die verschiedenen Isolationslevels:

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
```

Diese Befehle setzen den Isolationslevel für die Transaktion oder die Sitzung, abhängig von den spezifischen Regeln des verwendeten DBMS.

### 3. Standard-Isolationslevel von MS-SQL

Der Standard-Isolationslevel in Microsoft SQL Server ist `READ COMMITTED`. Dieser Level erlaubt keine sogenannten "Dirty Reads" (das Lesen von nicht bestätigten Änderungen), kann aber dennoch Phänomene wie "Non-Repeatable Reads" oder "Phantom Reads" zulassen.

### 4. Unterschiedliches Verhalten bei Datenmutationen aufzeigen

Das Verhalten von Transaktionen unter verschiedenen Isolationslevels kann durch das Auftreten oder die Vermeidung von bestimmten Phänomenen wie Dirty Reads, Non-Repeatable Reads und Phantom Reads gezeigt werden. Hier ein einfaches Beispiel:

- **READ UNCOMMITTED**: Erlaubt Dirty Reads. Eine Transaktion kann Änderungen sehen, die noch nicht festgeschrieben wurden. Dies kann zu Inkonsistenzen führen, wenn diese Änderungen später zurückgerollt werden.

- **READ COMMITTED**: Verhindert Dirty Reads, indem nur festgeschriebene Daten gelesen werden. Allerdings können Daten zwischen zwei Lesevorgängen innerhalb derselben Transaktion von anderen Transaktionen geändert werden, was zu Non-Repeatable Reads führt.

- **REPEATABLE READ**: Verhindert Dirty Reads und Non-Repeatable Reads, indem sichergestellt wird, dass alle gelesenen Daten während der gesamten Transaktion konstant bleiben. Jedoch können neue Datensätze, die den Suchkriterien entsprechen, von anderen Transaktionen hinzugefügt werden (Phantom Reads).

- **SERIALIZABLE**: Das strengste Level, das Dirty Reads, Non-Repeatable Reads und Phantom Reads verhindert. Es stellt sicher, dass die Transaktionen so ausgeführt werden, als ob sie nacheinander ablaufen würden, was jedoch die Parallelität und Leistung beeinträchtigen kann.

### Praktische Beispiele

#### 1. READ UNCOMMITTED

In diesem Level können Transaktionen Änderungen lesen, die noch nicht festgeschrieben wurden. Dies kann zu "Dirty Reads" führen. Angenommen, Transaktion A ändert einen Datensatz, hat aber noch kein Commit durchgeführt. Transaktion B kann diese Änderungen lesen, auch wenn sie möglicherweise nie bestätigt werden.

```sql
-- Transaktion A
BEGIN TRANSACTION;
UPDATE Accounts SET Balance = Balance - 100 WHERE AccountId = 1;

-- Transaktion B
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
BEGIN TRANSACTION;
SELECT Balance FROM Accounts WHERE AccountId = 1; -- Kann die unbestätigten Änderungen von Transaktion A sehen
COMMIT;
```

#### 2. READ COMMITTED

Dies ist der Standard-Isolationslevel in MS SQL Server. "Dirty Reads" sind nicht möglich, aber "Non-Repeatable Reads" können auftreten. Wenn Transaktion A Daten liest, Transaktion B diese Daten ändert und Transaktion A die Daten erneut liest, können sich die Daten zwischen den beiden Lesevorgängen unterscheiden.

```sql
-- Transaktion A
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN TRANSACTION;
SELECT Balance FROM Accounts WHERE AccountId = 1; -- Erster Lesevorgang

-- Transaktion B führt Änderungen durch und führt ein Commit aus
BEGIN TRANSACTION;
UPDATE Accounts SET Balance = Balance - 100 WHERE AccountId = 1;
COMMIT;

-- Transaktion A liest erneut
SELECT Balance FROM Accounts WHERE AccountId = 1; -- Zweiter Lesevorgang, möglicherweise unterschiedlich vom ersten
COMMIT;
```

#### 3. REPEATABLE READ

Dieser Level verhindert "Dirty Reads" und "Non-Repeatable Reads". Die in der Transaktion gelesenen Daten bleiben während der gesamten Transaktion konstant. Allerdings können "Phantom Reads" auftreten.

```sql
-- Transaktion A
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN TRANSACTION;
SELECT COUNT(*) FROM Accounts; -- Zählt die Anzahl der Konten

-- Transaktion B fügt ein neues Konto hinzu und führt ein Commit aus
BEGIN TRANSACTION;
INSERT INTO Accounts(AccountId, Balance) VALUES (3, 500);
COMMIT;

-- Transaktion A zählt erneut
SELECT COUNT(*) FROM Accounts; -- Ergebnis bleibt gleich, trotz neuer Daten durch Transaktion B
COMMIT;
```

#### 4. SERIALIZABLE

Dies ist das strengste Level, das "Dirty Reads", "Non-Repeatable Reads" und "Phantom Reads" verhindert. Es kann jedoch zu einer erhöhten Sperre und reduzierten Parallelität führen.

```sql
-- Transaktion A
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN TRANSACTION;
SELECT COUNT(*) FROM Accounts; -- Sperren werden angewendet, um Datenkonsistenz zu gewährleisten

-- Transaktion B versucht, ein neues Konto hinzuzufügen, wird aber blockiert, bis Transaktion A beendet ist
BEGIN TRANSACTION;
INSERT INTO Accounts(AccountId, Balance) VALUES (3, 500);
COMMIT;

-- Transaktion A zählt erneut und führt dann ein Commit aus
SELECT COUNT(*) FROM Accounts;
COMMIT;
```

#### 5. SNAPSHOT

Snapshot-Isolation bietet eine Versionierung der Daten, die es Transaktionen ermöglicht, Daten so zu sehen, wie sie zu Beginn der Transaktion waren, unabhängig von späteren Änderungen durch andere Transaktionen. Dies verhindert "Dirty Reads", "Non-Repeatable Reads" und "Phantom Reads", ohne dass Sperren erforderlich sind.

Um die Snapshot-Isolation zu verwenden, muss sie auf der Datenbankebene aktiviert werden:

```sql
ALTER DATABASE MyDatabase
SET ALLOW_SNAPSHOT_ISOLATION ON;
```

Dann kann eine Transaktion im Snapshot-Isolationslevel ausgeführt
