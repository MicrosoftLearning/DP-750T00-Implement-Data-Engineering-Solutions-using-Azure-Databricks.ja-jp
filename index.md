---
title: オンラインでホストされる手順
permalink: index.html
layout: home
---

このページは、[Microsoft Learn](https://learn.microsoft.com/en-us/training/courses/dp-750t00) での DP-750 (* Azure Databricks を使用して Data Engineering ソリューションを実装する*) Microsoft スキル習得コンテンツに関連する演習の一覧です

> **注**:コンテンツにバグが見つかった場合は、[GitHub リポジトリに新しい問題を作成](https://github.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/issues/new)してください。

{% assign labs = site.pages | where_exp:"page", "page.url contains '/Instructions/Labs'" %}
{% for activity in labs  %}
## ラボ {{ activity.lab.index}}: [{{ activity.lab.title }}]({{ site.github.url }}{{ activity.url }})  
  
{{ activity.lab.description }}

- 所要時間: {{ activity.lab.duration }}
- 詳しくは、[こちら]({{ activity.lab.module-url }})の学習モジュールをご覧ください。
- ノートブックのサポートについては、[こちら]({{ activity.lab.notebook }})で確認できます。

{% endfor %}
