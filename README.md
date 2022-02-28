# mssql-django Index Issue

This project was created to reproduce a problem with indexes created by mssql-django version 1.1.2.

Specifically, the mssql-django library is not correctly removing an index before modifying a column upon which the index 
depends.

Steps to reproduce:

1. Run migrations.
```commandline
migrate
```

2. Rollback migrations.
```commandline
migrate wagtailcore 0056
```

Error:
```
  Unapplying wagtailcore.0057_page_locale_fields_notnull...Traceback (most recent call last):
  File "/Users/rjschave/PycharmProjects/wagtail_mssql_django/venv/lib/python3.8/site-packages/django/db/backends/utils.py", line 84, in _execute
    return self.cursor.execute(sql, params)
  File "/Users/rjschave/PycharmProjects/wagtail_mssql_django/venv/lib/python3.8/site-packages/mssql/base.py", line 576, in execute
    return self.cursor.execute(sql, params)
pyodbc.ProgrammingError: ('42000', "[42000] [Microsoft][ODBC Driver 17 for SQL Server][SQL Server]The index 'wagtailcore_page_translation_key_locale_id_9b041bad_uniq' is dependent on column 'translation_key'. (5074) (SQLExecDirectW)")
```


SQL from wagtailcore 0057 forward migration
```sql
BEGIN TRANSACTION
--
-- Alter field locale on page
--
ALTER TABLE [wagtailcore_page] DROP CONSTRAINT [wagtailcore_page_locale_id_3c7e30a6_fk_wagtailcore_locale_id];
DROP INDEX [wagtailcore_page_locale_id_3c7e30a6] ON [wagtailcore_page];
DROP INDEX [wagtailcore_page_translation_key_locale_id_9b041bad_uniq] ON [wagtailcore_page];
ALTER TABLE [wagtailcore_page] ALTER COLUMN [locale_id] int NOT NULL;
CREATE INDEX [wagtailcore_page_locale_id_3c7e30a6] ON [wagtailcore_page] ([locale_id]);
CREATE UNIQUE INDEX [wagtailcore_page_translation_key_locale_id_9b041bad_uniq] ON [wagtailcore_page] ([translation_key], [locale_id]) WHERE [translation_key] IS NOT NULL AND [locale_id] IS NOT NULL;
ALTER TABLE [wagtailcore_page] ADD CONSTRAINT [wagtailcore_page_locale_id_3c7e30a6_fk_wagtailcore_locale_id] FOREIGN KEY ([locale_id]) REFERENCES [wagtailcore_locale] ([id]);
--
-- Alter field translation_key on page
--
DROP INDEX [wagtailcore_page_translation_key_locale_id_9b041bad_uniq] ON [wagtailcore_page];
ALTER TABLE [wagtailcore_page] ADD DEFAULT 'd48babde60dd4b11a643ba9a48a6d8ff' FOR [translation_key];
UPDATE [wagtailcore_page] SET [translation_key] = '08bd8dfe97bc457e8d59d7ee26ea24dd' WHERE [translation_key] IS NULL;
ALTER TABLE [wagtailcore_page] ALTER COLUMN [translation_key] char(32) NOT NULL;
CREATE UNIQUE INDEX [wagtailcore_page_translation_key_locale_id_9b041bad_uniq] ON [wagtailcore_page] ([translation_key], [locale_id]) WHERE [translation_key] IS NOT NULL AND [locale_id] IS NOT NULL;
SELECT d.name FROM sys.default_constraints d INNER JOIN sys.tables t ON d.parent_object_id = t.object_id INNER JOIN sys.columns c ON d.parent_object_id = c.object_id AND d.parent_column_id = c.column_id INNER JOIN sys.schemas s ON t.schema_id = s.schema_id WHERE t.name = 'wagtailcore_page' AND c.name = 'translation_key';
ALTER TABLE [wagtailcore_page] DROP CONSTRAINT [translation_key];
COMMIT;

```

SQL from wagtailcore 0057 backwards migration
```sql
BEGIN TRANSACTION
--
-- Alter field translation_key on page
--
ALTER TABLE [wagtailcore_page] ALTER COLUMN [translation_key] char(32) NULL;
--
-- Alter field locale on page
--
ALTER TABLE [wagtailcore_page] DROP CONSTRAINT [wagtailcore_page_locale_id_3c7e30a6_fk_wagtailcore_locale_id];
ALTER TABLE [wagtailcore_page] ALTER COLUMN [locale_id] int NULL;
ALTER TABLE [wagtailcore_page] ADD CONSTRAINT [wagtailcore_page_locale_id_3c7e30a6_fk_wagtailcore_locale_id] FOREIGN KEY ([locale_id]) REFERENCES [wagtailcore_locale] ([id]);
COMMIT;

```