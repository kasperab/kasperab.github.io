---
title: Stadskompassen
description: Quiz about political decisions to find out your leadership style.
category: Frontend
tags: React Node.js Bootstrap JavaScript HTML CSS Group
web: "https://linneabackgard.github.io/Stadskompassen"
code: "https://github.com/LinneaBackgard/Stadskompassen"
---

We wanted to make something like an educational game, and landed on a political quiz where instead of a party or political leaning it gives you a leadership style. The goal was to not judge people for their choices or say that some choice is better than another, but rather to show the benefits and drawbacks of every choice.

For most of the project we sat together in a video call, taking turns to program and share our screen while talking through everything. This allowed us to always know where the project was and what was going on, but had the drawback of limiting our effective time to that of a single person.

The website is made of several layers of nested components for different pages. For example: the main page can display the questions, which in turn display options. When choosing an option they will be replaced with some immediate feedback for that option before you continue to the next question and new options are shown. After finishing everything the questions get replaced with results. Each option gives one or two points towards a couple of leadership styles, and the one with most points is what you get at the end. You also get a list of every leadership style with points for each of them.
