---
title: 'Databases In Android - Part 2'
date: 2017-06-09
feature_image: "/images/pexels-sergei-starostin-6466141.jpg"
author: "Eric Daly"
tags: [
  "android",
]
summary: "Part 2 of a mostly code look at using SQLite in Android (not using Room)"
---

Here's my thought of the day: I recently helped out on a public forum for some cloud file sync software I like to run, and this guy asked how to move a folder from one partition to another because he accidentally installed the package on the root partition with limited space. After I answered that he could just use the mv command (Ubuntu Server), he responded I think via email which included his signature and a little about himself, as in his credentials. Apparently he is a Linux Consultant.
<br>
I don't say this as an insult to the guy. I appreciate that people ask for help, and are willing to learn (and polite!). There's nothing wrong with that. Maybe he's just getting started, but this is what he wants to become an expert in, so he's in the mindset that he **is** an expert so that he **becomes** an expert.
<br>
I guess the point is that my first reaction was "This guy is over-selling himself." But maybe part of the reason I think that is because I under-sell myself. Hm... just a thought. There still needs to be a balance though... I mean come on. I guess since I've created my first Hello World app, it's time to go update my resume:
> Android Master

<br>
<br>

So in part 1 I talked about creating a contract class which represents the database schema, and started setting up the database helper which defines the database version and handles creation and upgrades/downgrades of the database.

CursorAdapters
--------------

The next point on the list is the CursorAdapter. I don't think I can explain it any better than the Android documentation:
> Adapter that exposes data from a Cursor to a ListView widget.

But it doesn't hurt to try. The CursorAdapter maps the values from a cursor to the fields in a ListView.

<br>
Before you make the CursorAdapter, you need to know which fields you want to display in your layout. Here's what my list_item.xml looks like. This layout gets loaded inside of another layout's ListView, but that's another topic I don't want to get into right now.

```xml
<?xml version="1.0" encoding="utf-8"?><!-- Layout for a single list item in the list of pets -->
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="vertical"
    android:padding="@dimen/activity_margin">

    <TextView
        android:id="@+id/tv_product_name"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:fontFamily="sans-serif-medium"
        android:textAppearance="?android:textAppearanceMedium"
        android:textColor="#2B3D4D" />

</LinearLayout>
```

So the list_item layout only shows a single TextView, which is going to be the product name. Now we can start working on the CursorAdapter.

Here's what my ProductCursorAdapter looks like.

```java
public class ProductCursorAdapter extends CursorAdapter {

    public ProductCursorAdapter(Context context, Cursor c) {
        super(context, c, 0 /* flags */);
    }

    @Override
    public View newView(Context context, Cursor cursor, ViewGroup parent) {
        return LayoutInflater.from(context).inflate(R.layout.list_item, parent, false);
    }

    @Override
    public void bindView(View view, Context context, Cursor cursor) {
        TextView productNameTV = (TextView) view.findViewById(R.id.tv_product_name);

        String productNameString = cursor.getString(cursor.getColumnIndexOrThrow(ProductEntry.COLUMN_NAME_NAME));

        productNameTV.setText(productNameString);
    }
}
```

Notice in the bindView method, this is where you tie the data from the cursor to the elements in the ListView. Again, in this case all we're doing is getting the tv_product_name TextView, and setting the text to productNameString which comes from the cursor. The CursorAdapter automatically does this for all items in the Cursor, adding to the ListView as many items as needed.

<br>

I'm out of time today, so I'll continue to ContentProviders next time.
