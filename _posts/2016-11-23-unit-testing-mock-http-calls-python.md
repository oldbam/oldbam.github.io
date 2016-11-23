---
layout: default
title: How to Mock HTTP Calls for Unit Testing in Python
---

# How to Mock HTTP Calls for Unit Testing in Python

This post shows how one can mock HTTP calls for unit testing in python without any additional library. In this post we presume that HTTP communication is done using python [requests](http://docs.python-requests.org/en/master/) library.

### Simple GET request to download JSON

Imagine that our application issues a GET request to download latest comics metadata from xkcd.com. To do that, we need to be able to execute a request to http://xkcd.com/info.0.json. Here is an example code that does this:

```python
class XKCDDownloader:
    XKCD_LATEST_COMIC_URL = "http://xkcd.com/info.0.json"

    def download_latest_comic_metadata(self):
        r = requests.get(self.XKCD_LATEST_COMIC_URL)
        r.raise_for_status()
        return r.json()
```

In order to mock this class, one can safe the JSON returned from the xkcd server to a file and use the following code:

```
class MockedXKCDDownloader(XKCDDownloader):
    def download_latest_comic_metadata(self):
        with open(Util.get_absolute_file_path("TestData\\xkcd_latest_metadata.json")) as file:
            return json.load(file)
```

Any class that uses `MockedXKCDDownloader` will get the contents of stored JSON data when it calls `download_latest_comic_metadata`.
