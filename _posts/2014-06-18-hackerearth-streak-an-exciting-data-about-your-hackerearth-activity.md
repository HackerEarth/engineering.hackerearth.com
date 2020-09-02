---
author: Ravi Ojha
layout: post
title: "HackerEarth Streak: An exciting data about your HackerEarth activity"
description: "Code Streak:  Maximum number of unique problems solved continuously
 Day Streak: Maximum number of days such that one new problem is
   solved each day..."
category: 
tags: [HackerEarth, Algorithm, HackerEarth-Streak]
---
{% include JB/setup %}

We have been constantly adding features to our [HackerEarth Developer Profile](http://www.hackerearth.com/about/profile/), making it better with every new update.  
One of the exciting data in it is HackerEarth Streak on the [HackerEarth Activity](http://www.hackerearth.com/users/akatsuki/activity/hackerearth/) page.

This post explains how we process user data and compute such results. We split it into two parts:  

 - Code Streak:  Maximum number of unique problems solved continuously
 - Day Streak: Maximum number of days such that one new problem is
   solved each day

#### Code Streak: ####

**Extracting Relevant Data**  

All we have is the data about all the submissions made by any user. 
In order to process Code Streak we need to identify two things.

 1. To which problem the submission was made to?

Now we have several different types of Problems in [Challenges](http://www.hackerearth.com/challenges/) and [Practice Problem](http://www.hackerearth.com/problems/). For eg: Programming Problem, Approximate Problem, Golf Problem etc.

Each problem is assigned two attributes. `Type` (Programming Problem, Approximate Problem, Golf Problem etc) and `ID` (1,2,3 and so on).

So in order to uniquely identify any problem we need both the attributes, because a Programming Problem can have the same `ID` as Approximate Problem, the difference is in the `Type`.

 2. Was the submission graded Correct or Incorrect?

This is directly accessible from a single boolean attribute `Solved` which is `True` if Correct and `False` if Incorrect.

Now that we have extracted the data enough to calculate Code Streak, we move to the Data Structure part.

**Solution - The Data Structure behind**

Recently there was a question inspired from Code Streak in [June Easy Challenge](http://www.hackerearth.com/june-easy-challenge-14/problems/) as [Roy and Code Streak](http://www.hackerearth.com/problem/algorithm/roy-and-code-streak/).
It was simplified and `ID` ranges were restricted to convert the problem into easier one. A linear DP (Dynamic Programming) solution was enough to solve the problem.

Algorithm steps are as follows:

    Find all the ranges of Correct submissions
    For each range, count all the unique problems which have never been solved before
    Store the count in an array
    Find the maximum from the array

The second point is the challenge here. How do we know whether the problem with particular `Type` and `ID` was solved before or not?
Also the range of ID is not restricted here, unlike the problem in June Easy Challenge where DP solution worked. So DP fails here.
So we switched to Hashing. When a problem is solved for the first time, we increase the count and add the problem to the Hash. So the next time when we encounter the problem again it will be there in the Hash and hence we don't increase the count.

After processing all the ranges of Correct Submissions, all we have to do is find the `max` of all the counts. That's it. Code Streak calculated.

#### Day Streak: ####

**Extracting Relevant Data**

All the data that we extracted for Code Streak is reused here. Additional info is Timestamp (only `Date` part), which is also noted down when any solution is submitted.

**The Problem - The Solution**

The problem is, how to process this data efficiently?
One easy way out is, for each day extract all the user submissions and check if any new problem was solved that day. But for each day making such a query, means querying in a loop, seems inefficient.

So, I used a workaround, pre-calculate all the dates of current century, which turns out to be around 365*100 days.
We have an advantage that, both the pre-calculated dates and extracted user data are sorted w.r.t. `Date` in Timestamp.

So we can traverse through pre-calculated dates and extracted user data simultaneously maintaining two different variables and whenever both pre-calulated and user-submission dates match, we check if any unique problem was solved that day (this checking is similar to the problem faced in Code Streak and again it was solved using Hashing)

Further, we maintain another array of the same size as the number of days in a century. A day is marked `True` if any new unique problem was solved that day else it is marked `False`. Now all we have to do is find the longest range of all `True`s in this array. This can be easily done in linear time. There it is. Day Streak calculated.

Thus we avoided querying in a loop, yet above algorithm highly relies on number of submissions made by user one particular day.

**Note:** I believe we can still optimize these calculations by using any way other than Hashing. Please feel free to drop a comment below.

*Posted by [Ravi Ojha](http://www.hackerearth.com/users/akatsuki/).*