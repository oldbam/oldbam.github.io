---
layout: default
title: Protecting Data at Rest in Android Applications
description: Learn Android Security. See how to protect data at rest by analyzing vulnerable Android application
permalink: /android/security/:title
last_modified_at: 2017-2-6
---

{% include android-app-security-series-intro.html %}

### Prerequisites

* [InsecureBankv2](https://github.com/dineshshetty/Android-InsecureBankv2) installed on Android device or emulator
* `adb` client connected to a device

Optional:

* [Bytecode Viewer](https://github.com/Konloch/bytecode-viewer) installed on any machine

### Getting information

Protecting data at rest is important for applications that deal with sensitive data, like bank application we are working with. For these types of applications, client (i.e. device) should be considered untrusted. It may happen that a malicious application is installed on the device and, after obtaining root access, is able to get access to files belonging to other applications. Because of that, your application should not store any sensitive data on the device.

Let's navigate inside the application directory, and see all files that application works with.

```
root@donatello:/data/data/com.android.insecurebankv2 # ls -lR

.:
drwxrwx--x u0_a73   u0_a73            2016-12-02 16:25 app_webview
drwxrwx--x u0_a73   u0_a73            2016-12-02 16:24 cache
drwxrwx--x u0_a73   u0_a73            2016-12-02 16:24 databases
lrwxrwxrwx install  install           2016-12-19 16:17 lib -> /data/app-lib/com.android.insecurebankv2-1
drwxrwx--x u0_a73   u0_a73            2016-12-02 19:09 shared_prefs

./app_webview:
-rw------- u0_a73   u0_a73      38912 2016-12-02 16:25 Web Data
-rw------- u0_a73   u0_a73        512 2016-12-02 16:25 Web Data-journal

./cache:

./databases:
-rw-rw---- u0_a73   u0_a73      20480 2016-12-02 19:09 mydb
-rw------- u0_a73   u0_a73      12824 2016-12-02 19:09 mydb-journal

./shared_prefs:
-rw-rw---- u0_a73   u0_a73        165 2016-12-02 16:24 com.android.insecurebankv2_preferences.xml
-rw-rw---- u0_a73   u0_a73        209 2016-12-02 19:09 mySharedPreferences.xml
```

Inspecting SQLite databases does not show any potential vulnerabilities:

```
cd /data/data/com.android.insecurebankv2/
sqlite3 mydb
sqlite> .tables
android_metadata  names
sqlite> select * from android_metadata;
en_US
sqlite> select * from names;
1|jack
```

Let's see what we have stored in shared preferences:

```
cd /data/data/com.android.insecurebankv2/shared_prefs
cat com.android.insecurebankv2_preferences.xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <string name="serverport">8888</string>
    <string name="serverip">192.168.0.1</string>
</map>

cat mySharedPreferences.xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <string name="EncryptedUsername">amFjaw==
    </string>
    <string name="superSecurePassword">nH/HGFyUPFtmqK/gHS4Adw==
    </string>
</map>
```

While the first file just keeps settings for our connection to the server part of the application, second file reveals some interesting information. When you look at the username, even though the key name says _encrypted_, based on the `==` in the suffix, you may realize that this looks like base64 encoding. It is trivial to obtain unencoded base64 value, just search for some online decoder, and you will get a value `jack`. However, when we try the same with _superSecurePassword_, we get no luck, as the result does not seem to contain readable characters that would indicate a password.

It looks like we would need to look into the source code to understand if we can decrypt _superSecurePassword_. Let's open [Bytecode Viewer](https://github.com/Konloch/bytecode-viewer) and search for all classes that use _superSecurePassword_: 

![Searching for all classes that use password values stored in shared preferences in Bytecode Viewer](/images/android-search-decompiled-code-super-secure-password.PNG)

If we open LoginActivity.fillData method, we will see that we retrieve encrypted password and use `CryptoClass` to decrypt it:
```
final String string2 = sharedPreferences.getString("superSecurePassword", (String)null);
this.Password_Text.setText((CharSequence)new CryptoClass().aesDeccryptedString(string2));
```

So, we need to look inside `CryptoClass`: 

![Source code of CryptoClass as seen in Bytecode Viewer](/images/android-source-code-cryptoclass.PNG)

OK, from the decompiled code we can see that symmetric key is hardcoded into source code and initialization vector is also present nearby and set to all zeros. 

### Exploit

Since we have the decompiled source code, we are in a possession of code that can decrypt stored data. Just look into `aesDeccryptedString` and `aes256decrypt` methods from `CryptoClass`.

### Fix

For applications that deal with highly sensitive data, you should consider your client as being untrusted. This means that you should _not_ store any sensitive data on the client. In our case, it would be that we would not store password on the device in shared preferences. The fact that we store full password, not merely a hash, is also concerning. However, imagine a scenario where one _does_ need to store sensitive data on a device. What to do in this case? 

A potential solution is to use Password Based Encryption when you generate a key for symmetric encryption from a user provided data. One might take the following approach:

* When an application is installed, generate a random value which will be later used as a salt for password-based key derivation function
* Ask user for password when the value has to be encrypted
* Use Key Derivation Function with large number of iterations and sufficient salt length to generate the key. If I were to determine the current recommended values for which function to use for password-based key derivation, I would probably check with U.S. National Institute of Standards and Technology - NIST, you can look [here](http://dx.doi.org/10.6028/NIST.SP.800-132) and [here](https://pages.nist.gov/800-63-3/). At the time of writing the recommendations are [PBKDF2](https://en.wikipedia.org/wiki/PBKDF2) with salt value of 32 bits or more, at least 10000 iterations of hash function. Key length was recommended to be 112 bits as of December 2010, however, at least twice as large value is probably warranted. Again, you should check for updated values yourself.
* Use the generated key to encrypt password with symmetric encryption. AES-256 with Cipher Block Chaining mode (CBC) is considered fine at the time of writing (use [NIST](https://www.nist.gov/) recommendations for up-to-date values).

Another option to having user specifying password for key derivation is to store data for key derivation outside of the device, i.e. on the server side.

Please, note that our view of locally stored data would be incomplete if we didn't look at /sdcard. It is worth remembering that all data stored on SD card is world-readable, so, if your application _does_ store any data there, make sure that it is not sensitive. As for the InsecureBank application, we will take a look at SD card storage when we analyze Web Views in one of the other posts.

### Summary

In this post we looked at the data that was being stored on the device. We found that user's password was encrypted with a key hard-coded into application and stored in user preferences file, which means that it can be easily obtained by a malicious application which has root access on the device.
