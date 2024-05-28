+++
title = 'Required - HxpCTF 2022'
date = 2023-03-24 14:51:15
draft = false
tags = ["RE","nodejs","require","idontknowwhatimdoingplssendhelp"]
+++

**tl;dr**

+ nodejs require() shenanigans
+ prototype pollution hell

<!--more-->

**Challenge points**: 182
**No. of solves**: 46

## Challenge Description

I have written a super safe flag encryptor. I’m sure nobody can figure out what my original flag was:
0xd19ee193b461fd8d1452e7659acb1f47dc3ed445c8eb4ff191b1abfa7969


This is an analysis or at least an attempt to analyze how the cache or rather cache’s in node js require() function works under the context of the challenge [“required” from hxpCTF 2022](https://ctftime.org/task/24394)

Each JavaScript file is treated as a separate module in NodeJS. It uses commonJS module system : require(), exports and module.export. Every time a .js file is required, it is stored in the cache.

This is what you would end up reading when you search up about caching in nodejs but in reality it is much more complicated than this.

The given challenge contains a bunch of js files (approx 142) numbered between 1 to 1000 but with a lot of the numbers missing. Apart from that there is a “required.js” file which contains the core logic involved here. This file require()’s all the files including non-existent files. Calling require() on non-existent files should give a module not found exception but running this does not seem to produce any error. The reason as to why can be seen if we try opening up a few of the given existing files. Especially the files 28, 289…………………

Using the prototype pollution mentioned here([https://ctf.zeyu2001.com/2022/balsnctf-2022/2linenodejs](https://ctf.zeyu2001.com/2022/balsnctf-2022/2linenodejs)) we can basically “bind” one filename to another function. 

753 → 434

and then 556 deletes require.cache

![Untitled](Final%20required%200202624e16424d07ab2aff396cc16a35/Untitled.png)

require(’./753’) should not have worked but it does.

![Untitled](Final%20required%200202624e16424d07ab2aff396cc16a35/Untitled%201.png)

This means that caching in require is not as simple as it seems.

This led us to dive into the code to figure this out.

## Code analysis

[https://github.com/nodejs/node/blob/beb0520af74ed20c3d48a1b4f6ca8a89664976c6/lib/internal/modules/cjs/loader.js#L1023](https://github.com/nodejs/node/blob/beb0520af74ed20c3d48a1b4f6ca8a89664976c6/lib/internal/modules/cjs/loader.js#L1023)

```jsx
Module.prototype.require = function(id) {
  validateString(id, 'id');
  if (id === '') {
    throw new ERR_INVALID_ARG_VALUE('id', id,
                                    'must be a non-empty string');
  }
  requireDepth++;
  try {
    return Module._load(id, this, /* isMain */ false);
  } finally {
    requireDepth--;
  }
};
```

Here we can see the definition of the require function which calls Module._load. That is where the process begins.

[https://github.com/nodejs/node/blob/beb0520af74ed20c3d48a1b4f6ca8a89664976c6/lib/internal/modules/cjs/loader.js#L777](https://github.com/nodejs/node/blob/beb0520af74ed20c3d48a1b4f6ca8a89664976c6/lib/internal/modules/cjs/loader.js#L777)

Here, in `Module._load` we are introduced to two of the cache’s.

First one we see is `relativeResolveCache` which is a local cache defined here: [https://github.com/nodejs/node/blob/beb0520af74ed20c3d48a1b4f6ca8a89664976c6/lib/internal/modules/cjs/loader.js#L146](https://github.com/nodejs/node/blob/beb0520af74ed20c3d48a1b4f6ca8a89664976c6/lib/internal/modules/cjs/loader.js#L146)

The Second one is `Module._cache` which is the same as `require.cache`(and will be referred to require.cache from here on).

Initially a few checks are done on these cache’s(which we will come to later on) and if it is not cached, another function, `Module._resolveFilename()` is called. [https://github.com/nodejs/node/blob/beb0520af74ed20c3d48a1b4f6ca8a89664976c6/lib/internal/modules/cjs/loader.js#L874](https://github.com/nodejs/node/blob/beb0520af74ed20c3d48a1b4f6ca8a89664976c6/lib/internal/modules/cjs/loader.js#L874)

Here we see a new cache, `Module._pathCache`

Another thing to note is that here is where the function call to `trySelf` exists which is where the prototype pollution gadget exists. [https://github.com/nodejs/node/blob/beb0520af74ed20c3d48a1b4f6ca8a89664976c6/lib/internal/modules/cjs/loader.js#L942](https://github.com/nodejs/node/blob/beb0520af74ed20c3d48a1b4f6ca8a89664976c6/lib/internal/modules/cjs/loader.js#L942)

Again, checks are done on _pathCache and another function `Module._findPath()` is called. [https://github.com/nodejs/node/blob/beb0520af74ed20c3d48a1b4f6ca8a89664976c6/lib/internal/modules/cjs/loader.js#L511](https://github.com/nodejs/node/blob/beb0520af74ed20c3d48a1b4f6ca8a89664976c6/lib/internal/modules/cjs/loader.js#L511)

This is the function where the actual file from the file system is loaded but only if it does not exists in _pathCache.

### Cache collision’s

Now that we are introduced to all the major functions and cache’s involved here, lets take a look at how these interact with each other.

The most basic case is when a file is require()’d for the first time. It does not exists in any cache. `Module._load` calls `Module._resolveFilename` and finally ends up in `module._findPath()` 

```jsx
if (filename) {
      Module._pathCache[cacheKey] = filename;
      return filename;
    }
```

This not only returns the filename if it is successfully loaded in file system but also adds it to `_pathCache`. This is returned to [https://github.com/nodejs/node/blob/beb0520af74ed20c3d48a1b4f6ca8a89664976c6/lib/internal/modules/cjs/loader.js#L951](https://github.com/nodejs/node/blob/beb0520af74ed20c3d48a1b4f6ca8a89664976c6/lib/internal/modules/cjs/loader.js#L951) here which is then returned to [https://github.com/nodejs/node/blob/beb0520af74ed20c3d48a1b4f6ca8a89664976c6/lib/internal/modules/cjs/loader.js#L811](https://github.com/nodejs/node/blob/beb0520af74ed20c3d48a1b4f6ca8a89664976c6/lib/internal/modules/cjs/loader.js#L811). 

Here it is checked if “filename” is in require.cache. If not, it is `require.cache[filename]=module` is added and relativeResolveCache[relResolveCacheIdentifier]=filename. is also set.

```jsx
Module._cache[filename] = module;
  if (parent !== undefined) {
    relativeResolveCache[relResolveCacheIdentifier] = filename;
  }
```

It is important to note here that the key in require.cache and relativeResolveCache is different. 

---

Now, for the the case after a file has been loaded into cache: whenever `Module._load` is called, it first checks if the require()’d filename is in relativeResolveCache **and** if it is cached in require.cache.

[https://github.com/nodejs/node/blob/beb0520af74ed20c3d48a1b4f6ca8a89664976c6/lib/internal/modules/cjs/loader.js#L785](https://github.com/nodejs/node/blob/beb0520af74ed20c3d48a1b4f6ca8a89664976c6/lib/internal/modules/cjs/loader.js#L785).

If yes, the exported module stored in require.cache[filename] is returned from there.

(Note - filename is relativeResolveCache[relResolveCacheIdentifier])

If the require()’d filename is in relativeResolveCache but not in require.cache, it is deleted from relativeResolveCache and the rest of the code is executed.

---

Now onto the more complicated parts 🙂

With the prototype pollution mentioned earlier, it is possible to map one file to another. 

![Untitled](Final%20required%200202624e16424d07ab2aff396cc16a35/Untitled%202.png)

For the above example, assume 753 or 434 has not been added to any of the cache’s before.

When the file 28.js is called with first parameter as 753 and second parameter as 434 is called, 

prototype pollution object is created in 289.js and within 28.js there is a call to require(i) which is `require('./753')` .

`module._resolveFilename(753,...)` is called where the trySelf function [https://github.com/nodejs/node/blob/beb0520af74ed20c3d48a1b4f6ca8a89664976c6/lib/internal/modules/cjs/loader.js#L942](https://github.com/nodejs/node/blob/beb0520af74ed20c3d48a1b4f6ca8a89664976c6/lib/internal/modules/cjs/loader.js#L942) returns 434.

Since 434 is not in pathCache, `Module._findPath` is called which adds 753→434 to pathCache and returns filename which comes back to `Module._load` where `require.cache[filename]=module` and `relativeResolveCache[relResolveCacheIdentifier] = filename` is added.

If 753 exists in relativeResolveCache and 434 exists in require.cache, 753 cannot be further mapped to anything else.

But if 753 exists in relattiveResolveCache and 434 is not in require.cache, the mapping of 753 can be changed further and it overwrites _pathCache also. This situation arises especially when `require('./556')` is called which clears the entire require.cache due to which 434 is deleted.

Another corner case exists when lets say for example we clear require.cache and 434 no longer exists in require.cache but the mapping 753→434 exists in relativeResolveCache.

A call to `require('./28')(227,434,790)`  adds 227->434 to relativeResolveCache and pathCache and also adds 434 to `require.cache` .

This makes it such that 753’s mapping cannot be changed again since 753→434 exists in relativeResolveCache and 434 now exists in require.cache.

---

Looking at the bigger picture, we can conclude that relativeResolveCache+require.cache has the highest priority, then comes the prototype, then the _pathCache and finally the actual file.

The way the cache works can be simulated using python using the following (ugly) code and it will spit out the files loaded/the operations called in correct order along with correct arguments.

```python
f=open('required.js','r')

x=f.read().split('\n')

x=x[1:-2]

rcache=[]
rrcache={}
proto={}
pathcache={}

files=[]

def plis(file):
    global files,rcache,rrcache,proto,pathcache,line
    if file in rrcache.keys() and rrcache[file] not in rcache:
        rrcache.pop(file)
    if file in rrcache.keys() and rrcache[file] in rcache:
        return
    elif file in proto.keys():
        pathcache[file]=proto[file]
        if proto[file] not in rcache:
            rrcache[file]=proto[file]
            rcache.append(proto[file])
    elif file in pathcache.keys():
        if pathcache[file] not in rcache:
            rrcache[file]=pathcache[file]
            rcache.append(pathcache[file])
    else:
        pathcache[file]=file
        if file not in rcache:
            rrcache[file]=pathcache[file]
            rcache.append(file)

for line in x:
    args=line.split(')(')[1][:-1]
    args=args.split(',')
    i=args[0]
    j=args[1]
    t=args[2]
    file=line.split('./')[1].split("'")[0]

    if "require('./28')" in line:
        proto={}
        proto[i]=j
        plis(i)
    elif "require('./157')" in line:
        proto={}
        proto[i]=t
        plis(i)
    elif "require('./299')" in line:
        proto={}
        proto[t]=j
        plis(t)
    elif "require('./394')" in line:
        proto={}
        proto[t]=i
        plis(t)
    elif "require('./555')" in line:
        proto={}
        proto[j]=t
        plis(j)
    elif "require('./736')" in line:
        proto={}
        proto[j]=i
        plis(j)
    elif "require('./556')" in line:
        rcache=[]
    else:
        if file in rrcache.keys() and rrcache[file] not in rcache:
            rrcache.pop(file)
        if file in rrcache.keys() and rrcache[file] in rcache:
            files.append([rrcache[file],i,j,t])
        elif file in proto.keys():
            files.append([proto[file],i,j,t])
            pathcache[file]=proto[file]
            if proto[file] not in rcache:
                rrcache[file]=proto[file]
                rcache.append(proto[file])
        elif file in pathcache.keys():
            files.append([pathcache[file],i,j,t])
            if pathcache[file] not in rcache:
                rrcache[file]=pathcache[file]
                rcache.append(pathcache[file])
        else:
            pathcache[file]=file
            if file not in rcache:
                rrcache[file]=pathcache[file]
                rcache.append(file)
            files.append([file,i,j,t])

for file in files:
    name=file[0]+'.js'
    with open(name,'r') as f:
        y='f'+f.read().split(',f')[1]
        file[1]=int(file[1])%30
        file[2]=int(file[2])%30
        file[3]=int(file[3])%30
        y=y.replace('i',str(file[1]))
        y=y.replace('j',str(file[2]))
        y=y.replace('t',str(file[3]))
        if y[-1]==")":y=y[:-1]
        print(y)

f.close()
```