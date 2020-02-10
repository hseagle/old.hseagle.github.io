---
title: "如何使用Plainwhite使用"
author: pengxu
categories: jekyll PlainWhite
date: 2020-02-10 21:09:00 +0800
---

> 如何使用PlainWhite模板并定制相关选项

## 引入PlainWhite

在_config.yml中添加如下内容

```yml
remote_theme: thelehhman/plainwhite-jekyll
permalink: pretty
urls: https://hseagle.github.io
plugins:
    - jekyll-feed
    - jekyll-archives
    - jekyll-sitemap
plainwhite:
 name: Your Name
 tagline: Developer. Architect
 date_format: "%b %-d, %Y"
 sitemap: true
show_excerpts: true
```

## 激活sitemap

添加文件sitemap.xml

```bash
{%-if site.plainwhite.sitemap -%}
    <?xml version="1.0" encoding="UTF-8"?>
    <urlset
        xmlns="http://www.sitemaps.org/schemas/sitemap/0.9"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.sitemaps.org/schemas/sitemap/0.9
                http://www.sitemaps.org/schemas/sitemap/0.9/sitemap.xsd">
        <url>
            <loc>{{ site.url }}</loc>
            <lastmod>2019-06-07T00:00:00+00:00</lastmod>
            <priority>1.00</priority>
        </url>
        {%- for post in site.posts -%}
            <url>
                <loc>{{ site.url }}{{ post.url }}</loc>
                <lastmod>{{ post.date | date_to_xmlschema }}</lastmod>
                <changefreq>weekly</changefreq>
                <priority>0.80</priority>
            </url>
        {%- endfor -%}
    </urlset>    
{%- endif -%}
```

把index.html中的内容改为

```html
---
layout: home
---
```
