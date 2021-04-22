---
title: Android系统开发-APK安装包名或签名信息认证
date: 2021-03-25 11:23:03
categories: 
- Android系统
tags:
- PMS
- 系统
---

最近在做定制系统，完成了一个功能，安装应用的时候要对APK的包名或签名进行双重认证。

当APK安装的时候首先判断签名信息SHA-1是否和我们预设的相同，如果不同，表示这个APK不是我们签名的应用，然后再判断这个应用的包名，如果这个包名不在我们预设的包名列表中，就可以确认这个APK为非法安装，返回安装失败信息。

## 数据库和Provider

关于这个包名列表，我们是将数据存储到数据库中，在SettingsProvide中增加自己的数据库，并使用ContentProvider向外提供数据。

`frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/MyDatabaseHelper`

DatabaseHelper类，创建数据库

```java
package com.android.providers.settings;

import android.content.Context;
import android.database.Cursor;
import android.database.sqlite.SQLiteDatabase;
import android.database.sqlite.SQLiteOpenHelper;

public class MyDatabaseHelper extends SQLiteOpenHelper {
    public static final String DATABASE_NAME = "My.db";
    public static final int DATABASE_VERSION = 1;
    public static final String TABLE_NAME = "package_name";
    private static MyDatabaseHelper instance = null;

    public static MyDatabaseHelper getInstance(Context context) {
        if (instance == null) {
            instance = new MyDatabaseHelper(context);
        }
        return instance;
    }

    public MyDatabaseHelper(Context context) {
        super(context, DATABASE_NAME, null, DATABASE_VERSION);
    }

    @Override
    public void onCreate(SQLiteDatabase db) {
        String creatTable = "create table " + TABLE_NAME + "(_id integer PRIMARY KEY AUTOINCREMENT, packageName varchar(40))";
        db.execSQL(creatTable);
    }

    @Override
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {
    }

}

```

然后要将数据库通过ContentProvider向外提供

```java

package com.android.providers.settings;

import android.content.ContentProvider;
import android.content.ContentUris;
import android.content.ContentValues;
import android.content.UriMatcher;
import android.database.Cursor;
import android.database.sqlite.SQLiteDatabase;
import android.database.sqlite.SQLiteQueryBuilder;
import android.net.Uri;

public class MyProvider extends ContentProvider {

    private static final UriMatcher sUriMatcher;
    private static final int MATCH_PACKAGE_NAME = 1;
    public static final String AUTHORITY = "My";
    public static final Uri CONTENT_URI_FIRST = Uri.parse("content://" + AUTHORITY + "/package_name");

    static {
        sUriMatcher = new UriMatcher(UriMatcher.NO_MATCH);
        sUriMatcher.addURI(AUTHORITY, "package_name", MATCH_PACKAGE_NAME);
    }

    private MyDatabaseHelper mDbHelper;

    @Override
    public boolean onCreate() {
        //创建数据库
        mDbHelper = MyDatabaseHelper.getInstance(getContext());
        return false;
    }

    @Override
    public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) {
        SQLiteQueryBuilder queryBuilder = new SQLiteQueryBuilder();
        switch (sUriMatcher.match(uri)) {
            case MATCH_PACKAGE_NAME:
                queryBuilder.setTables(MyDatabaseHelper.TABLE_NAME);
                break;
            default:
                throw new IllegalArgumentException("Unknow URI: " + uri);
        }
        SQLiteDatabase db = mDbHelper.getReadableDatabase();
        Cursor cursor = queryBuilder.query(db, projection, selection, selectionArgs, null, null, null);
        return cursor;
    }

    @Override
    public String getType(Uri uri) {
        return null;
    }

    @Override
    public Uri insert(Uri uri, ContentValues contentValues) {
        SQLiteDatabase db = mDbHelper.getWritableDatabase();
        switch (sUriMatcher.match(uri)) {
            case MATCH_PACKAGE_NAME: {
                long rowID = db.insert(MyDatabaseHelper.TABLE_NAME, null, contentValues);
                if (rowID > 0) {
                    Uri retUri = ContentUris.withAppendedId(CONTENT_URI_FIRST, rowID);
                    return retUri;
                }
            }
						    break;
            default:
                throw new IllegalArgumentException("Unknown URI " + uri);
        }
        return null;
    }

    @Override
    public int delete(Uri uri, String selection, String[] selectionArgs) {
        SQLiteDatabase db = mDbHelper.getWritableDatabase();
        int count = 0;
        switch(sUriMatcher.match(uri)) {
            case MATCH_PACKAGE_NAME:
                count = db.delete(MyDatabaseHelper.TABLE_NAME, selection, selectionArgs);
                break;
            default:
                throw new IllegalArgumentException("Unknow URI :" + uri);
        }
        return count;
    }

    @Override
    public int update(Uri uri, ContentValues values, String selection, String[] selectionArgs) {
        SQLiteDatabase db = mDbHelper.getWritableDatabase();
        int count = 0;
        switch(sUriMatcher.match(uri)) {
            case MATCH_PACKAGE_NAME:
                count = db.update(MyDatabaseHelper.TABLE_NAME, values, selection, selectionArgs);
                break;
            default:
                throw new IllegalArgumentException("Unknow URI : " + uri);
        }
        this.getContext().getContentResolver().notifyChange(uri, null);
        return count;
    }
}

```

最后将provider注册到manifest文件

```xml
<provider android:name="MyProvider"
                  android:authorities="My"
                  android:multiprocess="false"
                  android:exported="true"
                  android:singleUser="true"
                  android:initOrder="100" />
```

然后就可以通过ContentProvider向数据库插入或查询包名列表

```java
public void insertPMSVerifyPackageName(List<String> verifyPackageNameList){
		ContentResolver resolver = mContext.getContentResolver();
		resolver.delete(Uri.parse("content://My/package_name"), null, null);
		for (int i = 0; i < verifyPackageNameList.size(); i++) {
		    ContentValues values = new ContentValues();
		    values.put("_id", i);
		    values.put("packageName", verifyPackageNameList.get(i));
		    resolver.insert(Uri.parse("content://My/package_name"), values);
		}
	}
```

## 签名 SHA-1

对于应用的签名认证，我们使用签名信息的SHA-1值来做签名判断，当APK解析后的SHA-1和我们预设的值相同，就表明应用签名认证通过。

预设的SHA-1值使用SystemProperties设置。

```java
public void setSignatureSHA1(String sha1){
      SystemProperties.set(PROPERTIES_MY_SIGNATURE,sha1);
}
```

## 安装认证

应用安装会在PMS服务中进行解析，我们就在installPackageLI函数中对APK进行认证。

`frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java`

```java
private void installPackageLI(InstallArgs args, PackageInstalledInfo res) {
		......
		String customSignature = SystemProperties.get(PROPERTIES_MY_SIGNATURE, DEFALT_SHA1);//默认SHA-1
    if (!TextUtils.isEmpty(customSignature)) {
    	final Signature[] signatures = pkg.mSignatures;
    	//这里只拿了apk的第一个签名，有可能APK会有双重签名的情况，这里可以改成循环判断
    	if (!getFingerprint(signatures[0], "SHA-1").equals(customSignature)) {
    			if(!fetchAllContacts().contains(pkgName)){
    					res.setError(PackageManager.INSTALL_FAILED_VERIFICATION_FAILURE,"Signature verification failed");
    					return;
    			}
    		}
    }
		......
}

//获取签名信息的SHA-1
private static String getFingerprint(Signature signature, String hashAlgorithm) {
        if (signature == null) {
            return null;
        }
        try {
            MessageDigest digest = MessageDigest.getInstance(hashAlgorithm);
            return toHexadecimalString(digest.digest(signature.toByteArray()));
        } catch (NoSuchAlgorithmException e) {
                    // ignore
        }
        return null;
}

//查询数据库中预设的包名
private List<String> fetchAllContacts() {
        List<String> pkgList = new ArrayList<>();
        ContentResolver contentResolver = mContext.getContentResolver();
        Cursor cursor = contentResolver.query(Uri.parse("content://My/package_name"),null, null, null, null);
        cursor.getCount();
        while(cursor.moveToNext()) {
            Log.e(TAG,"pkg name:"+cursor.getString(1));
            pkgList.add(cursor.getString(1));
        }
        cursor.close();
        return pkgList;
    }
```

如果APK签名信息错误，并且它的包名又没在预设的包名列表中，应用就无法安装，返回INSTALL_FAILED_VERIFICATION_FAILURE信息。否则走正常安装流程。