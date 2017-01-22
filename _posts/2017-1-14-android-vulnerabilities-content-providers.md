---
layout: default
title: Identifying and Fixing Security Vulnerabilities of Android Applications - Content Providers
permalink: /android/security/:title
---

### Prerequisites 

* [InsecureBankv2](https://github.com/dineshshetty/Android-InsecureBankv2) installed on Android device or emulator
* [Drozer](https://labs.mwrinfosecurity.com/tools/drozer) agent installed on Android device or emulator
* Drozer console runs and connected to drozer agent

### Getting information

Let's use [Drozer](https://labs.mwrinfosecurity.com/tools/drozer) to determine content providers that InsecureBank application exports:

```
dz> run app.provider.info -a com.android.insecurebankv2
Package: com.android.insecurebankv2
  Authority: com.android.insecurebankv2.TrackUserContentProvider
    Read Permission: null
    Write Permission: null
    Content Provider: com.android.insecurebankv2.TrackUserContentProvider
    Multiprocess Allowed: False
    Grant Uri Permissions: False
```

As seen from the above output, there is one content provider which is not protected by a permission. This means that any application on the same device can read and write to that provider. Let's determine what data does the given provider deal with. Usually, content providers use either SQLite databases or files as an underneath data storage with which they operate. In case of an SQLite database, content providers may be vulnerable to SQL injection vulnerabilities. The next step going forward is to determine which URIs does the provider deal with. We can do this by inspecting a reversed engineered source code of an application, or by using a dedicated drozer command (in a real world scenario, we would probably do both just to be on the safe side):

```
dz> run app.provider.finduri com.android.insecurebankv2
Scanning com.android.insecurebankv2...
content://com.android.insecurebankv2.TrackUserContentProvider/
content://com.google.android.gms.games
content://com.android.insecurebankv2.TrackUserContentProvider
content://com.android.insecurebankv2.TrackUserContentProvider/trackerusers
content://com.android.insecurebankv2.TrackUserContentProvider/trackerusers/
content://com.google.android.gms.games/
```

Next, we need to find if drozer found actual URIs. So, let's try queyring those URIs that deal with application:

```
dz> run app.provider.query content://com.android.insecurebankv2.TrackUserContentProvider/
Unknown URI content://com.android.insecurebankv2.TrackUserContentProvider/
dz> run app.provider.query content://com.android.insecurebankv2.TrackUserContentProvider/trackerusers
| id | name   |
| 1  | dinesh |
```

We did not have luck with the first one, but we were able to successfully retrieve the content of `content://com.android.insecurebankv2.TrackUserContentProvider/trackerusers`, which, apparently, shows us users that have used an application. 

### Exploit

Let's determine now if that content provider is vulnerable to SQL injection. As specified in Android [content providers documentation](https://developer.android.com/guide/topics/providers/content-provider-basics.html#Query), when constructing a query, projection defines which columns should be returned. This is where we can try to inject control character `'` to determine whether provider is vulnerable:

```
dz> run app.provider.query content://com.android.insecurebankv2.TrackUserContentProvider/trackerusers/* --projection "'"
unrecognized token: "' FROM names ORDER BY name" (code 1): , while compiling: SELECT ' FROM names ORDER BY name
```

As you can see, `'` token was used when constructing our query and that resulted in a SQL error. This means that our input was passed to SQL unsanitized. Now, let's try to determine if we can actually determine the structure of a database by constructing more elaborate injection:

```
dz> run app.provider.query content://com.android.insecurebankv2.TrackUserContentProvider/trackerusers/* --projection "* from sqlite_master --"
| type  | name             | tbl_name         | rootpage | sql                                                                            |
| table | android_metadata | android_metadata | 3        | CREATE TABLE android_metadata (locale TEXT)                                    |
| table | names            | names            | 4        | CREATE TABLE names (id INTEGER PRIMARY KEY AUTOINCREMENT,  name TEXT NOT NULL) |
| table | sqlite_sequence  | sqlite_sequence  | 5        | CREATE TABLE sqlite_sequence(name,seq)                                         |
```

In the above attack we used string `* from sqlite_master --` effectively turning the query that was executed into `SELECT * from sqlite_master`, which just gave us all tables in the underlying database.

As a side note, we will mention that drozer even has a dedicated command to execute similar type of an attack:

```
dz> run scanner.provider.sqltables -a content://com.android.insecurebankv2.TrackUserContentProvider/trackerusers
Accessible tables for uri content://com.android.insecurebankv2.TrackUserContentProvider/trackerusers:
  android_metadata
  names
  sqlite_sequence
```

### Fix

Proper design should always start with a discussion whether there is a legitimate need to export a given content provider, and then, what would be proper permissions to protect it (Hint: Use _signature_ permission to prevent other apps not developed by you from communicating with your app). But we already spent some time with the topic of permissions in one of the previous posts, so, let's try to see how to avoid SQL injection per se. 

When discussing the topic of SQL injection, it is almost customary to introduce a vulnerability by doing string concatenation. However, when we open a code of `TrackUserContentProvider`, we will see that this is not the case:

```
public Cursor query(Uri uri, String[] projection, String selection,	String[] selectionArgs, String sortOrder) {
		SQLiteQueryBuilder qb = new SQLiteQueryBuilder();
		qb.setTables(TABLE_NAME);
		
		switch (uriMatcher.match(uri)) {
			case uriCode:
				qb.setProjectionMap(values);
				break;
			default:
				throw new IllegalArgumentException("Unknown URI " + uri);
		}
		if (sortOrder == null || sortOrder == "") {
			sortOrder = name;
		}
		Cursor c = qb.query(db, projection, selection, selectionArgs, null, null, sortOrder);
		
		// ...
	}
```

The developer used a class `SQLiteQueryBuilder` from class library, and called `query` method on it passing `projection` parameter received from the caller. So, let's look into the [source code of this class](https://android.googlesource.com/platform/frameworks/base/+/master/core/java/android/database/sqlite/SQLiteQueryBuilder.java). At the time when I checked (January 2017), the `query` method had the following comment which indicated about protection against SQL injection in `selection` parameter:

```
// Validate the user-supplied selection to detect syntactic anomalies
// in the selection string that could indicate a SQL injection attempt.
```

So, there is a protection against injection in `selection` parameter. But what about `projection` parameter?! Turns out, more work is needed from the consumer of that API. 

There seems to be two ways how to deal with it. The first way that comes into mind is to do a whitelisting of fields that we allow to query. After all, `projection` parameter defines fields that may exist in the database, so, we can check whether the parameter contains legitimate fields before continuing with the query. Here is an example code one might add to the beginning of `query` method:

```
HashSet<String> validProjections = new HashSet<>(Arrays.asList("id", "name"));
for (String p : projection) {
    if (!validProjections.contains(p)) throw new IllegalArgumentException("Unknown projection: " + projection);
}
```

For our fix, we will choose another way by using `setProjectionMap` method from `SQLiteQueryBuilder` class. As stated in the [documentation](https://developer.android.com/reference/android/database/sqlite/SQLiteQueryBuilder.html#setProjectionMap(java.util.Map%3Cjava.lang.String,%20java.lang.String%3E)),

> If a projection map is set it must contain all column names the user may request, even if the key and value are the same.

`TrackUserContentProvider` already has a static `values` HashMap that is being used in the following method call:

```
qb.setProjectionMap(values);
```

The problem is that `values` HashMap is never populated. Let's add to it two possible columns that can exist in table `names`: `id` and `name` (Note, that since we don't specify a mapping between user provided names and database names, we use the same strings for keys and values in the HashMap):

```
values = new HashMap<>();
values.put("id", "id");
values.put("name", "name");
```

When we recompile our application and send it to the device, we will see that SQL injection does not work anymore:

```
dz> run app.provider.query content://com.android.insecurebankv2.TrackUserContentProvider/trackerusers/ --projection "'"
Invalid column '
```

### Summary

In this post we looked at exploiting SQL injection in unsecured content provider. We also showed how to fix this vulnerability either by introducing whitelisting or using a method from the framework.
