+++
title = 'Gitfile Explorer - Zer0pts 2022'
date = 2022-03-22 13:40:25
draft = false
tags = ["web","php","notes"]
+++

**tl;dr**
+ Bypass preg_match, read flag
<!--more-->



[http://gitfile.ctf.zer0pts.com:8001/](http://gitfile.ctf.zer0pts.com:8001/)

looks like a simple service which allows us to download files from github/gitlatb/bitbucket

preg match very weak only checks if github|gitlab|bitbucker anywhere in string.

passes result to php file_get_contents [https://www.php.net/manual/en/function.file-get-contents.php](https://www.php.net/manual/en/function.file-get-contents.php)

preg match can be bypassed easily....

only expects http(any characters)//(any characters with github somewhere).

http: can be used with filegetcontents to make http requests but since the “:” is not looked for in regex, we can not use the colon.

![Untitled](Gitfile%20Explorer%20ae340244ddd34781af7fd9c9c93856f7/Untitled.png)

^before

`http//githubrandomstuff.com/../../../../../../../../etc/passwd`
^after


now that we basically have directory traversal, we can edit the file=parameter to ../../../../../../../../../../flag.txt to get the flag.

[http://gitfile.ctf.zer0pts.com:8001/?service=https//github.com/&owner=ptr-yudai&repo=nautilus&branch=master&file=README/../../../../../../../../../../../../../flag.txt](http://gitfile.ctf.zer0pts.com:8001/?service=https//github.com/&owner=ptr-yudai&repo=nautilus&branch=master&file=README/../../../../../../../../../../../../../flag.txt)