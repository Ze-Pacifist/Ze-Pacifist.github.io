+++
title = 'The Mission - NahamCon 2025 CTF'
date = 2025-05-25T13:00:08+05:30
draft = false
tags = ["web", "graphql", "api"]
+++


**tl;dr**

+ Writeup for all challenges from "The Mission" - NahamCon 2025 CTF
+ Flag1 - Found in `robots.txt`
+ Flag2 - Java SpringBoot API actuator + WAF bypass using URL encoding
+ Flag3 - Use actuator heapdump to reconstruct Authorization header to get token for `/internal-dash`
+ Flag4 - Graphql introspection to reveal Queries -> use `users` query list all users id, email and usernames
+ Flag5 - Path traversal in internal-dash api's to reveal internal api's and change status of reports
+ Flag6 - Convince the chatbot to reveal the flag.

<!--more-->

## Introduction
"The Mission" was a series of challenges(or Challenge Group as they called it) provided by [HackingHub](https://hackinghub.io/) as part of NahamCon 2025 CTF. The target is a vulnerable BugBounty platform which players have to hack into and retrieve multiple flags.

## Challenge Description
Welcome to the BugBounty platform, you've submitted a report for Yahoo but oh no!! So has STÖK and it looks like you're going to get dupped!

Hack into the platform and change STÖK's report to duplicate first so you can grab the bounty!

You can login with the follow details:
Username: hacker
Password: password123

## Analysis
We are given a single url to start off with along with a `wordlist.txt`, which I sadly did not notice initially during the CTF and caused me a bit of pain, but i digress. Logging in with the provided credentials give us access to the following endpoints:
```md
/hackers - List of a few well known hackers + profile pictures from /uploads/<img>.jpg
/dashboard - List of our reports which is currently just one.
/api/v2/reports - The api from which reports are retrieved, given a `user_id`
/settings - User settings page
/api/v2/graphql - Graphql api which is called by the settings page to return username and email.
```
Apart from this, every page also has a chat icon which sends requests to `/api/v2/chat`

Note - This writeup is not in the "correct" order of flags but rather the order in which I went about solving the challenges. 

## Flag #6 (Bonus)
Checking out the chatbot given, it reveals to us that secrets will only be spilled to "Adam Langley".
![chatflag1](image.png)

With very easy convincing, we get a flag!
![chatflag2](image-1.png)
> Flag = flag_6{9c2001f18f3b997187c9eb6d8c96ba60} 

## Flag #4
Moving our attention to the graphql endpoint, lets take a look at the query that is being sent to the api
```json
{"query":"\n        query GetUser($id: ID!) {\n          user(id: $id) {\n            username\n            email\n          }\n        }\n        ","variables":{"id":"fd55a401-b110-4821-9155-add4653cb992"}}
```
This retrieves the username and email using the `user` Query with a variable `id` which is the user id. Lets take a look at the other Queries that are available through graphql introspection.
![foundusers](image-2.png)
```json
{"query": "query { __schema {  types {   name  fields { name  }} }}"}
```
Using this payload we can find out that `user` which takes id parameter and `users` which takes no parameter are available.
Simply querying users for id, username and email gives us the next flag.
```json
{"query": "query { users{ id \n username \n email } }"}
```
![flag4found](image-3.png)
> Flag = flag_4{253a82878df615bb9ee32e573dc69634}

Since we have the user_id's of all users now, we can use this in conjunction with the `/api/v2/reports?user_id=` endpoint to retreive the report_id's of all the users submitted reports. This information might turn out to be useful in the future.
I also ran the full graphql inrospection queries provided by Burp and it revealed that there were no mutations or other useful stuff and the only data we can retrieve is username, id and email through `user` and `users`.

## Flag #1
After playing around with the given graphql endpoint and getting no leads, we can start with a little more broad enumeration. Even though the given wordlist did not contain `robots.txt`, it's something that is present in `dirb/common.txt` which is a small wordlist I commonly use for minimal enumeration.

![robots.txtflag](image-4.png)
>Flag = flag_1{858c82dc956f35dd1a30c4d47bcb57fb}

robots.txt also reveals another endopint `/internal-dash` which has a login page where the provided credentials did not work, neither did any login bypass attempts through SQLi, noSQLi, password guessing etc.

## Flag #2
Lets take a closer look at the `/api` endpoint. Sending a request to [http://challenge.nahamcon.com:30224/api/](http://challenge.nahamcon.com:30224/api/) returns json response:
```json
{"server":"openjdk:19-jdk:bountyapi.war","message":"BugBountyPlatform API"}
```
This indicates that the api server is running a Java application. `/api/v2` give the same response while `/api/v1` returns "Deprecated...".
At this point, we can try fuzzing the API with the given wordlist on both versions of api and look for responses with status code `200` to indicate success or `400` to indicate missing fields.
![fuzzapi](image-5.png)
Sadly this does not return much so lets turn our attention to the API server.

We can see that the web server is running nginx but the API returns "openjdk:19-jdk:bountyapi.war" which means all requests to `/api` are processed by the bountyapi.war file. Searching for "Java api default endpoints" leads us to [https://docs.spring.io/spring-boot/docs/2.1.0.RELEASE/reference/html/production-ready-endpoints.html](https://docs.spring.io/spring-boot/docs/2.1.0.RELEASE/reference/html/production-ready-endpoints.html) which describes `actuator`. It is also listed on [HackTricks](https://book.hacktricks.wiki/en/network-services-pentesting/pentesting-web/spring-actuators.html).

Lets see if our server has exposed actuator. `/api/v2/actuator` does not exist but `/api/v1/actuator` triggers a WAF
![actuatorfound](image-6.png)
Trying out ways to bypass WAF, simply URL encoding the characters of `actuator` works and we can get the next flag
```html
http://challenge.nahamcon.com:30224/api/v1/%61%63%74%75%61%74%6f%72
```
![flag2found](image-7.png)
> Flag = flag_2{a67796e1232c71f5a37177550a98a054}

## Flag #3
The `actuator` api endpoint also directs us towards `/api/v1/actuator/heapdump`
![heapdump](image-8.png)
From the heapdump, we can piece together the following request
```html
post /api/v1/internal-dashboard/token HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImludGkifQ.YeqvfQ7L25ohhwBE5Tpmqo2_5MhqyOCXE7T9bG895Uk
Host: internal-testing-apps
Content-Type: application/json
Connection: keep-alive

{"username":"inti"}
```
This gives us a new api endopint along with the Authorization token used by user "inti"
![internaldashboardtokencreated](image-9.png)
Sending the request with correct Authorization headers gives us a "token" which is for "Internal dashboard". We know from the robots.txt file that was found earlier that the internal endpoint is /internal-dash. Currently we have a token for the /internal-dash endpoint but no idea where to use the token. Once again, we can try fuzzing /internal-dash/FUZZ with the provided wordlist and it turns up empty. Trying it with dirb/common.txt reveals 2 paths - `/login` and `/logout`
![fuzzing internal dash](image-10.png)

Sending a request to `/logout` reveals a cookie `int-token` which aligns with the name of the token that we recieved from the api.
![get cookie name](image-11.png)

We can use the int-token Cookie with the generated token to access `/internal-dash`
```
Cookie: int-token=a1c2860d05f004f9ac6b0626277b1c36e0d30d66bb168f0a56a53ce12f3f0f7a
```
![flag3](image-12.png)
> Flag = flag_3{324671450653c00ae981fd9e15f8e842}

## Flag #5
Now that we have access to the internal dashboard, lets take a look at the functionality offered there.

There is an `/internal-dash/api/report` endpoint which takes a post request with report id. Since we have all user_id's and can get the report_id from `/api/v2/reports?user_id=`, lets see what data is being returned for STÖK's report which is our target as per challenge description.
![nopermissiontosee](image-13.png)
We can try path traversal attacks on the `id` parameter and it returns a weird error `"error":"Unknown Endpoint"`.
Sending an empty id or "." in id returns `{"reports":[]}` reports list with 0 entries. Sending ".." in id reveals a new list of endpoints.
![internal api endpoints](image-14.png)

`../my-reports` returns empty but `../search` returns `{"error":"Missing query string parameter"}`. Trying `?report_id`, `?report` etc didn't work but checking back with the provided wordlist, we can fuzz the query parameter name to find that `?q` returns a new error `{"error":"Report Not Found"}`. So now we can provide a report_id and see what happens.
![reportdetails](image-15.png)

We now have a new parameter - `change_hash` but sending that to `search` and trying to change status does not work. 

Lets go back to fuzzing the new `/internal-dash/api/report` endpoint.
```sh
ffuf -w ~/ctf/temp/nahamconwordlist.txt -u http://challenge.nahamcon.com:30224/internal-dash/api/report/FUZZ -X POST -mc 200,400
```
using ffuf to fuzz /internal-dash/api/report/FUZZ with POST request reveals a new endpoint `status`.

> Note - Another way to find the endpoint is to just read through the javascript at /internal-dash
```js
...
try {
    const response = await fetch('/internal-dash/api/report/status', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
        },
        body: JSON.stringify({
            id: reportId,
            status: newStatus,
            change_hash: currentChangeHash
        })
    }
...
```

Using this endpoint with the `change_hash` we got earlier, we are now able to successfully change the status of STÖK's report to duplicate.
![duplicated](image-16.png)
But going back to our dashboard, our YAHOO report is still PENDING. lets change that to ACCEPTED using the same steps as above.

First we get a `change_hash` for our report:
![get change_has](image-17.png)
Then we use that to change status at `/internal-dash/api/report/status`
![changed status of our report](image-18.png)

Finally, visit our dashboard to get the flag!.
![get last flag](image-19.png)
> Flag = flag_5{a3da8939cec2050b44ed1ec9ded8f4f3}
