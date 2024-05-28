+++
title = 'Memo Drive - Linectf 2022'
date = 2022-03-29 16:55:28-04:00
draft = false
tags = ["web","lfi","notes"]
+++

**tl;dr**
+ url parse bug
<!--more-->

simple memo tool to write and store memos.

look at source code.

#uses starlette - lightweight toolkit ideal for building asychronous web services in python.

index, view, reset and save functionality. show /view with memo example and maybe LFI.

then looking at the source code.

/view endpoint

checks for & and . in both url query and the client id parameter.

we can try url encoding but since unquote is there that wont work.

vulnerablity in way the **path to file is constructed.**

join request.query_params.keys joins all the get parameters without the values but remember & is not allowed.

bug in how parameters are separated in starlette [https://github.com/encode/starlette/issues/1325](https://github.com/encode/starlette/issues/1325)

which actually stems from bug in urllib [https://python-security.readthedocs.io/vuln/urllib-query-string-semicolon-separator.html](https://python-security.readthedocs.io/vuln/urllib-query-string-semicolon-separator.html)

so we can use ; instead of & to separate the url parameters.

show log with using semicolon

so we can escape the unquote so url encoded dots will work.

`/view?2987bf4c72b6ade55901d57df14810f7=flag;/%2e%2e/`

replace with correct client it.

unintended:

2 uninteneded solutions both again based on how url is parsed.

1. 

the request.url url is derived frm host header.

[https://github.com/encode/starlette/discussions/1557](https://github.com/encode/starlette/discussions/1557)

the # renders request.url.query empty but request.query_params is not so we can do the same thing we did earlier without using the semicolon method.

1. 

hashtag can also be used in the parameter value so that filters dont work on the remaining part

final payload:

```html
/view?2987bf4c72b6ade55901d57df14810f7=#&2987bf4c72b6ade55901d57df14810f7=flag&/../
```