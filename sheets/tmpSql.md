Ah, das ändert die Diagnose deutlich. Wenn du **keinen Zugriff auf den Spring-Code** hast, kannst du die Ursache trotzdem ziemlich gut eingrenzen – direkt auf PostgreSQL-Seite.

Der wichtigste Punkt ist:

> Wenn derselbe `COUNT` in deiner PostgreSQL-Session ~1 Sekunde braucht, bedeutet das noch nicht automatisch, dass die Spring-App den gleichen Query nur 1 Sekunde lang ausführt.

### 1. Schau während der langsamen Anfrage in `pg_stat_activity`

Starte das:

```sql
SELECT
    pid,
    application_name,
    client_addr,
    state,
    wait_event_type,
    wait_event,
    now() - query_start AS duration,
    left(query, 500) AS query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY query_start;
```

Dann löst du den langsamen Request aus.

Interessant ist insbesondere:

```text
wait_event_type
wait_event
duration
```

Wenn du beispielsweise siehst:

```text
wait_event_type | Lock
```

wartet die App auf einen Lock.

Wenn du siehst:

```text
wait_event_type | Client
```

kann es etwas anderes sein – beispielsweise dass PostgreSQL bereits fertig ist bzw. auf den Client wartet.

### 2. Prüfe, ob die App überhaupt denselben SQL ausführt

In `pg_stat_activity` solltest du während des Requests das SQL der Spring-App sehen.

Wenn möglich, vergleiche:

```sql
SELECT count(*)
FROM ...
WHERE ...;
```

mit dem Query, den die App tatsächlich ausführt.

Gerade bei Hibernate/JPA kann das SQL erheblich komplexer sein als das, was man zunächst erwartet.

---

### 3. PostgreSQL kann die Laufzeit pro Query protokollieren

Falls du Zugriff auf die PostgreSQL-Konfiguration hast, ist `log_min_duration_statement` sehr hilfreich.

Zum Beispiel temporär:

```sql
ALTER SYSTEM SET log_min_duration_statement = 1000;
```

Danach:

```sql
SELECT pg_reload_conf();
```

Damit werden Statements geloggt, die länger als 1 Sekunde laufen.

Dann Request ausführen und im PostgreSQL-Log schauen.

Wenn dort steht:

```text
duration: 1023 ms
statement: SELECT count(*) ...
```

aber dein HTTP-Request dauert 20 Sekunden, dann ist ziemlich eindeutig:

**PostgreSQL braucht ~1 Sekunde, die restlichen ~19 Sekunden passieren außerhalb der DB.**

Wenn PostgreSQL dagegen loggt:

```text
duration: 19872 ms
```

dann ist es tatsächlich ein DB-seitiges Problem.

---

### 4. Besonders interessant: `application_name`

Wenn die Spring-App eine eigene `application_name` setzt, kannst du sie direkt identifizieren:

```sql
SELECT
    application_name,
    count(*)
FROM pg_stat_activity
GROUP BY application_name;
```

Dann beispielsweise:

```text
application_name | count
-----------------+------
my-spring-app    | 12
psql             | 1
```

Du kannst dann gezielt schauen:

```sql
SELECT
    pid,
    state,
    wait_event_type,
    wait_event,
    now() - query_start AS duration,
    query
FROM pg_stat_activity
WHERE application_name = 'my-spring-app';
```

---

### Mein Verdacht

Bei deinem beschriebenen Verhalten:

**PostgreSQL direkt: ~1 s**
**Spring HTTP Request: ~20 s**

würde ich aktuell **nicht davon ausgehen, dass dein Index das Problem ist**.

Es könnte beispielsweise sein:

```text
HTTP Request
    │
    ├── Connection aus Pool bekommen       15 s
    │
    ├── COUNT in PostgreSQL                 1 s
    │
    └── Rest                                 4 s
```

oder:

```text
HTTP Request
    │
    ├── andere DB-Abfragen                   8 s
    ├── COUNT                                1 s
    ├── weitere Verarbeitung                10 s
    └── Response                             1 s
```

Da du keinen App-Zugriff hast, ist **`pg_stat_activity` während eines tatsächlich laufenden 20-Sekunden-Requests** wahrscheinlich der beste nächste Schritt.

Wenn du möchtest, kann ich dir auch ein **einziges SQL-Skript geben, das du während des Requests laufen lässt und das dir möglichst eindeutig zeigt, ob die Spring-App gerade auf Lock, Connection/DB, I/O oder tatsächlich auf dem `COUNT` hängt.**
