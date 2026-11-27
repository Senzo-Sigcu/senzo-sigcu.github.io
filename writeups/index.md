---
layout: page
title: Write-ups
---

Lab walkthroughs and analysis. Each one is tagged by **platform** and by the **skills** it demonstrates, so you can scan for the kind of work you care about rather than reading everything.

{% assign entries = site.pages | where: "writeup", true | sort: "date" | reverse %}
{% if entries.size == 0 %}
_No write-ups published yet._
{% else %}
{% for w in entries %}
### [{{ w.title }}]({{ w.url | relative_url }})
**Platform:** {{ w.platform }}
**Skills:** {% for s in w.skills %}`{{ s }}`{% unless forloop.last %} {% endunless %}{% endfor %}
{% endfor %}
{% endif %}

---

**Skill tags I use:** `Enumeration` · `Vulnerability Identification` · `Payload Selection` · `Privilege Escalation` · `Log Analysis/Detection` · `Reporting`. I add a new tag only when it genuinely names a distinct capability: tags are for scanning, not decoration.
