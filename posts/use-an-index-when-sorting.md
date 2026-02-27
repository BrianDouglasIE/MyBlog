---
title: Use an index when sorting
tags: [sql]
date: 26/02/2026
---

When working with a database it is a very common task to retrieve a sorted
list of entities. What you might not realize is that this can be a very 
expensive operation.

<!-- more -->

If an index is not present on the table which the select is being used on
a filesort may be performed. This is what happens in the example below using
MariaDB. When performing a file sort, the database fetches every row and sorts
each in memory before performing a query, which in this case is selecting the
first 10 results.

Applying an index on the column you wish to sort by (`ORDER BY`) means that the
database can skip the sort altogether. Instead of the previous scenario of
fetching all rows and then sorting them, the index will allow it to just grab the
first 10 entries.

## The Top-N query

<magpie-trinket>He hasn't just made up the term _Top-N query_ here is a reference and explantation from [Apache Flink](https://nightlies.apache.org/flink/flink-docs-master/docs/sql/reference/queries/topn/#:~:text=Top%2DN%20queries%20are%20useful,express%20a%20Top%2DN%20query.).</magpie-trinket>

A Top-N query is where you want to get the first N or last N results of table. It's
a super common query to perform. But as we explained above, it can be very slow if used
without an index.

I'll be using the schema displayed below for each of the sql snippets in this post.
The schema represents an example database that can be found on the
[mariadbtutorial.com](https://www.mariadbtutorial.com/getting-started/mariadb-sample-database/)
website.

![nation db schema](/images/nation-schema.png)

You'll see from the above schema that each `countries` entity can have many `country_stats`
entities. This means there is likely to be a lot of `county_stats` entities. So let's fetch
the top 10 `country_stats` by population.

```sql
SELECT country_id, population
FROM country_stats
ORDER BY population DESC
LIMIT 10;
```

Running the following query with `explain` tells me that 9514 rows were checked and that
a filesort was performed. This confirms that I wrote above, when there is no index the
database has to search through each row THEN do the sort.

Let's fix this by adding an index on the `country_stats` table targeting the `population`
column in descending order.

```sql
CREATE INDEX idx_population_desc ON country_stats (population DESC);
```

Now when the original query is run, `explain` tells me that only 10 records were checked
and that the `idx_population_desc` index was used.

<table>
    <thead>
        <tr>
            <th>Method</th>
            <th>Rows Checked</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>Using filesort</td>
            <td>9514</td>
        </tr>
        <tr>
            <td>Using index</td>
            <td>10</td>
        </tr>
    </tbody>
</table>

So when doing a Top-N query make sure to add an index on the column and specify the order
you wish to sort by.