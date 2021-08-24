---
categories:
- System administration
date: "2020-04-15"
title: "Recover OwnCloud calendars"
description: "How to recover calendars from a OwnCloud database dump."
keywords: ["owncloud", "nextcloud", "caldav", "calendar", "recover", "dump", "database", "sql", "removed"]
aliases: ["/recover-owncloud-calendar/"]
---

I had to recover someone's calendars from an OwnCloud SQL dump.
I will detail the steps I went through in this post.

# Export CalDav calendar from SQL dump

To explore a large SQL file, here `my_database_dump.sql`, it is easier to import it
as a new database rather than exploring the text file.
Here I am using PostgreSQL but it should also work with MySQL or MariaDB.

```bash
sudo -u postgres createdb owncloud_backup
sudo -u postgres psql owncloud_backup < my_database_dump.sql
```

## Identify calendars in database

Open a new database connection,

```bash
$ sudo -u postgres psql owncloud_backup
```

Now that the database is loaded, it is possible to execute SQL queries to locate the identifier `id` of calendars we want to recover.
You can use a query like this one,

```sql
SELECT id, principaluri FROM oc_calendars WHERE principaluri LIKE '%username';
```

## Extract CalDav files from database

With each calendar `id`, you can now adapt and run this little script to extract
CalDav objects.
You might need to install `psycopg2` Python module.

If you want to identify as `postgres` to the database, you might need to run this script as `postgres` user.

```python
import psycopg2

# Calendar id to extract
calendarid = 42  # Change me!

# Open a connection to owncloud_backup database
connection = psycopg2.connect(
    user="postgres",
    database="owncloud_backup",  # Change me!
)

# Query all calendar objects
cursor = connection.cursor()
cursor.execute(f"SELECT calendardata, uri FROM oc_calendarobjects WHERE calendarid = {calendarid}")
records = cursor.fetchall() 

# Save each ICS object in /tmp/calendar_42/
for record in records:
    calendardata, uri = record[0], record[1]
    with open(f"/tmp/calendar_{calendarid}/" + uri, 'w') as f:
        f.write(calendardata.tobytes().decode('utf-8'))
```

Now you should have a collection of [iCalendar files](https://fr.wikipedia.org/wiki/ICalendar)
in `/tmp/calendar_ID/`.

# Import calendar

You can concatenate all `.ics` from a directory to generate a iCalendar
then import this calendar.

You can also directly open the WebDav folder with a file explorer and copy-paste these files.
For exemple, a GVFS browser (e.g. Nautilus, Thunar, Dolphin) supports a URI like
`davs://owncloud.crans.org/remote.php/caldav/calendars/my_username/my_calendar_name/`.
You will be able to find this URL in OwnCloud calendar app.

After that, your calendars should be restored!
