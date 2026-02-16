---
title: Modeling Tags in a Postgres schema
tags: [sql, postgres]
date: 16/02/2026
---

When working on a db schema to model the [OpenGraph protocol](https://ogp.me) I made an initial mistake with
how I modelled tags within the database.

<!-- more -->

The OpenGraph protocol is used to supply information about web pages in a predictable format. This is useful
as sites can use the protocol to create things such as the link previews you will have seen on social media.
The protocol specifies many data points, and has a concept of _data types_. For example you can have a web page
for a movie. This would be represented by the _video.movie_ type in the protocol. This can contain nested types
like _video:actor_ and _video:director_, so it's perfect for modeling in a relational database.

Quite a few of these entities can contain _tags_, which are represented by strings. Initially I opted for a simple pivot
table for each instance of a tag. This was a mistake as I ended up having to create many tables along the lines of
`og_book_tag` and `og_article_tag` for example.

As always if something feels wrong it usually is. I took a quick look over the schema and realised that the initial
_og_object_ (which is the parent entity) is referenced in each of these child tables, 
and as such I could create an _og_tags_ table to track all the tags as I have done below.

```sql
create table og_tags (
    id bigint generated always as identity primary key,
    og_object_id bigint not null references og_objects(id) on delete cascade,
    tag text not null,
    created_at timestamptz not null default now(),
    primary key(og_object_id, tag)
);

-- always remember to add your indices
create index idx_og_tags_og_object_id on og_tags(og_object_id);
create index idx_og_tags_tag on og_tags(tag);
```

Now I can select the tags relevant to an entity using joins. 

In the example below I get the tags for the _og_video_movies_ related to the _og_object_ 
with an id of `123`. This means that I can delete all the redundant pivot tables I had initially created.

```sql
SELECT t.tag
FROM og_tags t
JOIN og_objects o ON t.og_object_id = o.id
JOIN og_video_movies vm ON vm.og_object_id = o.id
WHERE vm.og_object_id = 123; 
```
