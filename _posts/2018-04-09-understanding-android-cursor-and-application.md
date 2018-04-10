---
layout:     post
title:      "理解Android中的Cursor及其应用"
date:       2018-04-09
author:     "winson"
tags:
    - Android
---

近来在做项目的时候发现在APP各组件间共享数据是件麻烦事，尤其涉及到异步更新的时候，就会出现各种烦人的listener，各种callback。于是想想是不是有什么统一接口来实现自动更新之类的。于是就想到了Cursor，于是就想到了用ContentProvider来实现这个功能。
<!-- TOC -->

- [0x00 从如何自定义一个ContentProvider开始](#0x00-从如何自定义一个contentprovider开始)
- [0x01 无法避免的Uri](#0x01-无法避免的uri)
    - [Uri的组成](#uri的组成)
    - [Uri的匹配与UriMatcher](#uri的匹配与urimatcher)
    - [扩展](#扩展)
- [0x02 Cursor的动态更新](#0x02-cursor的动态更新)
    - [应用场景之一、我们常用的API Loader与LoaderManager](#应用场景之一我们常用的api-loader与loadermanager)
    - [应用场景之二、数据库中的Cursor ： SQLiteCursor](#应用场景之二数据库中的cursor--sqlitecursor)
    - [应用场景之三、ContentProvider中的Cursor](#应用场景之三contentprovider中的cursor)
    - [原理](#原理)
    - [ContentObserver与DatasetObserver的区别](#contentobserver与datasetobserver的区别)
    - [绝佳范例一 : CursorAdapter](#绝佳范例一--cursoradapter)
    - [绝佳范例二 : CursorLoader](#绝佳范例二--cursorloader)

<!-- /TOC -->
## 0x00 从如何自定义一个ContentProvider开始
参考[官网教程](https://developer.android.com/guide/topics/providers/content-provider-creating.html)

## 0x01 无法避免的Uri
### Uri的组成
一旦使用上Cursor，就无法避免要使用这个类，然后就无法避免要搞清楚Android是如何组织Uri的。
[这篇文章](https://blog.csdn.net/harvic880925/article/details/44679239)说得比较清楚了，就不再重复了，这里只说一下重点。

基本结构是这样的

```
[scheme:][//authority][path][?query][#fragment]
```

其中authority又能分成`[userinfo@]host:port`的形式，所以其结构也可以是这样的
```
[scheme:][//[userinfo@]host:port][path][?query][#fragment]
```
userinfo一般写成`name:pwd`的形式。

其各部分的意义基本跟平时见到的网络uri差不多，就后面那个`[#fragment]`不一样。用`#`分割的后面那部分都是属于`fragment`。
举个栗子：
```
http://www.wtf.com:8080/yourpath/fileName.htm?index=10&path=32&id=4#winson
```

`http`就是`scheme`, `www.wtf.com:8080`就是authority，`/yourpath/fileName.htm`这部分就是`path`, `query`是一个键值对，多个键值对间用`&`分隔。
`winson`就是`fragment`。
<br>
都很简单直接。
<br>
那在Android中怎么操作这种Uri呢？Android的Uri类提供了众多操作Uri的方法，仅列举其中一些常用的：

- getScheme
  返回其中的authority部分，在上例中即`http`
- getAuthority
  返回其中的authority部分，在上例中就是`www.wtf.com:8080`
- getHost/getPort
  返回主机以及端口号，就是`www.wtf.com`以及`8080`
- getPath
  返回`/yourpath/fileName.htm`这部分
- getQuery 
  返回`index=10&path=32&id=4`这部分
- getFragment
  返回`winson`
- getPathSegments
  返回Path的所有分段，上例中就是返回`yourpath`, `fileName.htm`
- getLastPathSegment
  这个就跟上面那个一样，只是返回的是path中最后的那个元素，即`fileName.htm`
- getQueryParameterNames/getQueryParameters/getQueryParameter/getBooleanQueryParameter
  这几个都是类似的，只是方便我们取出URI中的查询参数
- withAppendedPath
  拼接一个`path`到uri上。很好理解，比如有一个uri是`content://my.app.com/media`，我想在后面拼个ID 20变成这样`content://my.app.com/media/20`，直接这么用`Uri.withAppendedPath(uri, "20");`可以简单理解为字符串拼接。
  其实还可以用另一个工具类`ContentUris`，`ContentUris.withAppendedId(uri, 20)`，一样的。

### Uri的匹配与UriMatcher
首先是匹配规则，可以参看[Android官网的说明](https://developer.android.com/guide/topics/providers/content-provider-creating.html?hl=zh-cn)。
简单说来就是记住两点就可以了：
- 没有特殊符号的URI直接全部匹配
- 符号`*`匹配由任意长度的任何有效字符组成的字符串
- 符号`#`匹配由任意长度的数字字符组成的字符串

`#`通常用于匹配单行内容，即某个ID的记录。但这其实完全看ContentProvider是怎么实现的，我硬是要提供很多也是可以的。只是不符合我们的默认约定。
<br>
`UriMatcher`就是基于以上规则而来，用法就是：
- 1. 往里面加规则
- 2. 调用match(uri)返回一个匹配码，告诉使用者匹配了哪种模式

所以我们一般的用法就是：<br>
第一，加规则：<br>
```

private static final int MATCH_TABLE3 = 1;
private static final int MATCH_TABLE3_ID = 2;
private static final int MATCH_TABLE3_ATTR1 = 3;

static {
    sUriMatcher.addURI("com.example.app.provider", "table3", MATCH_TABLE3);
    sUriMatcher.addURI("com.example.app.provider", "table3/#", MATCH_TABLE3_ID);
    sUriMatcher.addURI("com.example.app.provider", "table3/#/attr1", MATCH_TABLE3_ATTR1);
}
```
第二，用：<br>
```
public Cursor query(@NonNull Uri uri, @Nullable String[] projection, @Nullable String selection, @Nullable String[] selectionArgs, @Nullable String sortOrder) {
    ...
    ...
    switch (sUriMatcher.match(uri)) {
        case MATCH_TABLE3:
            ...
            break;
        case MATCH_TABLE3_ID:
            ...
            break;
        case MATCH_TABLE3_ATTR1:
            ...
            break;
    }
    ...
    ...
}
```

我一般不用`*`，因为觉得没这个必要吧。

### 扩展
- 绝对URI和相对URI
绝对URI：以scheme组件起始的完整格式，如http://fsjohnhuang.cnblogs.com。表示以对标识出现的环境无依赖的方式引用资源。 
相对URI：不以scheme组件起始的非完整格式，如fsjohnhuang.cnblogs.com。表示以对依赖标识出现的环境有依赖的方式引用资源。 

- 不透明URI和分层URI
不透明URI：scheme-specific-part组件不是以正斜杠(/)起始的，如mailto:fsjohnhuang@xxx.com。由于不透明URI无需进行分解操作，因此不会对scheme-specific-part组件进行有效性验证。 
分层URI：scheme-specific-part组件是以正斜杠(/)起始的，如http://fsjohnhuang.com。

## 0x02 Cursor的动态更新
首先我们来看看平时我们在哪里会用到Cursor<br>
### 应用场景之一、我们常用的API Loader与LoaderManager

步骤：
1. 获取LoaderManager. 这可以通过activity或者fragmentactivity获取，但是现在我们一般都是直接使用support包的appcompatactivity，所以都是`getSupportLoaderManager`
2. 然后初始化一个Loader：`loaderManager.initLoader(loader_id, bundle, callback);`
3. 实现上面的callback，一共三个方法`onCreateLoader`, `onLoadFinished`, `onLoaderReset`.
4. 当查询条件发生变化的时候重启Loader`loaderManager.restartLoader(loader_id, bundle, this);`

重点是callback的其中三个方法的实现.<br>
`onCreateLoader` : 创建一个Loader用来, 下面的例子是用来加载Cursor
`onLoadFinished` : 就是加载已经成功了，然后你就可以用这个加载完成的数据了，对于Cursor，就可以`mAdapter.swapCursor(newCurosr)`
`onLoaderReset` : 先前一个`onLoadFinished`传进来的数据要失效了，现在可以做些收尾的工作，对于Cursor，就是close了。一般跟Adapter一起用，就是`mAdapter.swapCursor(null)`

这个时候获取到的Cursor一般与CursorAdapter一起使用，~~CursorAdapter监视了Cursor的变化，一旦Cursor有变化，他就让他监视的Cursor requery, requery会触发注册在Cursor上的DataSetObserver, 而CursorAdapter在onChanged里面直接调用了   notifyDatasetChanged，这样就相当于透明实现了UI更新了，完全不需要程序员的代码参与，数据更新自动完成。但是有个坑，就是使用CursorAdapter的Cursor一定要有一列名字叫`_id`的列，否则不能实现自动更新。~~

> 不要使用CursorAdapter的自动更新功能， 因为其会直接在UI线程上更新。CursorLoader已经实现了自动更新。如果一定要使用CursorAdapter的自动更新，请确保数据列有'_id'列。

贴一段使用这些API的一般代码，来自[Android官方文档](https://developer.android.com/guide/components/loaders.html?hl=zh-cn)：

```
public static class CursorLoaderListFragment extends ListFragment
        implements OnQueryTextListener, LoaderManager.LoaderCallbacks<Cursor> {

    // This is the Adapter being used to display the list's data.
    SimpleCursorAdapter mAdapter;

    // If non-null, this is the current filter the user has provided.
    String mCurFilter;

    @Override public void onActivityCreated(Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);

        // Give some text to display if there is no data.  In a real
        // application this would come from a resource.
        setEmptyText("No phone numbers");

        // We have a menu item to show in action bar.
        setHasOptionsMenu(true);

        // Create an empty adapter we will use to display the loaded data.
        mAdapter = new SimpleCursorAdapter(getActivity(),
                android.R.layout.simple_list_item_2, null,
                new String[] { Contacts.DISPLAY_NAME, Contacts.CONTACT_STATUS },
                new int[] { android.R.id.text1, android.R.id.text2 }, 0);
        setListAdapter(mAdapter);

        // Prepare the loader.  Either re-connect with an existing one,
        // or start a new one.
        getLoaderManager().initLoader(0, null, this);
    }

    @Override public void onCreateOptionsMenu(Menu menu, MenuInflater inflater) {
        // Place an action bar item for searching.
        MenuItem item = menu.add("Search");
        item.setIcon(android.R.drawable.ic_menu_search);
        item.setShowAsAction(MenuItem.SHOW_AS_ACTION_IF_ROOM);
        SearchView sv = new SearchView(getActivity());
        sv.setOnQueryTextListener(this);
        item.setActionView(sv);
    }

    public boolean onQueryTextChange(String newText) {
        // Called when the action bar search text has changed.  Update
        // the search filter, and restart the loader to do a new query
        // with this filter.
        mCurFilter = !TextUtils.isEmpty(newText) ? newText : null;
        getLoaderManager().restartLoader(0, null, this);
        return true;
    }

    @Override public boolean onQueryTextSubmit(String query) {
        // Don't care about this.
        return true;
    }

    @Override public void onListItemClick(ListView l, View v, int position, long id) {
        // Insert desired behavior here.
        Log.i("FragmentComplexList", "Item clicked: " + id);
    }

    // These are the Contacts rows that we will retrieve.
    static final String[] CONTACTS_SUMMARY_PROJECTION = new String[] {
        Contacts._ID,
        Contacts.DISPLAY_NAME,
        Contacts.CONTACT_STATUS,
        Contacts.CONTACT_PRESENCE,
        Contacts.PHOTO_ID,
        Contacts.LOOKUP_KEY,
    };
    public Loader<Cursor> onCreateLoader(int id, Bundle args) {
        // This is called when a new Loader needs to be created.  This
        // sample only has one Loader, so we don't care about the ID.
        // First, pick the base URI to use depending on whether we are
        // currently filtering.
        Uri baseUri;
        if (mCurFilter != null) {
            baseUri = Uri.withAppendedPath(Contacts.CONTENT_FILTER_URI,
                    Uri.encode(mCurFilter));
        } else {
            baseUri = Contacts.CONTENT_URI;
        }

        // Now create and return a CursorLoader that will take care of
        // creating a Cursor for the data being displayed.
        String select = "((" + Contacts.DISPLAY_NAME + " NOTNULL) AND ("
                + Contacts.HAS_PHONE_NUMBER + "=1) AND ("
                + Contacts.DISPLAY_NAME + " != '' ))";
        return new CursorLoader(getActivity(), baseUri,
                CONTACTS_SUMMARY_PROJECTION, select, null,
                Contacts.DISPLAY_NAME + " COLLATE LOCALIZED ASC");
    }

    public void onLoadFinished(Loader<Cursor> loader, Cursor data) {
        // Swap the new cursor in.  (The framework will take care of closing the
        // old cursor once we return.)
        mAdapter.swapCursor(data);
    }

    public void onLoaderReset(Loader<Cursor> loader) {
        // This is called when the last Cursor provided to onLoadFinished()
        // above is about to be closed.  We need to make sure we are no
        // longer using it.
        mAdapter.swapCursor(null);
    }
}
```

### 应用场景之二、数据库中的Cursor ： SQLiteCursor
一看`SQLiteCursor`，可能不少人不知道这是什么东西，都没用过。其实你用过的，而且用得很多，只是你不知道而已。<br>
回顾一下远古时代我们是如何在Android里面使用数据库的，首先是继承`SQLiteOpenHelper`，重载其中的`onCreate`, `onUpgrade`, 看起来基本像下面这样：

```
public class MyDatabaseHelper extends SQLiteOpenHelper {

    private static final String DATABASE_NAME = "DBName";

    private static final int DATABASE_VERSION = 2;

    // Database creation sql statement
    private static final String DATABASE_CREATE = "create table MyEmployees
                                 ( _id integer primary key,name text not null);";

    public MyDatabaseHelper(Context context) {
        super(context, DATABASE_NAME, null, DATABASE_VERSION);
    }

    // Method is called during creation of the database
    @Override
    public void onCreate(SQLiteDatabase database) {
        database.execSQL(DATABASE_CREATE);
    }

    // Method is called during an upgrade of the database,
    @Override
    public void onUpgrade(SQLiteDatabase database,int oldVersion,int newVersion){
        Log.w(MyDatabaseHelper.class.getName(),
                         "Upgrading database from version " + oldVersion + " to "
                         + newVersion + ", which will destroy all old data");
        database.execSQL("DROP TABLE IF EXISTS MyEmployees");
        onCreate(database);
    }
}
```

> 也可以重载`onConfigure`，这样可以设置数据库的一些特性，比较少用，[参考这个](https://github.com/codepath/android_guides/wiki/Local-Databases-with-SQLiteOpenHelper)

然后用的话就是这样：

```
private MyDatabaseHelper dbHelper = new MyDatabaseHelper(getContext());
...
...
Cursor c = dbHelper.query(false, table, projection, selection, selectionArgs...);
if (c.moveToFirst()) {
    do {
        String attr1 = c.getString(...)
        int attr2 = c.getInt(...)

        JavaBean bean = new JavaBean(attr1, attr2...);
        list.add(bean);
    } while(c.moveToNext());
}
return list;
```

> `SQLiteDatabase`还有两个更基础的方法叫`rawQuery`以及`execSQL`，前者用于需要返回`Cursor`的情况，后者用于不需要返回的情况，不限于查询，比如`drop table`这种。

这就是很多ORM框架做的事情，当然他们的实现会复杂的多，但基本原理就是上面那样。<br>
这里返回的Cursor就是`SQLiteCursor`实例，或者说默认返回的是这个，如果你觉得默认的`SQLiteCursor`不满足你的需求，需要查询结果返回cursor之前做一些定制，比如打印sql语句这种，你就可以这样：

```
CursorFactory cursorFactory = new CursorFactory(){
    @Override public Cursor newCursor(final SQLiteDatabase db,final SQLiteCursorDriver masterQuery,final String editTable,final SQLiteQuery query){
      Log.d(TAG,query.toString());
      return new SQLiteCursor(db,masterQuery,editTable,query);
    }
};
db.queryWithFactory(cursorFactory, ...);
```

当然了这纯属闲得蛋疼，但你可以扩展到做其他的事情，比如记录日志啊，防止SQL注入什么的。参考[这里](http://www.javased.com/index.php?api=android.database.sqlite.SQLiteDatabase.CursorFactory)<br>

### 应用场景之三、ContentProvider中的Cursor

在ContentProvider的query中设置NotificationUri

```
@Override
public Cursor query(Uri uri, String[] projection, String selection, String[] 
selectionArgs,String sortOrder) {
    SQLiteDatabase database = sqLiteOpenHelper.getReadableDatabase();
    Cursor cursor = database.query(ContactContract.CONTACT_TABLE, projection,
    selection,selectionArgs, null, null, sortOrder);
    //设置NotificationUri
    cursor.setNotificationUri(contentResolver, ContactContract.SYNC_SIGNAL_URI);
    return cursor;
}
```

在ContentProvider的insert，update，delete中触发NotificationUri

```
@Override
public Uri insert(Uri uri, ContentValues values) {
    SQLiteDatabase database = sqLiteOpenHelper.getWritableDatabase();
    long id = database.insert(ContactContract.CONTACT_TABLE, null, values);

    if (id >= 0) {
        //触发NotificationUri
        contentResolver.notifyChange(ContactContract.SYNC_SIGNAL_URI, null);
    }

    return uri.withAppendedPath(ContactContract.CONTACT_URI, String.valueOf(id));
}

@Override
public int update(Uri uri, ContentValues values, String selection, 
    String[] selectionArgs) {
    SQLiteDatabase database = sqLiteOpenHelper.getWritableDatabase();
    int result = database.update(ContactContract.CONTACT_TABLE, values, 
        selection, selectionArgs);

    if (result > 0) {
        //触发NotificationUri
        contentResolver.notifyChange(ContactContract.SYNC_SIGNAL_URI, null);
    }

    return result;
}

@Override
public int delete(Uri uri, String selection, String[] selectionArgs) {
    SQLiteDatabase database = sqLiteOpenHelper.getWritableDatabase();
    int result = database.delete(ContactContract.CONTACT_TABLE, selection, 
        selectionArgs);

    if (result > 0) {
        //触发NotificationUri
        contentResolver.notifyChange(ContactContract.SYNC_SIGNAL_URI, null);
    }

    return result;
}
```

### 原理
在回顾了相关常用使用方法后，我们来研究下其原理。<br>
几个关键方法：<br>
Cursor.setNotificationUri : 设置通知的URI，跟`ContentResolver.notifyChanged`一起用，notifyChanged的时候填的URI就是这个时候设置的URI<br>
ContentResolver.notifyChanged : 纯粹的通知作用，<br>
Cursor.registerContentObserver : 注册一个监听器，一旦数据源中跟这个cursor关注的URI对应的数据发生变化，就会触发这个监听器。<br>
Cursor.registerDataSetObserver : 注册一个监听器，一旦Cursor(的缓冲区)里面的数据发生了变化就触发这个监听器。<br>

以上几个方法必定成套使用，除非作为代替你手动调用了Cursor里面的onChanged方法。

其基本过程是，当查询发起方向数据源发起了一个query，数据源会生成一个新的Cursor，并且会调用这个Cursor的setNotificationUri。当调用了这个方法的时候Cursor就会在内部调用ContentResolver.registerContentObserver关注自己URI的变化。
当有其他的代码调用这个数据源的`insert`、`delete`、`update`这种操作改变了相关数据时，数据源会调用ContentResolver.notifyChanged向一个特定的URI发出通知。这个时候如果Cursor监听的URI匹配这个通知的URI，那Cursor的内部类的SelfContentObserver.onChange就会被触发，然后Cursor.onChanged就会被调用。这个时候如果这个Cursor上注册有ContentObserver，那这些ContentObserver的onChanged就会被触发。<br>
<br>
注意！这个时候Cursor的内容还没有发生变化，是数据源发生了变化，这个通知只是告诉你数据源里面你感兴趣的数据比如那查询的那张表的数据发生了变化，至于你是不是要去重新获取数据，那是你的事。<br>\
要想重新获取数据，请使用`Cursor.requery`或者直接重新查询获取新的Cursor。

如果使用了`Cursor.requery`，那这个时候注册在这个Cursor上的DataSetObserver的`DataSetObserver.onChanged`会被触发，表明Cursor的数据已经更改了，请重新刷新相关界面或进行相关的处理工作。

对于动态更新的逻辑的处理，因为涉及到新旧Cursor切换以及大多数情况下会涉及UI更新以及Acitivty或Fragment生命周期的切换等，处理起来比较麻烦。一句话，Cursor直接涉及到UI的推荐直接使用Loader与LoaderManager相关API，否则自己管理。

### ContentObserver与DatasetObserver的区别
1. ContentObserver
   ContentObserver主要是通过Uri来监测特定的Databases的表，如果该Databases表有变动则会通知更新cursor中的数据。
   如果使用ContentProvider操作数据库，在ContentProvider的query()方法中会通过Cursor.setNotificationUri()注册uri描述的表，在insert，delete，query操作之后都会调用getContext().getContentResolver().notifyChange()。是当uri描述的db表中有insert，delete，query操作之后，notifyChange()会通知该cursor注册的ContentObserver，并调用ContentObserver的onChange方法。CursorAdapter的onChange一般会调onContentChanged，在onContentChanged中调用Cursor.requery()来更新cursor中的数据。

   <font color="red">用途：database table中有变动后通知用户刷新cursor中的数据。</font>

2. DatasetObserver
   DatasetObserver主要是当注册它的cursor中发生变动时会调用其中的方法，让用户做一些界面刷新等操作。
   首先cursor通过registerDataSetObserver()注册DatasetObserver   当cursor数据有变动时，例如调用了cursor的requery()，会调用cursor的onChanged通知用户cursor中的内容有变动，用户可以在onChanged里做一些刷新界面的操作。一般会在onChanged里调用notifyDataSetChanged通知framework，framework收到通知会调用CursorAdapter的getView来做界面刷新等工作。

   <font color="red">用途：cursor中的数据有变动时通知用户刷新界面。</font>

### 绝佳范例一 : CursorAdapter
对于如何使用Cursor的动态更新功能，`CursorAdapter`这个类是很好的参考。其基本思想就是注册Cursor的两种Observer，然后收到ContentObserver的通知后直接requery，然后触发Cursor的DatasetObserver，然后在onChanged里面notifyDatasetChanged.需要`CursorAdapter.FLAG_AUTO_REQUERY`这个flag，默认的构造函数：
```
public CursorAdapter(Context context, Cursor c) {
    init(context, c, FLAG_AUTO_REQUERY);
}
```
会默认使用这个flag。<br>
但是有个问题，这个构造函数已经deprecated了（因为触发的requery会运行在UI线程，有可能会导致卡顿）。想用的同学权衡一下利弊吧，或者直接CursorLoader也是没问题的。当然你可以不使用这个flag。但是这样就没有了自动更新了。

### 绝佳范例二 : CursorLoader
官方推荐与Cursor一起使用的，基于AsyncTask的异步加载器，没有使用Cursor.requery，而是收到ContentObserver.onChanged后直接重新查询一个新的Cursor出来。