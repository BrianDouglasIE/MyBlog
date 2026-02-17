---
title: Add generics to MariaDB Node Connector
date: 17/02/2026
tags: [sql, node]
---

When working with an instance of a MariaDB database I noticed that the [npm package](https://www.npmjs.com/package/mariadb)
used to connect did not support Typescript generics. This was something I felt I could fix, so I sent in a PR.

<!-- more -->

## Why generics?

Generics allow us to replace loosely typed APIs, anywhere `any` is used in Typescript, with contextual type
information. You can think of a generic as a wildcard type. These are useful in the context of connecting to a database
as they allow us to typehint the return type of queries, as well as the argument lists of prepared statements, for example.
This means that we get _compile time_ safety when working with queries and statements.

I've added a short video below to demonstrate the advantage of this. You'll notice that when the `email` value is changed to a 
`number`, an error is shown as it does not match the type supplied to the prepared statement.

<video controls width="600" style="max-width: 100%">
  <source src="https://private-user-images.githubusercontent.com/164613613/501041355-5d9c210c-8eb7-49b5-be9f-43890d2c5aa5.mp4?jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3NzEzMjY1OTcsIm5iZiI6MTc3MTMyNjI5NywicGF0aCI6Ii8xNjQ2MTM2MTMvNTAxMDQxMzU1LTVkOWMyMTBjLThlYjctNDliNS1iZTlmLTQzODkwZDJjNWFhNS5tcDQ_WC1BbXotQWxnb3JpdGhtPUFXUzQtSE1BQy1TSEEyNTYmWC1BbXotQ3JlZGVudGlhbD1BS0lBVkNPRFlMU0E1M1BRSzRaQSUyRjIwMjYwMjE3JTJGdXMtZWFzdC0xJTJGczMlMkZhd3M0X3JlcXVlc3QmWC1BbXotRGF0ZT0yMDI2MDIxN1QxMTA0NTdaJlgtQW16LUV4cGlyZXM9MzAwJlgtQW16LVNpZ25hdHVyZT0yZDI0YzM5MzkzNjYwZjRhMTcxNTk0NWE2YmQyOTUxODc4MWYzYWFlZGEwYTIzNjliOWZiZmM1N2UxNzVlNzhjJlgtQW16LVNpZ25lZEhlYWRlcnM9aG9zdCJ9.TUdHL9g4gK6KeZz8Knzzst5dwJO7_y-x3pRADS1O7oY" type="video/mp4">
  Your browser does not support the video tag.
</video>

Code example from video:

```typescript
type UserInsertParams = [string, Buffer, Buffer<ArrayBufferLike>]
const userInsertSQL = 'INSERT INTO users (email, password_hash, salt) VALUES (?, ?, ?)'
const userInsert = await conn.prepare<UserInsertParams>(userInsertSQL) // UserInsertParams passed as values type

const salt = crypto.randomBytes(16);
return await userInsert.execute([ // will only accept a value of type UserInsertParams
  email,
  crypto.pbkdf2Sync(password, salt, 310000, 32, 'sha256'),
  salt
])
```

## Implementing generics

<magpie-trinket>You can see the source PR at [mariadb-connector-nodejs/pull/334](https://github.com/mariadb-corporation/mariadb-connector-nodejs/pull/334)</magpie-trinket>

This is actually something that is easy to implement. Even though the mariadb connector has an extensive codebase, 
I knew that I was looking for a .d.ts file that contained the type definitions. Usually this file is generated on the fly,
however in this instance it seemed to be static. Using the LSP command "go to definition" is was able to find the `Prepared`
interface inside a `callback.d.ts` file.

I notticed that the interface lacked a generic argument, as expected. So to fix I added the following:

```typescript
interface Prepare {} // before

interface Prepare<V> {} // after
```

This allowed for any method within the interface to receive a generic also. See the `execute` method below.
Note that exceute did use a generic for it's result, but not for it's argument list.

```typescript
execute<T = any>(values: any, callback: (err: SqlError | null, result?: T, meta?: any) => void): void; // before

execute<T = any>(values: V, callback: (err: SqlError | null, result?: T, meta?: any) => void): void; // after 
```

The full list of changes can been seen on the pr, [mariadb-connector-nodejs/pull/334](https://github.com/mariadb-corporation/mariadb-connector-nodejs/pull/334).

I'm always glad to be able to contribute to an open source library. Especially a contribution that benefits my use of
said library.
