# Sqlx cheatsheet

Below is a list of useful examples for using the sqlx crate. Although it is an incredible library their docs are a bit difficult to navigate (and could do with some more examples).

The type safety enforced by the library, although a massive plus, makes a few thing slightly unintuitive (e.g. using custom enums, dealing with nullable columns). I banged my head on these for awhile, re-reading the docs over and over. The examples below will hopefully help you avoid this.

<br>

# Using Enums

## Postgres Enum Type

Note the usage of:

- `type_name` to match the postgres field name
- non-macro `query`

```
CREATE TYPE mood AS ENUM ('sad', 'ok', 'happy');
CREATE TABLE IF NOT EXISTS demo (current_mood mood, template TEXT NOT NULL);
```

```
#[derive(sqlx::Type)]
#[sqlx(type_name = "mood", rename_all = "lowercase")]
enum Mood {
    Sad,
    Ok,
    Happy,
}

// Insert
sqlx::query("INSERT INTO test (current_mood, template) VALUES ($1, $2)")
    .bind(Mood::Sad)
    .bind("asdf")
    .execute(&mut conn)
    .await?;


// Select
let moods: Vec<Mood> = sqlx::query("SELECT current_mood from demo")
    .map(|f: PgRow| f.get("current_mood"))
    .fetch_all(&mut conn)
    .await?;
```

<br>

## `repr` and storing it as an integer

- The benefit here is you get compile time support and don't have to redefine your enum at the schema level

```
CREATE TABLE IF NOT EXISTS test (mood SMALLINT, template TEXT NOT NULL);
```

```
#[repr(i16)]
#[derive(sqlx::Type, FromPrimitive)]
enum Mood {
    #[default]
    Sad = 0,
    Ok = 1,
    Happy = 2,
}


// Insert
sqlx::query!(
    "INSERT INTO test (mood, template) VALUES ($1, $2)",
    Mood::Sad as i16,
    "asdf"
)
.execute(&mut conn)
.await?;


// Select
let moods: Vec<Mood> = sqlx::query!("select mood from test")
    .map(|f| Mood::from(f.mood))
    .fetch_all(&mut conn)
    .await?;
```

<br>

## Getting a single column

### Using a postgres primitive type

```
let template: String = sqlx::query!("select template from demo")
    .map(|f| f.template)
    .fetch_one(&mut conn)
    .await?;


let templates: String = sqlx::query!("select template from demo")
    .map(|f| f.template)
    .fetch_all(&mut conn)
    .await?;
```

# Handling nullable postgres fields
