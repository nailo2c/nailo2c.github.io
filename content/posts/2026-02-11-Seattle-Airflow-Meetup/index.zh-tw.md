---
title: "Seattle Airflow Meetup 潛入日記"
date: "2026-02-11T00:00:00-00:00"
draft: true
tags: []
categories: ["Airflow"]
---

大膽地報名了Seattle Airflow Meetup，活動當天還遇到Seattle Seahawks封王遊行，原以為交通會非常崩潰，好在Meetup時間完美避開了封王遊行。

Meetup 在 Amazon 大樓內舉辦，換了訪客證之後順利地進去了。參與人數比我想像中少，最後目測人數大概15~20人左右。

食物看起來很讚，吃起來也很讚。當我拿完食物後，發現非常少人在會議室，只好吃的時候跟他們硬聊。一開始跟 Astronomer 的人聊，問了他們這個 Meetup 的頻率，聽起來下次舉辦可能會在Q4？另外我也提到台灣也有類似的活動，只是我們以 Virtual Meetup 的方式進行。

後來跟兩位 Amazon 工程師聊，他說他們 team 有在用 Airflow，最近忙著升級，要從 2.5 升到 2.6 (這也太舊🤣)。他們說這些 Dags 都存在超過五年了，而他們也才入職一兩年，當初 Dags 的作者也都不在了，十分無奈。

今天有兩個 Topic，第一個 Topic 是2025年問卷review，基本上跟不久前的 Town Hall 內容差不多。第二個是 Niko 大大講解 Multi-tenant in Airflow，我自己聽一聽是在思考這樣背後的 metadata table 會不會需要大改？原本想提問，但想一想或許看 PR 就能找到答案了。最後發現可以看 AIP ticket的內容，十分詳細。

https://cwiki.apache.org/confluence/display/AIRFLOW/AIP-67+Multi-team+deployment+of+Airflow+components

整體來說是不錯的體驗，期待下一次的 Seattle Airflow Meetup。
