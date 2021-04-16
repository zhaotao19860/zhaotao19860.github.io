---
layout: post
title: social icons
date: 2021-02-08 20:54:00 +0800
category: Tools
---

#### 概述

博客中社交图标(social icons)通常用来表示链接一些网站的logo，使界面简洁好看，比如github/qq/email等，下面从获取到配置到使用给出了基于jeckll的详细配置，并提供了将svg图标缩小和竖排改为横排的方法。

#### 获取

```bash
git clone https://github.com/edent/SuperTinyIcons.git
cd SuperTinyIcons/images/svg
cp email.svg github.svg zhaotao19860.github.io/public/img
```

#### _config.yml(配置)

```yaml
title: ZhaoTao
email: zhaotao19860@qq.com
description:
  Just Enjoy It
highlighter-theme: monokai
minima:
  social_links:
    github:  zhaotao19860
```

#### social.html(包装)

```html
{% raw  %}
{%- assign social = site.minima.social_links -%}
<ul class="social-media-list">
  {%- if social.github -%}<li><a rel="me" href="https://github.com/{{ social.github | cgi_escape | escape }}"
      title="{{ social.github | escape }}"><img src="/public/img/github.svg" /></a></li>{%- endif -%}
  {%- if site.email -%}<li><a rel="me" href="mailto:{{ site.email | escape }}" title="{{ social.email | escape }}"><img
        src="/public/img/email.svg" /></a></li>{%- endif -%}
</ul>
{% endraw %}
```
#### sidebar.html(引用)

```html
{% raw  %}
<div class="sidebar-item sidebar-header">
	<div class='sidebar-brand'>
		<a href="/about/">{{ site.title }}</a>
	</div>
	<p class="lead">{{ site.description }}</p>
	<div class="social-links">
		{%- include social.html -%}
	</div>
</div>
{% endraw %}
```

#### public/css/style.css(一行显示)

```css
.social-media-list li {
	display: inline-block;
}
```

#### default.html

```html
{% raw  %}
<html lang="{{ page.lang | default: site.lang | default: "en" }}">
  {%- include head.html -%}
  <style>@import url(/public/css/syntax/{{ site.highlighter-theme }}.css);</style>
  <title>{{ site.title }}</title>
  <!-- <link href="/public/css/bootstrap.min.css" rel="stylesheet"> -->

  <link href="/public/css/style.css" rel="stylesheet">
  <body>
  	<div class="container"> 
		<div class="sidebar">
			{% include sidebar.html %}
		</div>
		<div class="content">
			{{ content }}
		</div>
	</div>
  </body>
</html>
{% endraw %}
```

#### 关键点

```
1. 调整svg大小
   使用preserveAspectRatio/viewBox/width/height五个参数；
2. 调整svg颜色
   使用fill参数；
3. 将512*512调整为32*32
    <svg xmlns="http://www.w3.org/2000/svg"
    aria-label="GitHub" 
    role="img"
    viewBox="0 0 512 512">
    <rect width="512" height="512" rx="15%" fill="#1B1817"/>
    <path fill="#fff" d="M335 ... 16 12z"/>
    </svg>
    改为：
    <svg xmlns="http://www.w3.org/2000/svg"
    aria-label="GitHub" 
    role="img"
    preserveAspectRatio="xMidYMid meet"  x="0"   y="0"
    viewBox="0 0 512 512" width="32"  height="32">
    <rect width="32" height="32" rx="15%" fill="#1B1817"/>
    <path fill="#fff" d="M335 ... 16 12z"/>
    </svg>
```

