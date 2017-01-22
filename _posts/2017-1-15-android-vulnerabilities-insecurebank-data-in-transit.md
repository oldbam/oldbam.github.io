---
layout: default
title: Identifying and Fixing Security Vulnerabilities of Android Applications - Protecting Data in Transit
permalink: /android/security/:title
---

### Prerequisites

* [InsecureBankv2](https://github.com/dineshshetty/Android-InsecureBankv2) installed on Android device or emulator
* Server part of the [InsecureBankv2](https://github.com/dineshshetty/Android-InsecureBankv2) is running on the host computer
* HTTP proxy such as [Fiddler](http://www.telerik.com/fiddler) set up on the host computer
* WiFi on an android device or emulator is configured to use HTTP proxy 

### Getting information

In order to spy on network communications between a device and a server component, we need to set up a local HTTP proxy and route our web traffic via this proxy. There are many articles on the web which explain how to do this, I can recommended one that shows how to do that with [Fiddler](http://www.telerik.com/fiddler) that's available at their [documentation page](http://docs.telerik.com/fiddler/Configure-Fiddler/Tasks/ConfigureForAndroid).

Once the proxy is up and running, we can try to login and observe what requests does the application generate. As you can see, client application sends request over HTTP to `/login` endpoint and includes password in plaintext, thus making it available to any observer on the network: 

![Request to login endpoint from InsecureBank application is being sent over HTTP with password in plaintext](/images/android-login-request-as-seen-in-Fiddler.PNG)

Note that `%40` and `%21` are just URL-encoded values of `@` and `!` respectively. You can use Text Wizard in Fiddler (Tools -> TextWizard...) to convert values between various encodings: 

![Using Text Wizard in Fiddler to decode URL-encoded data](/images/android-fiddler-text-wizard.PNG)

After we log in, we can execute additional requests by trying different functionality of our app. Let's try to transfer money. An interesting observation one can make when looking at the captured traffic is that password was still passed from the device to the server: 

![Request to get accounts endpoint from InsecureBank application contains user's password](/images/android-transfer-request-as-seen-in-Fiddler.PNG)

However, we did not specify password anywhere, which means that an application stores it on the device somewhere. It is also surprising that it's not the hash of the password being stored, but full password. In one of the subsequent posts, we will research this issue further.

If we try to change password, we will see that application makes it quite difficult to select a strong password. However, after we capture a request in Fiddler, we may try to reissue it and set password to a dictionary word _hello_. No validation is applied on the server as to the strength of the password, and request succeeds: 

![Request to change password endpoint as seen in Fiddler](/images/android-change-password-request-as-seen-in-fiddler.PNG)

You may as well try to change the username, and set password for somebody else. 

Please, note, that it was not our goal to do any analysis of the server-side part of application. However, when evaluating mobile applications it is also necessary to look at the server-side API which may have its own vulnerabilities.

### Fix

One should always remember that it is possible to capture HTTP communication between a mobile device and server-side endpoints. Proper care should be taken to secure data in transit by using [Transport Layer Security](https://en.wikipedia.org/wiki/Transport_Layer_Security). Applications which work with sensitive data would also employ additional safeguards by utilizing [Certificate Pinning](https://en.wikipedia.org/wiki/HTTP_Public_Key_Pinning). If an app used certificate pinning correctly, we would not be able to [decrypt HTTPS traffic](https://www.fiddlerbook.com/fiddler/help/httpsdecryption.asp) using HTTP proxy such as [Fiddler](http://www.telerik.com/fiddler).

### Summary

In this post we looked at intercepting HTTP traffic from Android application InsecureBank. We found several vulnerabilities, most notable of which was the absence of Transport Layer Security.
