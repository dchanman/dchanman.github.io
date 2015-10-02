---
layout: post
title: "Pigeonhole Sort"
modified:
categories: blog
excerpt: An Amazon interview question
tags: [programming]
image:
  feature:
date: 2015-10-01T23:26:04-07:00
---
I had coffee with an old mentor today. The timing was quite interesting; he just had a phone interview with Apple and I just finished an interview for a summer internship the same afternoon.

Since we were both in an interview mood, he told me an interesting interview question from Amazon:

> Suppose you had 10 million 10-bit numbers. How would you sort them and what is the time complexity?

I just had my Comp Sci class in the morning teach me about sorting algorithms. I instinctively said:

**"I'd just mergesort them and have a complexity of O(n log n)."**

And then he told me to think about it some more. 

---

A 10-bit number is in the range of 0 to 1024. You have 10 million of these numbers. That means it's impossible to **not** get any duplicates.

Because there are so many, the easiest way to sort them is to actually count them.

{% highlight c %}

int counts[1024] = {0};

/* Count up the numbers */
for (i = 0; i < 10000000; i++) {
	number = input_numbers[i];
	counts[number]++;
}
{% endhighlight %}

Because we're only going through the input_numbers once, the sorting complexity is  
**O(n log n)**.

---

The nifty thing about this is how this set of 10 million numbers essentially gets compressed into a small array of counts. If you wanted to look up a specific index, say index 72359 for example, you would have to add up the counts starting for the number 0 until you reached 72359. Although this seems like an expensive operation to have to do all this adding, remember that we have a constant maximum of 1024 numbers to add. Indexing will essentially be done at constant time (although much faster for lower index values).

Taking this question a step further, to speed up indexing, you could precalculate an index lookup table:

{% highlight c %}
int index_cache[1024] = {0};

/* Calculate the index values and stash them in our cache */
for (i = 1; i < 1024; i++) {
	index_cache[i] = index_cache[i-1] + counts[i-1];
}
{% endhighlight %}

And there we go. If we needed to find an index, we could just binary search our cache.
