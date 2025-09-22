![Idempotent word art](idempotent.webp)

The secret word of the day is **idempotent**. This fancy word can be used when working with web services and SQL databases. In programming, it means that a function will return the same result no matter how many times it is called. 

This means it's safe to repeat the same request; you will always get the same outcome. In asynchronous environments that may have a retry mechanism, like many modern web services, it ensures a consistent response from the application. It is also useful when performing an SQL database "migration"

You can think of traditional relational databases like a data table. The **schema** defines the column names and the type of data they can contain. Each row (or record) on the table contains a value. Empty or 'null' can be a value. 

A **migration** is an SQL operation that modifies these columns on a table, changing the schema by adding or removing columns.  A **mutation** is just updating values, changing cells, or adding and removing rows. Coming from AWS, where NoSQL is more common, I felt a bit silly asking devs, "What do you mean we are 'firing a mutation'?"

Fortunately, you don't need to learn SQL to work with databases using AI. Supabase has excellent integration with popular vibe coding tools like V0, and Claude can write all the code you need. Though it sometimes forgets to make migrations Idempotent, or even name them correctly! 

Supabase keeps all your migrations and applies them in order whenever you reset the database.  By ensuring they are Idempotent, you can run them any number of times without breaking the schema, so retries or resets never add duplicate columns or mismatched tables.

Whenever I modify the schema using AI, I ask Claude to confirm that we adhere to the project guidelines, which specify naming conventions and idempotency. I'm always surprised by how many times it says "Oops! I forgot all about that, let's fix the migration."

Here's my supabase_guidelines.md file. I'd be curious to hear your tricks for working on databases with AI - and if these guidelines make sense to an actual engineer!

Remember when someone says the secret word, scream! That should liven up the meeting. 
