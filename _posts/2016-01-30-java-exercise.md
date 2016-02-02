---
layout: post
title: Turning an OO Design test into an FP excercise 
excerpt: "Having fun with Java 8"
categories: Development
comments: false
---

When it comes to inventing a subject for a coding excercise, I'm not the most imaginative person. I've done the usual write your blogging platform in Spring, Clojure, maybe even Play at some point, as well as the omnipresent URL shortener (and at some point I should toss them on GH for clients to have a look, if only I haven't deleted most of them... doh!) So I'm often wondering about a mini-project to practice a new technique or outright learn a new trick or whatnot. Often though, I can't thing of anything simple enought, and I end sketching out marvellously complicated ideas that are more apt for a startup than for a Sunday afternoon workout. 

A couple days ago I was lucky enough be given a Java test from a prospective customer and what a golden opportunity it was! Over the next couple of days and weeks, I will use it as a pretext to practice several implementations besides the Java one, and if I happen to bother long enough, even venture into comparisons between the different implementations. 
For now let's start with Java. 8

## The problem

It's a typical data ingestion problem you would encounter in an ETL or data processing facility: we have an input CSV file - the first row being the column header names - a configuration file describing a mapping of the columns to an internal model and finally a file to map the IDs of the incoming data to those of our own data collection. Pretty easy, at first sight but you can make it as difficult as you want, possibly implementing quite a bit of properties of a relational database system.

Of course the solution is meant to be written in OO Java to show off your modeling skills, your Design Patters and whatnot, but here's where it gets tricky: as more and more people are suggesting, Object Oriented design is needlessly complicated and simpler solutions, yet still as powerful are possible when taking the FP approach.

## Java 8

Solving the problem in Java 8 means dishing out a stream pipeline such as the following code blob, where other interesting variables have been also similarly derived.  

{% highlight java %}
            try (Stream<String> lines = Files.lines(Paths.get(filename))) {
                lines.map(TAB_PATTERN::split)
                        .filter(entry -> rowIds.contains(entry[0]))
                        .filter(checkString -> checkString.length > Collections.max(outputIndexes))
                        .map(strings -> {
                            final StringBuilder stringBuffer = new StringBuilder();
                            // replace the ID with corresponding newId
                            strings[0] = idMapping.get(strings[0]);
                            for (int index : outputIndexes) {
                                stringBuffer.append(strings[index]).append("\t");
                            }
                            return stringBuffer.toString().trim();
                            // print result to stdout
                        }).forEach(System.out::println);
            }
{% endhighlight %}

There's a potential `IndexOutOfBoundException` lurking in the inside for loop that could be caused by a malformed file containing a short row. I'd rather assume it can never happen and introduce another filter to ignore the corrupted string. Another good todo is to always *try with resource* whenever touching the filesystem.

So It's all very interesting and all, it's great to solve the problem with just a handful of lines of code rather that define a whole family of domain objects, design patterns and whatnot , but there still are a couple things that make it a bit difficult to reason about what is going on in the code. (although it's still way more readable than a OO solution, by virtue of it's conciseness.)

* For one thing, how does filter skip an element when the predicate evaluates to false? Does it break, out of the loop or does it poison the element but still pass it on the rest of the pipeline?
* Inside the ```map``` lambda, I mutated the input String array; will this hurt when running in parallel, over multiple threads?
* There's a ```RuntimeException``` in the most inner loop, how will the stream handle it: will it bail out or will it go on with the next stream element?
* The last step of the pipeline contains a side effect, what will happen when run across parallel threads, possibly with state change inside? Ouch!

So in summary, Java Streams are great but not so much as to really give total peace of mind. I see where they come from and will use them whenever possible but I'm still not that convinced.