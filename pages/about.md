---
layout: page
title: About
description:改变从现在开始
keywords: amber
comments: true
menu: 关于
permalink: /about/
---

仰慕「优雅编码的艺术」。

## 坚信

* 熟能生巧
* 努力改变人生

## Skill Keywords

#### Software Engineer Keywords
<div class="btn-inline">
    {% for keyword in site.skill_software_keywords %}
    <button class="btn btn-outline" type="button">{{ keyword }}</button>
    {% endfor %}
</div>

#### Mobile Developer Keywords
<div class="btn-inline">
    {% for keyword in site.skill_mobile_app_keywords %}
    <button class="btn btn-outline" type="button">{{ keyword }}</button>
    {% endfor %}
</div>

#### Windows Developer Keywords
<div class="btn-inline">
    {% for keyword in site.skill_windows_keywords %}
    <button class="btn btn-outline" type="button">{{ keyword }}</button>
    {% endfor %}
</div>
