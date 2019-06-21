---
#date: 2019-06-18
title: make iframe page transparent
#tags: ["tag"]
#categories: ["cat"]
#summary: summary here
links:
  - icon_pack: fas
    icon: blog
    name: Originally article
    url: 'https://www.cnblogs.com/yeminglong/p/3842048.html'
---

1. Adding **allowtransparency="true"** to an iframe tag.

```html
<iframe src="http://www.ylsapt.com/ProductSearch.html" width="100%" height="214" frameborder="0" scrolling="no" allowtransparency="true"></iframe>
```

2. Make body style of your page to transparent. 

```html
<body style="background:transparent;">  
```