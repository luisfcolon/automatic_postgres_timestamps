# Automatic postgres timestamps

Common timestamps for most postgres database tables are:

* created_at
* updated_at

It's easy to default `created_at` to `NOW()` when inserting a new record.

Manually setting `updated_at` every time a row is modified is a tedious meh and often times forgotten task.

Below is an example that will trigger a function to automatically set the `updated_at` column to `NOW()` for every row updated on a table during an update SQL query.

## Setting up the SQL

### Table schema

```
CREATE TABLE IF NOT EXISTS users (
  id           UUID PRIMARY KEY NOT NULL,
  firstname    TEXT NOT NULL,
  lastname     TEXT NOT NULL,
  created_at   TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  updated_at   TIMESTAMP WITH TIME ZONE
);
```

### Creating the function

This function will check to see if `updated_at` is part of the current query. If not it will set it to `NOW()`. 

```
CREATE OR REPLACE FUNCTION set_updated_at_to_now()
RETURNS TRIGGER AS $$
BEGIN
  IF to_jsonb(NEW) ? 'updated_at' THEN
    NEW.updated_at = NOW();
  END IF;

  RETURN NEW;
END;
$$ language 'plpgsql';
```

On it's own this function does nothing. We need to associate it to a table and table action (create, update, delete, etc).

### Creating the trigger

This trigger will call our new function `set_updated_at_to_now()` on the `users` table before any `UPDATE` SQL query is executed. 

```
CREATE TRIGGER trigger_users_set_updated_at
  BEFORE UPDATE ON users
  FOR EACH ROW
  EXECUTE PROCEDURE set_updated_at_to_now();
```

We no longer have to worry about setting this timestamp when updating any database table this trigger is assigned to.

#### We can add this function to as many tables as we want

```
CREATE TABLE IF NOT EXISTS accounts (
  id           UUID PRIMARY KEY NOT NULL,
  name         TEXT NOT NULL,
  type         TEXT NOT NULL,
  created_at   TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  updated_at   TIMESTAMP WITH TIME ZONE
);
```

```
CREATE TRIGGER trigger_accounts_set_updated_at
  BEFORE UPDATE ON accounts
  FOR EACH ROW
  EXECUTE PROCEDURE set_updated_at_to_now();
```

## See it in action

### Adding a new user

```
INSERT INTO users 
VALUES(
  '425db3cc-c587-4076-8c97-1072df7a7906',
  'John',
  'Smith'
);
```

Now we have an entry in our `users` table. `updated_at` is empty.

```
                  id                  | firstname | lastname |          created_at           | updated_at 
--------------------------------------+-----------+----------+-------------------------------+------------
 425db3cc-c587-4076-8c97-1072df7a7906 | John      | Smith    | 2019-04-19 21:06:53.457559-04 | 
```

### Updating the user we created

Let's change John's last name to Doe.

```
UPDATE users
SET lastname = 'Doe'
WHERE id = '425db3cc-c587-4076-8c97-1072df7a7906';
```

Let's see how the table looks now:

```
                  id                  | firstname | lastname |          created_at           |          updated_at           
--------------------------------------+-----------+----------+-------------------------------+-------------------------------
 425db3cc-c587-4076-8c97-1072df7a7906 | John      | Doe      | 2019-04-19 21:06:53.457559-04 | 2019-04-19 21:12:40.926747-04
(1 row)
```

Look at that! The `updated_at` field was updated and we didn't have to set it ourselves. Nice.

## Seeing it all together

```
CREATE TABLE IF NOT EXISTS users (
  id           UUID PRIMARY KEY NOT NULL,
  firstname    TEXT NOT NULL,
  lastname     TEXT NOT NULL,
  created_at   TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  updated_at   TIMESTAMP WITH TIME ZONE
);

CREATE TABLE IF NOT EXISTS accounts (
  id           UUID PRIMARY KEY NOT NULL,
  name         TEXT NOT NULL,
  type         TEXT NOT NULL,
  created_at   TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  updated_at   TIMESTAMP WITH TIME ZONE
);

CREATE OR REPLACE FUNCTION set_updated_at_to_now()
RETURNS TRIGGER AS $$
BEGIN
  IF to_jsonb(NEW) ? 'updated_at' THEN
    NEW.updated_at = NOW();
  END IF;

  RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER trigger_users_set_updated_at
  BEFORE UPDATE ON users
  FOR EACH ROW
  EXECUTE PROCEDURE set_updated_at_to_now();
  
CREATE TRIGGER trigger_accounts_set_updated_at
  BEFORE UPDATE ON accounts
  FOR EACH ROW
  EXECUTE PROCEDURE set_updated_at_to_now();
```

# Fin