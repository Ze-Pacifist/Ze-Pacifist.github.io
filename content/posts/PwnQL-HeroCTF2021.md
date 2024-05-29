+++
title = 'PwnQL - Hero CTF 2021'
date = 2021-04-26T11:14:36-04:00
draft = false
+++

**tl;dr**
+ Wildcard SQL injection
<!--more-->

This is a writeup for the PwnQL series of challenges in the web category of Hero CTF.

![](https://cdn-images-1.medium.com/max/2000/1*d5lWybc5B515iGR-YE4kmA.png)

## PwnQL #1

We are provided with a link and the description tells us to log in as admin. We do not have a password but do hackers really need one? (¬‿¬)

![](https://cdn-images-1.medium.com/max/2870/1*4LVvqUTZQK1aGU2DPkdC9g.png)

Random default passwords like “password” or “password123” does not seem to work(No idea why) but checking the source of the webpage, we are greeted with an intriguing message in the comments.

![](https://cdn-images-1.medium.com/max/2000/1*kEVkPR-78D_VwGD3Ulih6g.png)

This suggests the presence of a backup file which the admin forgot to remove and we can download it by navigating to /login.php.bak

```php
    <?php

    require_once(__DIR__  . "/config.php");

    if (isset($_POST['username']) && isset($_POST['password'])) {
        $username = $_POST['username'];
        $password = $_POST['password'];

    $sql = "**SELECT * FROM users WHERE username = :username AND password LIKE :password;**";
        $sth = $db->prepare($sql, array(PDO::ATTR_CURSOR => PDO::CURSOR_FWDONLY));
        $sth->execute(array(':username' => $username, ':password' => $password));
        $users = $sth->fetchAll();

    if (count($users) === 1) {
            $msg = 'Welcome back admin ! Here is your flag : ' . FLAG;
        } else {
            $msg = 'Wrong username or password.';
        }
    }
```

This filereveals that we would have to do an SQL injection attack to bypass the login. As we have the SQL query used to retrieve the username and password from the database, crafting a payload becomes much easier.

We already know that the username is “admin” so we have to find a way to bypass the password check. But instead of the usual “SELECT ” Operator, the server is using the “LIKE” SQL Operator to check the password.

The LIKE Operator, as given in [w3schools](https://www.w3schools.com/sql/sql_like.asp), is used in a WHERE clause to search for a specified pattern in a column. This operator has two wildcards which can be used with it:

* The percent sign (%) represents zero, one, or multiple characters

* The underscore sign (\_) represents one, single character.

Using admin as username and “%”*(which represents the entire password which we do not know)* as password, we are able to solve the first part of the challenge.

![](https://cdn-images-1.medium.com/max/3344/1*6wi2LzF81-WItZ3guCgVYA.png)
> username-admin
> password-%
> flag-Hero{pwnQL_b4sic_0ne_129835}

## PwnQL #2

![](https://cdn-images-1.medium.com/max/2000/1*i7Dkab_iwY7GIhJgzKJSyg.png)

Here, we are provided with the same login page but to get the flag, we have to extract the admin password. This was a pretty interesting challenge and if it was a normal SQL injection, it could be done using the “UNION SELECT” operator but in this case, the presence of wildcards- ‘%’ and ‘\_’ with the “LIKE” operator makes it easy for us.

As mentioned above, the ‘%’ wildcard can be used to denote multiple characters which is why we were able to bypass the login with just one ‘%’ sign. The ‘_’ only denotes single characters. So, if were to try just one ‘_’, we would not be able to bypass the login. But this behaviour is useful to us as we can find out the number of characters in the password.

Using username:admin and password:`__________`(10 ‘\_’), we are able to bypass login, indicating that the password has 10 characters.

### Extracting the password

We have to brute force each character to find the password and that can be done using two methods.

If, for example, the password was “password”, using “p_________”(p and 9 ‘_’) will bypass the password. So will using “p%”. So we can either choose to replace each ‘_’ with a character or keep adding characters before the ‘%’ sign(pa%, pass% etc.) till we bypass login and finally find the password.

I wrote the following python script to automate the process.

```py
    import requests
    from string import printable

    printable = printable.replace("%", '')

    url = "[http://chall1.heroctf.fr:8080/index.php](http://chall1.heroctf.fr:8080/index.php)"

    flag=''
    temp='%'

    def test(flag):
     payload={"username":"admin","password":flag+temp}
     r=requests.post(url, data=payload)
     print(f'\r{flag}', end='')
     return 'Here' in r.text

    while len(flag)<10:
     for i in printable:
      if test(flag+i):
       flag+=i
       break

    print(flag)
    #Hero{s3cur3p@ss}
```

This returns the following output

![](https://cdn-images-1.medium.com/max/2000/1*LVl2pTow-N_D5Fo3lflvMw.png)

We have successfully extracted the password- “s3cur3p@ss”
> Flag-Hero{s3cur3p@ss}

This was a very unique and awesome challenge. Props to the HeroCTF team for an amazing CTF.

If you would like to read more about SQL wildcards, I recommend the following article:

[https://www.w3schools.com/sql/sql_wildcards.asp](https://www.w3schools.com/sql/sql_wildcards.asp)

Thanks for reading!!