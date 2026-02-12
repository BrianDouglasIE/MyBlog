---
title: Some useful rsync tips
tags: [linux]
date: 12/02/2026
---

I've been using rsync to deploy this blog. Because I live on the edge and don't need any deploy pipeline headaches.
And tbh, this is blog is not critical software it's a static site hosted by a caddy server on UpCloud.
Anyway here are some quick tips from my experience deploying on the edge.

<!-- more -->

## The trailing slash matters

Take these two examples:

```
rsync -av source/ dest/
rsync -av source dest/
```

The first copies the contents of `source` into `dest`, the second `source` into `dest`.

<chicken-asks>I bet that one caught him out a few times</chicken-asks>
<magpie-replies>Well it is top of the list...</magpie-replies>

## There is a --dry-run flag

Super handy, adding the `--dry-run` flag show you what would happen if you run the command. Without actually
running it. 

Always use `--dry-run` if you are doing a `--delete`.

## Interrupted syncs can be resumed

```
rsync -av --partial --progress source/ dest/
```

The above will resume syncing where it left off if the connection is interrupted. This is useful when you are doing large uploads, video
for example. My connection is flaky at times so the combination of `--partial --progress` does come in handy.

## Certain folders can be excluded

```
rsync -av --exclude='node_modules/' source/ dest/
```

This blog is a simple js script, it uses bun now, but there was a time when it had a `node_modules` folder. Luckily rsync has an `--exclude` flag
that can be used to prevent unwanted files/folder getting synced.
