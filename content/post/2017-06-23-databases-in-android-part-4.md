---
draft: false
title: Databases In Android - Part 4
summary: Part 4 of a mostly code look at using SQLite in Android (not using Room)
author: Eric Daly
date: 2017-06-23
thumbnail: images/uploads/pexels-sergei-starostin-6466141.jpg
feature_image: /images/pexels-sergei-starostin-6466141.jpg
tags:
  - android
---

OK so in our Product sample, the ContentProvider looks like this:

```java
public class ProductProvider extends ContentProvider {
    public static final String LOG_TAG = ProductProvider.class.getSimpleName();
    private ProductHelper mDbHelper;

    // Set up URI matcher codes
    private static final int PRODUCTS = 100;
    private static final int PRODUCT_ID = 101;
    private static final UriMatcher sUriMatcher = new UriMatcher(UriMatcher.NO_MATCH);
    static {
        sUriMatcher.addURI(ProductContract.CONTENT_AUTHORITY, ProductContract.PATH_PRODUCT, PRODUCTS);
        sUriMatcher.addURI(ProductContract.CONTENT_AUTHORITY, ProductContract.PATH_PRODUCT + "/#", PRODUCT_ID);
    }

    @Override
    public boolean onCreate() {
        mDbHelper = new ProductHelper(getContext());
        return true;
    }

    @Nullable
    @Override
    public Cursor query(@NonNull Uri uri, @Nullable String[] projection, @Nullable String selection, @Nullable String[] selectionArgs, @Nullable String sortOrder) {
        SQLiteDatabase database = mDbHelper.getReadableDatabase();
        Cursor cursor;

        int match = sUriMatcher.match(uri);
        switch (match) {
            case PRODUCTS:
                cursor = database.query(ProductEntry.TABLE_NAME, projection, selection, selectionArgs, null, null, sortOrder);
                break;
            case PRODUCT_ID:
                selection = ProductEntry._ID + "=?";
                selectionArgs = new String[] {String.valueOf(ContentUris.parseId(uri))};
                cursor = database.query(ProductEntry.TABLE_NAME, projection, selection, selectionArgs, null, null, sortOrder);
                break;
            default:
                throw new IllegalArgumentException("Cannot query unknown URI " + uri);
        }

        cursor.setNotificationUri(getContext().getContentResolver(), uri);

        return cursor;
    }

    @Nullable
    @Override
    public String getType(@NonNull Uri uri) {
        final int match = sUriMatcher.match(uri);
        switch (match) {
            case PRODUCTS:
                return ProductEntry.PRODUCT_LIST_TYPE;
            case PRODUCT_ID:
                return ProductEntry.PRODUCT_ITEM_TYPE;
            default:
                throw new IllegalStateException("Unknown URI " + uri + " with match " + match);
        }
    }

    @Nullable
    @Override
    public Uri insert(@NonNull Uri uri, @Nullable ContentValues contentValues) {
        final int match = sUriMatcher.match(uri);
        switch (match) {
            case PRODUCTS:
                return insertProduct(uri, contentValues);
            default:
                throw new IllegalArgumentException("Insertion is not supported for " + uri);
        }
    }

    private Uri insertProduct(Uri uri, ContentValues values) {
        SQLiteDatabase db = mDbHelper.getWritableDatabase();

        long id = db.insert(ProductEntry.TABLE_NAME, null, values);

        getContext().getContentResolver().notifyChange(uri, null);

        return ContentUris.withAppendedId(uri, id);
    }

    @Override
    public int delete(@NonNull Uri uri, @Nullable String whereClause, @Nullable String[] whereArgs) {
        // Get writable database
        SQLiteDatabase database = mDbHelper.getWritableDatabase();

        int numRowsDeleted = 0;

        final int match = sUriMatcher.match(uri);
        switch (match) {
            case PRODUCTS:
                // Delete all rows that match the whereClause and whereArgs
                numRowsDeleted = database.delete(ProductEntry.TABLE_NAME, whereClause, whereArgs);
                break;
            case PRODUCT_ID:
                // Delete a single row from the pets table using the given ID
                whereClause = ProductEntry._ID + "=?";
                whereArgs = new String[] { String.valueOf(ContentUris.parseId(uri)) };
                numRowsDeleted = database.delete(ProductEntry.TABLE_NAME, whereClause, whereArgs);
                break;
            default:
                throw new IllegalArgumentException("Deletion is not supported for " + uri);
        }

        if (numRowsDeleted != 0) {
            getContext().getContentResolver().notifyChange(uri, null);
        }

        return numRowsDeleted;
    }

    @Override
    public int update(@NonNull Uri uri, @Nullable ContentValues contentValues, @Nullable String s, @Nullable String[] strings) {
        return 0;
    }
}
```

A ContentProvider should implement all of the CRUD methods, passing arguments through to the database, and then passing results back to the calling method. It's kind of like a reverse proxy...

Focusing on the query method first, it basically opens a readable copy of the database, and then either queries the whole product table or a single product depending on the URI passed in. You can set up arbitrary URI matcher codes at the top, but just make sure they're unique within the provider. If you noticed cursor.setNotificationUri, that's used for notifying the CursorLoader that a change was made so that it can automatically update. More on that later.

I'm tempted to say that the rest of the code is self explanatory... Even if that's not true, you can use this as a starting point and fill things in from there. This is pretty barebones as it is.

Next time we'll look at CursorLoader, and that should conclude the Databases In Android series.
