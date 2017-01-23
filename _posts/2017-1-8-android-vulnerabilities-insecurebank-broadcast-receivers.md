---
layout: default
title: Identifying and Fixing Security Vulnerabilities of Android Applications - Broadcast Receivers
description: Identify and fix security vulnerabilities in Android broadcast receiver implementations
permalink: /android/security/:title
---

### Prerequisites 

* [InsecureBankv2](https://github.com/dineshshetty/Android-InsecureBankv2) installed on Android device or emulator
* [Drozer](https://labs.mwrinfosecurity.com/tools/drozer) agent installed on Android device or emulator
* Drozer console runs and connected to drozer agent

Optional:

* [Bytecode Viewer](https://github.com/Konloch/bytecode-viewer) installed on a host machine

### Getting information

In the [previous post](/android/security/android-vulnerabilities-insecurebank-activities) we analyzed exported Activities. In this post we will look at exported Broadcast Receiver. Let's use drozer to check which receivers we have in our application:

```
dz> run app.broadcast.info -a com.android.insecurebankv2 -i
Package: com.android.insecurebankv2
  com.android.insecurebankv2.MyBroadCastReceiver
    Intent Filter:
      Actions:
        - theBroadcast
    Permission: null
```

As you see, `MyBroadCastReceiver` processes actions with name `theBroadcast`, it is exported and not protected by a permission, meaning that any app can create an `Intent` which will result in this receiver being triggered. In order to determine what this receiver can do, we need to look at the source code. There are different ways how to do that, I will use [Bytecode Viewer](https://github.com/Konloch/bytecode-viewer) which is handy because it contains multiple decompilers and allows to compare output that they produce side-by-side. On the image below you can see how the source code looks like: 

![Source code of vulnerable Broadcast Receiver as seen in Bytecode Viewer](/images/android-insecurebank-receiver-source-code.PNG) 

If we look into the source code, we would see that two parameters are being retrieved from the `Intent`:

```
String str1 = paramIntent.getStringExtra("phonenumber");
String str2 = paramIntent.getStringExtra("newpass");
```

Then, the code reads data stored in Shared Preferences, does some cryptographic operations, and at the end calls `SmsManager.sendTextMessage()`. 

### Exploit

Let's issue a drozer command to try to trigger our Broadcast Receiver:

```
dz> run app.broadcast.send --action theBroadcast --extra string phonenumber 12345 --extra string newpass A1!B2!C3!
```

If we look at our Android device now, we will see that we are about to send an sms message. Setting a premium rate sms number and forcing users to send messages without their consent is one of the ways bad guys can be making money: 

![Sending sms message after triggering Broadcast Receiver](/images/android-insecurebank-broadcast-receiver-exploited.PNG)

### Fix

Obviously, the need to export Broadcast receiver in this case is probably negligible. However, imagine that you _do_ have a need to export it. A naive approach one might take is to introduce a custom permission and protect your receiver by this permission. Below are the changes that one would need to introduce to `AndroidManifest.xml`: 

![Code changes to create new permission](/images/android-code-changes-create-new-permission.PNG) 

![Code changes to protect broadcast receiver with a custom permission](/images/android-code-changes-protect-broadcast-receiver-permission.PNG)

After these changes, if we try to trigger an action on `MyBroadCastReceiver` using drozer, the following error may be observed in logcat: 

![Starting an action of broadcast receiver fails due to insufficient permissions](/images/android-logcat-message-start-broadcast-permission-denied.PNG). 

Unfortunately, the above approach does not help much. The only thing that is required from an attacker to be able to execute this action is to add the following string to his application's manifest:

```
<uses-permission android:name="com.android.insecurebankv2.MyBroadCastReceiverPermission" />
```

There are many users out there who can be tricked into installing applications that request more permissions than they actually need. If you are using drozer, you may even rebuild agent apk with any permission by issuing the following command:

```
drozer agent build -p com.android.insecurebankv2.MyBroadCastReceiverPermission
```

If we reinstall agent on the Android device and try to get information about it using drozer, you will see the following:

```
dz> run app.package.info -a com.mwr.dz
Package: com.mwr.dz
  Application Label: drozer Agent
  Process Name: com.mwr.dz
  Version: 2.3.3
  Data Directory: /data/data/com.mwr.dz
  APK Path: /data/app/com.mwr.dz-1.apk
  UID: 10131
  GID: [3003]
  Shared Libraries: null
  Shared User ID: null
  Uses Permissions:
  - android.permission.INTERNET
  - com.android.insecurebankv2.MyBroadCastReceiverPermission
  Defines Permissions:
  - None
```

We can now successfully execute an action of broadcast receiver. If you observe logcat, you will see the following:

![Successfully triggering an action on broadcast receiver from updated drozer agent](/images/android-successfully-executing-action-broadcast-receiver.PNG)

Note: As you see on the above image, the information that application logs is quite extensive and includes both an old and new passwords. Prior to Android version 4.1, third-party applications could request `android.permission.READ_LOGS` and use information available in the log. As of now, only system applications can use this permission.

So, what would be the better way to protect our broadcast receiver? Well, if we are sure that we want this action to be triggered out of applications that we control, we can use _signature_ level permission. In this case, only applications that we signed with the same key will be able to obtain this permission. Here are the changes that we have to introduce to `AndroidManifest.xml`: 

![Code changes to protect broadcast receiver with a signature permission](/images/android-code-changes-protect-broadcast-receiver-signature-permission.PNG)

After making the changes and reinstalling InsecureBank application on the device, try to trigger an action on our broadcast receiver will result in permission denial.

### Summary

In this post we looked at exploiting unsecured broadcast receivers. We also made several changes to protect broadcast receiver with custom permission.
