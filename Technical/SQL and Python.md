The shortcomings of database object-relational mappers (ORMs) are well-documented. Most memorable is [this](http://blogs.tedneward.com/post/the-vietnam-of-computer-science/) 2006 piece by Ted Neward comparing the ORM quagmire to the US involvement in Vietnam. Hyperbolic, maybe - but also pretty evocative and certinaly memorable. This was the piece that made me realise: an ORM is not the only - nor necessarily the best - way to wire up your arbitrary program code with your database.

In many cases, however, it may be the first place you end up. SQLAlchemy and Django ORM are monsters in the Python world, and depending on how you came to coding generally and to Python in particular, you may think of them as your only realistic options when trying to wire up a database. They aren't.

Of course, it would also be going too far to say that using an ORM is never a good idea. There are plenty of situations where it is an excellent idea. This post is not about those situations. This post is about situations where you explicitly do not want to use an ORM for whatever reason. Perhaps you think SQL is an excellent and capable language in and of itself. Perhaps you are constitutionally inclined to using a lightweight, thin API to talk to your database instead of a full-blown ORM. Perhaps you just really want to improve your SQL. For me, a combination of these concerns led me to wonder what life is like outside of ORM-land.

It is, of course, possible to talk directly to the database using a Python driver. Let's talk SQLite as an example. Let's start by doing some setup.

```python
import sqlite3

database='db/sqlite.db'
conn = sqlite3.connect(database)
```

Next, let's consider the following code to initialise the database from a separate `.sql` script file.

```python
cursor = conn.cursor()
with open('db/db_init.sql', 'r') as sqlite_file:
    sql_script = sqlite_file.read()
cursor.executescript(sql_script)
cursor.close()
```

The example of database initialisation is germane; while it might be a good approach to a one-off process such as initialisation, you don't want to call a script every time you want to run arbitrary queries against the database. To do that, you can treat SQL statements as ordinary Python strings.

```python
cursor = conn.cursor()
query = """
    SELECT DISTINCT
        "security_ticker"
    FROM
        "securities";
"""
cursor.execute(query).fetchall()
cursor.close()
```

That looks a bit hacky, doesn't it? It works great, though, and keeps the SQL query alongside your own code. If you're using Pycharm, you'll even get SQL highlighting and code completion, which is extremely handy.

You absolutely can write all of your queries like this if you like - I know I did, for months. This morning on [Hacker News](https://news.ycombinator.com/item?id=24130712), however, I came across an interesting new library called [`aiosql`](https://github.com/nackjicholson/aiosql). This is really a variation on a theme - a very simple, clean API that allows you to keep all of your SQL statements in a separate `.sql` file but to trivially call them as named methods from within your Python application code.

From the example on the Github page, let's assume that you have the following SQL file:

```sql
-- name: get-all-users
-- Get all user records
select userid,
    username,
    firstname,
    lastname
from users;

-- name: get-user-by-username^
-- Get user with the given username field.
select userid,
    username,
    firstname,
    lastname
from users
where username = :username;
```

You can then use the following to load your queries into your Python code:

```python
import sqlite3
import aiosql

conn = sqlite3.connect("myapp.db")
queries = aiosql.from_path("users.sql", "sqlite3")

users = queries.get_all_users(conn)
# >>> [(1, "nackjicholson", "William", "Vaughn"), (2, "johndoe", "John", "Doe"), ...]

users = queries.get_user_by_username(conn, username="nackjicholson")
# >>> (1, "nackjicholson", "William", "Vaughn")
```

Three things you'll notice immediately are: queries are called according to the names given to them in the SQL file (in this case, `get-all-users()` and `get-user-by-username()`); you can include descriptive comments in the SQL file itself; and that queries can be parametrized using the `^` and `:<parameter>` notation in your SQL file.

Personally, I find this much cleaner and more legible than simply writing out my SQL queries as Python strings. It also allows you to use your SQL files in other tools, such as `psql`, `pgcli` or `litecli`.

Several of the commenters on Hacker News mentioned similarities between `aiosql` and [`pugsql`](https://github.com/mcfunley/pugsql), another thin API that provides for the use of parametrized queries stored in a separate `.sql` file. I haven't looked at that yet but intend to update this post in future once I have.

For now, I'm going to use `aiosql` in a bunch of my projects which don't require an ORM.

Now that we've finished, don't forget to close the database connection!

```python
conn.close()
```