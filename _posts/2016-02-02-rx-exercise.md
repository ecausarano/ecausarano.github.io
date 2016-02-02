---
layout: post
title: ETL with RxJava 
excerpt: "Having fun with RxJava"
categories: Development
comments: false
---

A couple days ago I made a post discussing about Java 8 streams for CSV files processing; the solution is quite straightforward and easy to understand, until one starts to wonder at the small details in the code. Those details you normally end up wasting countless hours on.

Let's look at a different solution to the problem, one that looks more like the `Monads` that the Java 8 developers didn't want to be caught alive mentioning. ;)

## RxJava

RxJava introduces the abstract data type *observable sequences* to define and encapsulate programming logic. Each step can be written as a pure function and composed together with other abstractions that don't produce any result except interact with the outside world. This way you can hide all dirty, exception-prone code in a Subscriber and test it separately, while the rest of the purely logical code can be verified with mock Observables and Subscribers. Nice.

If you were using Scala or some other functional language you'd be calling these Monads. Here's an example of the same exercise written using Rx:

{% highlight java %}
    public static void main(String[] args) throws IOException {
        final String filename = System.getProperty(INPUT_FILE);
        try (final FileInputStream fileInputStream = new FileInputStream(filename)) {
            final Observable<String> lines = Observable.from(new IterableWrapper(fileInputStream));
            lines.subscribe(line -> {
                final String[] rowItems = tabPattern.split(line);
                Observable.from(rowItems)
                        .skipWhile(s1 -> !containsId(s1))
                        .subscribe(new Processor());
            });
        }
    }

    private static boolean containsId(String id) {
        return "ID1".equals(id);
    }

    private static class Processor extends Subscriber<String> {
        StringBuffer buffer = new StringBuffer();

        @Override
        public void onCompleted() {
            System.out.println(buffer.toString().trim());
        }

        @Override
        public void onError(Throwable e) {
            System.err.printf("Processing error: %s", e.getMessage());
        }

        @Override
        public void onNext(String s) {
            buffer.append(s).append("\t");
        }
    }   
{% endhighlight %}

The problem with Rx is that's still Java: even when you're writing a "pure" function - one that given a certain input produces a given output and nothing else, always - you can't really ever be sure that it is not interacting with state under you nose, or croaking under a runtime exception. It's still entirely possible to throw exceptions or do IO outside of the subscribers, don't do that.

{% highlight java %}

	// ...
		
        try (final FileInputStream fileInputStream = new FileInputStream(filename)) {
            final Observable<String> lines = Observable.from(new IterableWrapper(fileInputStream));
            lines.subscribe(
                    nextLine -> {
                        final String[] rowItems = tabPattern.split(nextLine);
                        Observable.from(rowItems)
                                .skipWhile(s1 -> !containsId(s1))
                                .subscribe(new Processor());
                    },
                    throwable -> {
                        System.err.printf("Got an error, but we can handle that! Message: %s", throwable.getMessage());
                    });
        }
    }

    private static boolean containsId(String id) {
        final int randomInt = new Random().nextInt(10);
        System.out.printf("Random interaction: %s\n", randomInt);
        if (randomInt < 2) {
            throw new RuntimeException("A-Ha!");
        }
        return "ID1".equals(id);
    }
{% endhighlight %}

```
Random interaction: 1
Random interaction: 4
ID1	VAL11	VAL12	VAL12
Random interaction: 9
Random interaction: 4
Random interaction: 0
Processing error: A-Ha!Processing error: A-Ha!Processing error: A-Ha!Processing error: A-Ha!
```

It's still awesome though... 

