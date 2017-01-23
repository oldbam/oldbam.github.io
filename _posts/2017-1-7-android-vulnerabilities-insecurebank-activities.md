---
layout: default
title: Identifying and Fixing Security Vulnerabilities of Android Applications - Activities
description: Identify and fix security vulnerabilities in Android activities implementations
permalink: /android/security/:title
---

### Prerequisites 

* [InsecureBankv2](https://github.com/dineshshetty/Android-InsecureBankv2) installed on Android device or emulator
* [Drozer](https://labs.mwrinfosecurity.com/tools/drozer) agent installed on Android device or emulator
* Drozer console runs and connected to drozer agent

### Getting information

In this blog series we will be using drozer extensively. [Drozer manual](https://labs.mwrinfosecurity.com/assets/BlogFiles/mwri-drozer-user-guide-2015-03-23.pdf) contains information how to get started, an important thing to remember is to allow port forwarding via adb and connect drozer console on your host computer to a drozer agent running on your android device or emulator:

```
adb forward tcp:31415 tcp:31415
drozer console connect
```

After we connect drozer console to an agent, we need to find package name of the InsecureBankv2 application. One can do this either by using `adb` or `drozer`:

```
adb shell
pm list packages | grep bank

// alternative way

dz> run app.package.list -f bank
```

This way or another, we now know that the package of interest is `com.android.insecurebankv2`.

One of the first things we can do with drozer is to determine attack surface:

```
dz> run app.package.attacksurface com.android.insecurebankv2
Attack Surface:
  5 activities exported
  1 broadcast receivers exported
  1 content providers exported
  0 services exported
    is debuggable
```

Let's look closer at activities. The command below will enumerated exported activities along with the permissions necessary to invoke them, i.e. activities that can be launched by other processes on Android device:

```
dz> run app.activity.info -a com.android.insecurebankv2
Package: com.android.insecurebankv2
  com.android.insecurebankv2.LoginActivity
    Permission: null
  com.android.insecurebankv2.PostLogin
    Permission: null
  com.android.insecurebankv2.DoTransfer
    Permission: null
  com.android.insecurebankv2.ViewStatement
    Permission: null
  com.android.insecurebankv2.ChangePassword
    Permission: null
```

As you can see, there are 5 exported activities. One can guess that `LoginActivity` should be exported, as it is probably the one launched when the application starts. However, other 4 activities may not necessarily be the one that should be allowed to be launched directly.

### Exploit

Here is a way how one can use drozer to launch an exported activity:

```
dz> run app.activity.start --component com.android.insecurebankv2 com.android.insecurebankv2.ChangePassword
```

On your android device/emulator you will see:

![Vulnerable ChangePassword Activity launched from drozer](/images/android-insecurebank-exported-activity-launched.PNG)

The screen you are seeing will allow to change password without authenticating to the application. In this specific case, another person should have had an access to the device to be able to change password. However, there are cases when parameters can be passed to the activities being launched, and those activities would operate on the given parameters. It is important to keep that in mind when evaluating real-world applications (looking into the source code of exported activities would be warranted to determine whether it reads any parameters from an intent that was used to launch it).

### Fix

The activity we looked at should not be exported, so the fix would be to remove `exported` attribute: 

![Fixing vulnerable activity by removing exported attribute](/images/android-insecurebank-exported-activity-fix.PNG). 

If we try to launch activity from drozer after this fix, we will get:

```
Permission Denial: starting Intent { flg=0x10000000 cmp=com.android.insecurebankv2/.PostLogin (has extras) } from ProcessRecord{ad574180 1878:com.mwr.dz:remote/u0a72} (pid=1878, uid=10072) not exported from uid 10071
```

### Summary

In this post we looked at exploiting exported Android activities. We also showed how to fix this vulnerability either by removing `exported` attribute.
