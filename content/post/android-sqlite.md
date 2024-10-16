---
draft: false
title: Databases In Android - Part 1
summary: Part 1 of a mostly code look at using SQLite in Android (not using Room)
author: Eric Daly
date: 2017-05-26
thumbnail: images/uploads/pexels-sergei-starostin-6466141.jpg
feature_image: /images/pexels-sergei-starostin-6466141.jpg
tags:
  - android
---

Android provides an **easy** way to connect to SQLite databases. It's so easy, assuming you know how to query a database and follow some guides. But if you need to do anything more complicated than a few tables with low record counts you'll find that there's a lot more to it. Welcome to [ContentProviders](https://developer.android.com/guide/topics/providers/content-providers.html), [CursorLoaders](https://developer.android.com/reference/android/content/CursorLoader.html), and [CursorAdapters](https://developer.android.com/reference/android/widget/CursorAdapter.html). The good news is that a lot of these concepts translate into different environments besides Android.

In this post (this might end up being a multi-part series), I want to dive into how all these things work together, explore some alternatives, and go into some specific examples while keeping it relatively simple. I want this to be an overview of the concepts that can be referenced later on.

<br>

Getting Started With SQLite
---------------------------
If you know a little bit about writing queries, the SQLite part should be pretty easy. Generally you set up a contract class that models the schema of your database. The outer class defines the database, and the inner classes define each table. BaseColumns by the way is used to include \_ID as a column name, so that your datababase plays nicely with certain Android classes like CursorAdapter.

If you're starting out with a new app, you'll need to insert a new class and call it something like databaseNameContract, or in my case ProductContract. Here's an example from a sample database app I made:

```java
public final class ProductContract {
    // private constructor prevents accidental instantiation of the contract class
    private ProductContract() {}

    // inner class defines the table
    public static class ProductEntry implements BaseColumns {
        public static final String TABLE_NAME = "product";
        public static final String COLUMN_NAME_NAME = "name";
    }
}
```

<br>
So now that we have the simplest schema in the world (and pretty useless), we can move on to the next thing: SQLiteOpenHelper. Apparently we need some help opening SQLite, so it's time to insert another class. This one will be called ProductHelper and it extends SQLiteOpenHelper. The point of all this is to keep track of the database version, and manage database creation and updates. It will come in especially handy several app versions later when the database structure changes.

```java
public class ProductHelper extends SQLiteOpenHelper {
    public static final int DATABASE_VERSION = 1;
    public static final String DATABASE_NAME = "products.db";
    private static final String SQL_CREATE_ENTRIES = "CREATE TABLE " + ProductEntry.TABLE_NAME + " (" +
            ProductEntry._ID + " INTEGER PRIMARY KEY," +
            ProductEntry.COLUMN_NAME_NAME + " TEXT)";

    public ProductHelper(Context context) {
        super(context, DATABASE_NAME, null, DATABASE_VERSION);
    }

    @Override
    public void onCreate(SQLiteDatabase db) {
        db.execSQL(SQL_CREATE_ENTRIES);
    }

    @Override
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {
        onCreate(db);
    }
}
```

<br>
Dang, this is starting to look like a lot of stuff but it seems like there's no end in sight. There is an end, we just can't see it yet. Just a couple more things and we'll have a legit working (but barely) database app.

<br>
<br>
To be continued...
