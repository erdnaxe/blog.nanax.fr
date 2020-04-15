---
layout: post
title: "Recover OwnCloud calendars"
description: "How to recover calendars from a OwnCloud database dump."
comments: true
category: miscellaneous
icon: assets/icons/icon-doc.svg
keywords: ["owncloud", "nextcloud", "calendar", "recover", "dump", "database", "sql", "removed"]
lang: en
---

Learn how to recover calendars from a OwnCloud database dump.

Recently I had to recover someone's calendars from a large OwnCloud SQL dump.
I will detail the steps I went through in this post.

**Table of content**:

* TOC
{:toc}

# Export CalDav calendar from SQL dump

To explore a large SQL file, the easiest is to import it as a new database.
Here I am using PostgreSQL but it should work with MySQL/MariaDB.

```bash
sudo -u postgres createdb owncloud_backup
sudo -u postgres psql owncloud_backup < my_database_dump.sql
```

## Identify calendars in database

Now you can explore the database and recover the `id` of calendars you want to
recover.

```bash
sudo -u postgres psql owncloud_backup
```

You can use a query like this one,

```sql
SELECT id, principaluri FROM oc_calendars WHERE principaluri LIKE '%username';
```

## Extract CalDav files from database

With the calendar `id`, you can now adapt and run this little script to extract
caldav objects.

```python
import psycopg2

calendarid = 42  # Change me!

connection = psycopg2.connect(
    user="postgres",
    database="owncloud_backup",  # Change me!
)

cursor = connection.cursor()
cursor.execute("SELECT calendardata, uri FROM oc_calendarobjects WHERE calendarid = " + str(calendarid))
records = cursor.fetchall() 

for record in records:
    calendardata, uri = record[0], record[1]
    with open("/tmp/" + str(calendarid) + "/" + uri, 'w') as f:
        f.write(calendardata.tobytes().decode('utf-8'))
```

# Import calendar

You can concatenate all `.ics` from a resulting CalDav directory to generate a iCalendar
then import this calendar.

You can also directly open the Dav folder with a file explorer and copy-paste these files.
For exemple, Nautilus support a URI like
`davs://owncloud.crans.org/remote.php/caldav/calendars/my_username/my_calendar_name/`.

You will be able to find this URL in OwnCloud calendar app.

After that, your calendars should be back!
