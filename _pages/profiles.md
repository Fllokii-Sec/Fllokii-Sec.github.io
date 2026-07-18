---
layout: page
title: toolbox
permalink: /toolbox/
nav: true
order: 4
---

{% if site.data.skills %}
  {% for section in site.data.skills %}
    <h3 class="mt-4" style="font-weight: 600;">{{ section.category }}</h3>
    <hr style="margin-top: 5px; margin-bottom: 20px;">
    
    <div class="toolbox-grid" style="display: flex; gap: 20px; flex-wrap: wrap; margin-bottom: 40px;">
      {% for tool in section.tools %}
        <div class="tool-item" style="text-align: center; width: 85px;">
          <img src="https://img.shields.io/badge/-{{ tool.name | url_encode }}-{{ tool.color }}?style=square&logo={{ tool.icon | url_encode }}&logoColor=white" 
     width="70" 
     height="70" 
     alt="{{ tool.name }}" 
     style="border-radius: 8px;">
          <p style="font-size: 12px; margin-top: 7px; font-weight: bold; line-height: 1.2; color: var(--global-text-color);">{{ tool.name }}</p>
        </div>
      {% endfor %}
    </div>
  {% endfor %}
{% endif %}
