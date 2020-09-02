---
author: Sayan Chowdhury
layout: post
title: "Analytics for Challenges"
description: "Detailed Analytics of data retrieved from challenge"
category: 
tags: [Challenge, Analytics, Google Charts]
---
{% include JB/setup %}

First view the awesome charts and graph for [HackerEarth Practice Challenge] (http://www.hackerearth.com/hackerearth-practice-challenge/analytics/). I will
explain it's implementation details later in the post.

<br>
**Why bother for Challenge Analytics?**

We thrive on challenge but challenge is no fun without a detailed analytics.
In this number-driven world, analytics has evolved as a blanket term for the number of techniques to turn the raw data into useful information.

Late February, I started working on to implement Analytics using the data
collected during the challenges over the period of time and present it
lucrative manner. After performing a random wild-goose chase through various
JavaScript charting libraries, I decided to stop at Google Charts Tools.

<br>
**Why use Google Charts Tools?**

There are a lot of reasons to choose Google Charts over the others charting libraries.
 - Free.
 - Lightweight and Reliable.
 - Healthy Documentation.

<br>
**What you gain from Challenge Analytics?**

The HackerEarth Challenge Analytics gives following details:

 - *Submission Count Analytics*
<br>The Submission Count Analytics displays a Line-Chart with the total number of submissions at a particular instance of time. This chart is updated at regular time intervals.
 - *Language Analytics*
 <br>The Language Analytics displays a Pie-Chart depicting the popularity of a programming language supported by HackerEarth,
   during a challenge. Personally, this chart is my rock favorite as it helps me figure out the current trend of a particular
   language. 
 - *Submission Analytics*<br>
   The Submission Analytics displays a Pie-Chart depicting the submission status namely 
    * AC - Accepted
    * CE - Compilation Error
	* TLE - Time Limit Exceeded
	* MLE - Memory Limit Exceeded
	* RE - Runtime Error
 - *Pinhole Analytics*
 <br>The Pinhole Analytics displays a Pie-Chart depicting the run-status for
 the solution against various test-cases.
 - *Multilingual Users*
<br> A true programmer proves himself/herself by programming around in various kind of languages. This table is leaderboard for them, the more the language you use in a challenge the higher you get in the table.

<br>
**Implementing Challenge Analytics**

The Challenge Analytics uses two of the Google Charts 
- Line-Chart
- Pie-Chart

As talked about earlier Submission Count Analytics uses a Line Chart
for depicting the computed data

The Line Chart(Google Chart Tools) is documented [here](https://developers.google.com/chart/interactive/docs/gallery/linechart).

Google charts requires the JSAPI library, This library is loaded by:
{% highlight js %}
    <script type="text/javascript" src="https://www.google.com/jsapi"></script>
{% endhighlight %}

<br>
The script given below loads the Google Visualization and the Chart libraries. This is also responsible for displaying the chart.
    
    {% highlight js %}
    <script type="text/javascript">
        google.load("visualization", "1",{ callback : drawChart, packages:["corechart"]});
        function drawChart() {
        var data = new google.visualization.DataTable();
        data.addColumn('string', 'Time');
        data.addColumn('number', 'Submissions');
        data.addColumn({type:'string',role:'tooltip'});
        data.addRows( {{ submission_count_data|safe }} );

        var options = {
            'width': 850,
            'height': 500,
            'chartArea': {left:100, top:70},
            'pointSize': 4,
            hAxis: {
                title: 'Time',
                slantedText: true,
                slantedTextAngle: 20,
                textStyle: {fontSize: 10}
            },
            vAxis: {
                title: 'Submissions',
            }
        };
        var chart = new google.visualization.LineChart(document.getElementById('submission-count-chart'));
        chart.draw(data, options);
    }
    </script>
    
    <div id="submission-count-chart"></div>
    {% endhighlight %}
<br>
The *google.load* package name is "corechart"
{% highlight js %}
    google.load("visualization", "1", {packages: ["corechart"]});
{% endhighlight %}
<br>
The visualization's class name is *google.visualization.LineChart*
{% highlight js %}
    var visualization = new google.visualization.LineChart(container);
{% endhighlight %}
<br>
The *drawChart()* function creates a [DataTable](https://developers.google.com/chart/interactive/docs/datatables_dataviews) and is populated with computed data. The required number of columns are added mentioning the data format
{% highlight js %}
    data.addColumn('number', 'Submissions');
{% endhighlight %}
<br>
For customizing the tooltip, an extra column was added and the 'role':'tooltip' is specified.
{% highlight js %}
    data.addColumn({type:'string',role:'tooltip'});
{% endhighlight %}
<br>
The *option* object is used for customizing the chart to be displayed. The customizable option for a Line-Chart is available [**here**](https://developers.google.com/chart/interactive/docs/gallery/linechart#Configuration_Options).

This instantiates an instance of chart class you want to use for e.g. LineChart, PieChar etc., by passing in some options. The Google chart constructors takes a single parameter: a reference to the DOM element in which to draw the visualization.
{% highlight js %}
      var chart = new google.visualization.PieChart(document.getElementById('submission-count-chart'));
{% endhighlight %}

<br>
Once the chart is ready, a HTML element is created to hold the particular chart.
{% highlight html%}
    <div id="submission-count-chart"></div><div id="submission-count-chart"></div>
{% endhighlight %}
<br>
PieChart works on a similar principle, except for:

 - The visualization's class name is google.visualization.PieChart
{% highlight js %}
        var visualization = new google.visualization.PieChart(container);
{% endhighlight %}
 - The Options for PieChart is documented [**here**](https://developers.google.com/chart/interactive/docs/gallery/piechart#Configuration_Options).

While implementing Google Charts, I faced a weird problem, due to the use of google.load() after the loading of the charts the page would go black instantly, but the problem was solved by using callback parameter to google.load(). A nice blog-post can be read [here](http://jona.than.biz/blog/using-google-charts-api-inside-jquery-load/).

For any queries or suggestions, you can shoot me a mail at sayan@hackerearth.com.

P.S. I am an undergraduate student at Dr. B.C.Roy Engineering College, Durgapur. You can also find me [@chowdhury_sayan](https://twitter.com/chowdhury_sayan)

*Posted by Sayan Chowdhury, Intern @HackerEarth*
