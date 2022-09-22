---
title: josephus-problem
date: 2022-06-29 11:22:08
tags:
---

思路：
solution based on linked-list
solution based on DP
-

{% katex %} 
1 => k\%n + 1 => (1 + k -1)\%n + 1
{% endkatex %}
-
{% katex %} 
2 => (k+1)\%n + 1 => (2 + k -1)\%n + 1
{% endkatex %}
-
{% katex %} 
3 => (k+2)\%n + 1 => (3 + k -1)\%n + 1
{% endkatex %}
-
{% katex %} 
f(n, k) = (f(n-1, k) + k -1)\%n + 1
{% endkatex %}
