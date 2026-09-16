---
layout: page
title: Portfolio
tags: [portfolio, experience]
excerpt: "Portfolio list."
---

{%- assign date_format = site.minima.date_format | default: "%b %Y" -%}
{%- assign items = site.portfolio | sort: "start_date" | reverse -%}

<div class="post-list-section">
  <div class="section-header">
    <span class="section-title">Portfolio</span>
    <span class="section-meta">{{ items.size }} projects &middot; newest first</span>
  </div>

  <div class="post-rows">
    {%- for item in items -%}
    <a class="post-row" href="{{ item.url | relative_url }}">
      <span class="post-row-date">
        {{ item.start_date | date: date_format }}&ndash;{%- if item.end_date -%}{{ item.end_date | date: date_format }}{%- else -%}Present{%- endif -%}
      </span>
      <span class="post-row-title">{{ item.title | escape }}</span>
      <span class="post-row-arrow">&rarr;</span>
    </a>
    {%- endfor -%}
  </div>
</div>
