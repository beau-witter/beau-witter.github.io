---
layout: page
title: About
permalink: /about/
start_year: 2014
---

## The Author
{% assign current_year = 'now' | date:'%Y' %}
{% assign total_years = current_year | minus: page.start_year %}
Hi there. My name is Beau, and I have been working in the field of Software Engineering, professionally, for {{total_years}} years. I left college with a Bachelors of Science in Computer Science and a job as an entry level Software Engineer. The first 7 odd years of that time I was a Full Stack .NET Developer. The last {{total_years | minus: 6}} or so years (yes, some overlap) of the time leading up to starting this blog, and including its continued updating, I started going deeper into the realm of DevOps and increasing the amount of my time spent to that end.

I am an avid learner. I believe that one of the most fun activities out there is to learn! In the space I work in, one can find an enormous amount of both breadth and depth which ensures there is always more to learn.

## The Blog

This blog sets out to do what pretty much any personal tech blog sets out to do, I would think. I run into novel issues that take hours of searching and reading and finding to try to discover the root cause, why it is like that, and what to do about it. And this can be from multiple sites and sources! Then what? I just either implement what I figured out and it works or discover it can’t actually do that and pivot or move on? All that discovery, searching, and compilation wouldn’t be able to help anyone else or even probably me when I come back to the same problem again.

Thus the Cloud Tech Info Dump blog! This will be a repository of sorts for specific issues or problems I discover and tackle and the results of that effort.