**production-grade набор решений** для MS SQL Server (2017+), покрывающий все три сценария. Надёжность, устойчивость к грязным данным, и необходимость работать в условиях **ограниченных ресурсов и высокой ответственности** (миграции, дедупликация, инкрементальная загрузка).

---

## 🧩 1. **Динамический хеш любой таблицы**  
*(надёжный, через `INFORMATION_SCHEMA`, без ручного перечисления колонок)*

```sql
CREATE OR ALTER PROCEDURE dbo.GetTableHash
    @SchemaName SYSNAME = 'dbo',
    @TableName SYSNAME,
    @OrderByColumn SYSNAME = NULL,  -- если не задан — используем PK
    @TableHash VARBINARY(32) OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @sql NVARCHAR(MAX);
    DECLARE @columns NVARCHAR(MAX);
    DECLARE @orderCol SYSNAME;

    -- Определяем колонку для ORDER BY: либо задана, либо первичный ключ
    IF @OrderByColumn IS NULL
    BEGIN
        SELECT TOP 1 @orderCol = c.COLUMN_NAME
        FROM INFORMATION_SCHEMA.TABLE_CONSTRAINTS tc
        JOIN INFORMATION_SCHEMA.CONSTRAINT_COLUMN_USAGE ccu ON tc.CONSTRAINT_NAME = ccu.CONSTRAINT_NAME
        JOIN INFORMATION_SCHEMA.COLUMNS c ON ccu.TABLE_NAME = c.TABLE_NAME AND ccu.COLUMN_NAME = c.COLUMN_NAME
        WHERE tc.CONSTRAINT_TYPE = 'PRIMARY KEY'
          AND tc.TABLE_SCHEMA = @SchemaName
          AND tc.TABLE_NAME = @TableName
        ORDER BY c.ORDINAL_POSITION;
        
        IF @orderCol IS NULL
            THROW 50000, 'No primary key and no @OrderByColumn provided. Cannot ensure deterministic order.', 1;
    END
    ELSE
        SET @orderCol = QUOTENAME(@OrderByColumn);

    -- Формируем безопасную конкатенацию ВСЕХ колонок
    SELECT @columns = STRING_AGG(
        'ISNULL(' +
            CASE 
                WHEN DATA_TYPE IN ('bit') THEN 
                    'CAST(CASE WHEN ' + QUOTENAME(COLUMN_NAME) + ' = 1 THEN ''1'' ELSE ''0'' END AS VARCHAR(1))'
                WHEN DATA_TYPE LIKE '%char' OR DATA_TYPE LIKE '%text' OR DATA_TYPE IN ('xml') THEN
                    QUOTENAME(COLUMN_NAME)
                WHEN DATA_TYPE IN ('datetime', 'datetime2', 'smalldatetime', 'date', 'time') THEN
                    'FORMAT(' + QUOTENAME(COLUMN_NAME) + ', ''yyyy-MM-ddTHH:mm:ss.fffffff'')'
                ELSE
                    'CAST(' + QUOTENAME(COLUMN_NAME) + ' AS VARCHAR(500))'
            END +
        ', '''')',
        ' + CHAR(31) + '
    )
    FROM INFORMATION_SCHEMA.COLUMNS
    WHERE TABLE_SCHEMA = @SchemaName
      AND TABLE_NAME = @TableName
      AND COLUMN_NAME NOT IN ('rowversion', 'timestamp')  -- исключаем системные колонки
    ORDER BY ORDINAL_POSITION;

    IF @columns IS NULL
        THROW 50000, 'No columns found for hashing.', 1;

    SET @sql = N'
        SELECT @hash = HASHBYTES(''SHA2_256'',
            STRING_AGG(
                CONVERT(VARCHAR(MAX), row_hash, 2),
                ''''
            ) WITHIN GROUP (ORDER BY ' + @orderCol + N')
        )
        FROM (
            SELECT ' + @orderCol + N',
                HASHBYTES(''SHA2_256'', ' + @columns + N') AS row_hash
            FROM ' + QUOTENAME(@SchemaName) + N'.' + QUOTENAME(@TableName) + N'
        ) t;
    ';

    EXEC sp_executesql @sql, N'@hash VARBINARY(32) OUTPUT', @hash = @TableHash OUTPUT;
END;
```

> ✅ **Использование**:  
> ```sql
> DECLARE @h VARBINARY(32);
> EXEC dbo.GetTableHash 'dbo', 'customers', @TableHash = @h OUTPUT;
> SELECT CONVERT(VARCHAR(64), @h, 2) AS hash_hex;
> ```

> ⚠️ **Ограничения**:  
> - Не поддерживает `VARBINARY(MAX)`/`IMAGE` (нужно хешировать отдельно, если есть).  
> - Колонки `rowversion`/`timestamp` игнорируются — они всегда разные.  
> - Требуется PK или явный `@OrderByColumn`.

---

## 🔍 2. **Сравнение двух таблиц: diff-хеш и список расхождений**

```sql
CREATE OR ALTER PROCEDURE dbo.CompareTables
    @Schema1 SYSNAME = 'dbo',
    @Table1 SYSNAME,
    @Schema2 SYSNAME = 'dbo',
    @Table2 SYSNAME,
    @KeyColumn SYSNAME  -- должен существовать в обеих таблицах
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @sql NVARCHAR(MAX);

    -- Получаем список колонок, общих для обеих таблиц (по имени)
    DECLARE @cols NVARCHAR(MAX);
    SELECT @cols = STRING_AGG(QUOTENAME(c1.COLUMN_NAME), ', ')
    FROM INFORMATION_SCHEMA.COLUMNS c1
    JOIN INFORMATION_SCHEMA.COLUMNS c2 
        ON c1.COLUMN_NAME = c2.COLUMN_NAME
    WHERE c1.TABLE_SCHEMA = @Schema1 AND c1.TABLE_NAME = @Table1
      AND c2.TABLE_SCHEMA = @Schema2 AND c2.TABLE_NAME = @Table2
      AND c1.COLUMN_NAME != 'rowversion';

    IF @cols IS NULL
        THROW 50000, 'No common columns found.', 1;

    -- Хешируем строки в обеих таблицах
    SET @sql = N'
    WITH
    t1 AS (
        SELECT ' + QUOTENAME(@KeyColumn) + N' AS key_val,
               HASHBYTES(''SHA2_256'', 
                   ' + REPLACE(@cols, ', ', ' + CHAR(31) + ') + N'
               ) AS row_hash
        FROM ' + QUOTENAME(@Schema1) + N'.' + QUOTENAME(@Table1) + N'
    ),
    t2 AS (
        SELECT ' + QUOTENAME(@KeyColumn) + N' AS key_val,
               HASHBYTES(''SHA2_256'', 
                   ' + REPLACE(@cols, ', ', ' + CHAR(31) + ') + N'
               ) AS row_hash
        FROM ' + QUOTENAME(@Schema2) + N'.' + QUOTENAME(@Table2) + N'
    )
    SELECT 
        COALESCE(t1.key_val, t2.key_val) AS key_val,
        CASE 
            WHEN t1.key_val IS NULL THEN ''MISSING_IN_T1''
            WHEN t2.key_val IS NULL THEN ''MISSING_IN_T2''
            WHEN t1.row_hash != t2.row_hash THEN ''CONTENT_DIFF''
            ELSE ''MATCH''
        END AS status
    FROM t1
    FULL OUTER JOIN t2 ON t1.key_val = t2.key_val
    WHERE t1.row_hash != t2.row_hash 
       OR t1.key_val IS NULL 
       OR t2.key_val IS NULL;
    ';

    EXEC sp_executesql @sql;
END;
```

> ✅ **Использование**:  
> ```sql
> EXEC dbo.CompareTables 'staging', 'customers_new', 'dbo', 'customers', 'id';
> ```
> Вернёт:  
> - `MISSING_IN_T1` — строка есть только в T2  
> - `MISSING_IN_T2` — строка есть только в T1  
> - `CONTENT_DIFF` — ключ есть, но данные отличаются  

> 💡 Для **полного diff по данным** (без ключа) — используй row_hash + полное внешнее соединение по хешу, но это дорого.

---

## 🔄 3. **Incremental load с хешированием только новых/изменённых строк**

> Предположение: в таблице есть `updated_at DATETIME2` или `created_at`.

```sql
-- Шаблон для ETL-скрипта (не процедура, а встраиваемый блок)
DECLARE @last_run DATETIME2 = '2025-01-01T00:00:00'; -- или из таблицы метаданных

WITH new_or_changed AS (
    SELECT 
        id,
        HASHBYTES('SHA2_256',
            ISNULL(CAST(id AS VARCHAR(50)), '') + CHAR(31) +
            ISNULL(name, '') + CHAR(31) +
            ISNULL(email, '') + CHAR(31) +
            ISNULL(FORMAT(created_at, 'yyyy-MM-ddTHH:mm:ss.fffffff'), '') + CHAR(31) +
            ISNULL(CAST(CASE WHEN is_active = 1 THEN '1' ELSE '0' END AS VARCHAR(1)), '')
        ) AS row_hash
    FROM customers
    WHERE updated_at > @last_run   -- или COALESCE(updated_at, created_at) > @last_run
)
-- Вставляем/обновляем в целевую таблицу только changed
MERGE target_customers AS tgt
USING new_or_changed AS src ON tgt.id = src.id
WHEN MATCHED AND tgt.row_hash != src.row_hash THEN
    UPDATE SET 
        name = /* ... */, 
        email = /* ... */,
        row_hash = src.row_hash,
        updated_at = SYSDATETIME()
WHEN NOT MATCHED THEN
    INSERT (id, name, email, created_at, is_active, row_hash)
    VALUES (/* ... */, src.row_hash);
```

> ✅ **Ключевые практики**:
> - Храни `row_hash` в целевой таблице — это экономит 90% CPU на последующих сравнениях.
> - Используй `MERGE` только если понимаешь его блокировки. Иначе — `UPDATE` + `INSERT NOT EXISTS`.
> - Для очень больших таблиц — разбивай по партициям (`WHERE id BETWEEN ...`).

---

## 🔥 Заключение: 

1. **Никогда не хешируй "на глаз"** — используй проверенные шаблоны выше.
2. **Всегда храни `row_hash` в staging-таблицах** — это твой щит от багов.
3. **Тестируй на грязных данных** — особенно с `|`, `\n`, `NULL`, unicode.
4. **Автоматизируй через процедуры** — меньше ручного кода = меньше ошибок.

 дать:
- Скрипт для генерации `row_hash` колонки в любой таблице
- Шаблон для Airflow/SSIS с вызовом этих процедур
- Unit-тесты на T-SQL (tSQLt)
