# Blog

## Useful commands

```
hugo server
```

create new content
```
hugo new content content/posts/x.md
```

Random stuff to try out:
## Tabs
```md
{{</* tabs tabTotal="2" */>}}

{{%/* tab tabName="First Tab" */%}}
This is markdown content.
{{%/* /tab */%}}

{{</* tab tabName="Second Tab" */>}}
{{</* highlight text */>}}
This is a code block.
{{</* /highlight */>}}
{{</* /tab */>}}

{{</* /tabs */>}}
```
## Details
dropdown menu like thing
```md
{{</* details summary="A detail dropdown" */>}}
Markdown content
{{</* /details */>}}
```
