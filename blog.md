---
permalink: /blog/
---

## ✍️ Blog

This blog serves as my **primary writing hub**, where I document and reflect on my research
and engineering work in security and trustworthy AI.

I mainly write about:

- **AI & Software Supply Chain Security**
- **MCP Attacks, Tool Poisoning, and Defenses**
- **Secure Systems Design (Rust, Sandboxing, Telemetry)**

Most posts are written as **research notes, engineering deep dives, or companion articles**
to my papers and open-source projects.

---

## 📌 Featured Posts

- [Practical Techniques for High-Performance Processing on Large-Scale Graphs](https://newt-tan.medium.com/practical-techniques-for-high-performance-processing-on-large-scale-graphs-379921d2c2b9)
- [Monitoring Docker Runtime Activity on Linux with Tracee: What the Documentation Doesn’t Tell You](https://newt-tan.medium.com/monitoring-docker-runtime-activity-on-linux-with-tracee-what-the-documentation-doesnt-tell-you-6b22669e251c)
- [How to Solve Dependency Issues in Mythic’s C2 Profile Under a New Virtual Environment](https://newt-tan.medium.com/how-to-solve-dependency-issues-in-mythics-c2-profile-under-a-new-virtual-environment-1fbd1a4bc16d)
- [Exploiting Hidden Weakness in ML-Powered Web Apps: A Case Study](https://newt-tan.medium.com/exploiting-hidden-weakness-in-ml-powered-web-apps-a-case-study-6be4e70f31c8)

---

## 🔗 External Writing

Some articles are currently published on Medium for broader dissemination.
Future long-form and research-oriented posts will be **hosted directly on this site**.

👉 Medium profile: https://newt-tan.medium.com/

---

## Recent Posts

<ul>
{% for post in site.posts limit:5 %}
  <li>
    <a href="{{ post.url }}">{{ post.title }}</a>
    <small>({{ post.date | date: "%Y-%m-%d" }})</small>
  </li>
{% endfor %}
</ul>