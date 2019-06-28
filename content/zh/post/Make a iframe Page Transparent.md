---
title: 使iframe页面透明
tags: ["HTML"]
categories: ["Web"]
links:
- icon_pack: fas
  icon: blog
  name: 来源文章
  url: https://www.cnblogs.com/yeminglong/p/3842048.html

---
1> 将**allowtransparency =“true”**添加到iframe标记。

```html
<iframe src="http://www.ylsapt.com/ProductSearch.html" width="100%" height="214" frameborder="0" scrolling="no" allowtransparency="true"></iframe>
```

2> 使页面的正文样式透明。

```html
<body style="background:transparent;">  
```

