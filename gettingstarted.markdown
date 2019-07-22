---
layout: default
---

{% capture x %}{% include gettingstarted.markdown %}{% endcapture %}
{{ x | markdownify }}
