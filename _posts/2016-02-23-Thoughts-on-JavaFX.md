---
layout: post
title: Thoughts on JavaFX & Spring
excerpt: "Working with JavaFX is fun but can become a bit tricky when Spring is involved"
categories: Development
comments: false
---

It's been a while since I last touched JavaFX, but in general I'm quite happy with it. Sure, the layout engines are not as delicious as Apple's [Auto Layout](https://developer.apple.com/library/ios/documentation/UserExperience/Conceptual/AutolayoutPG/index.html) but there are a lot of pretty and functional widget libraries such as [JFxtras](http://jfxtras.org/) as well as [ControlsFx](http://fxexperience.com/controlsfx/). Just when I begin to doubt whether all UIs will eventually become Js SPAs I bump into awesome stuff like [Graphing in 3D with Dex and Vis.js](https://dexvis.wordpress.com/2016/02/21/graphing-in-3d-with-dex-and-vis-js/) and pause.

Whatever, today I'm hacking at [Heron](https://github.com/ecausarano/heron) and right now this means fighting the usual JavaFX Vs. Spring battle. I'm here to chronicle findings, frustrations, cookbooks, workarounds hacks and my own conclusions on the subject (as usual for future reference.)

So, let's start with the typical error

{% highlight java %}
feb 23, 2016 10:18:44 AM org.springframework.context.support.AbstractApplicationContext refresh
WARNING: Exception encountered during context initialization - cancelling refresh attempt
org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'mainWindowController': Injection of autowired dependencies failed; nested exception is org.springframework.beans.factory.BeanCreationException: Could not autowire field: eu.heronnet.module.gui.fx.views.BundleView eu.heronnet.module.gui.fx.controller.MainWindowController.bundleView; nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'localStorageView': Invocation of init method failed; nested exception is java.lang.ExceptionInInitializerError
...
Caused by: java.lang.IllegalStateException: Toolkit not initialized
	at com.sun.javafx.application.PlatformImpl.runLater(PlatformImpl.java:273)
	at com.sun.javafx.application.PlatformImpl.runLater(PlatformImpl.java:268)
	at com.sun.javafx.application.PlatformImpl.setPlatformUserAgentStylesheet(PlatformImpl.java:550)
	at com.sun.javafx.application.PlatformImpl.setDefaultPlatformUserAgentStylesheet(PlatformImpl.java:512)
	at javafx.scene.control.Control.<clinit>(Control.java:87)
	... 46 more
{% endhighlight %}

Duh! It's the usual chicken-egg-you-shall-obey-my-lifecycle, and-stay-out-of-my-thread! class of issues that haunt [Swing](https://docs.oracle.com/javase/tutorial/uiswing/concurrency/index.html), [SWT](http://help.eclipse.org/mars/index.jsp?topic=%2Forg.eclipse.platform.doc.isv%2Fguide%2Fswt_threading.htm) and so on. After all GUIs are among the most delicate and complex constructions one can build so it's quite understandable that the library designers had to set some pretty draconian rules just to maintain their own sanity.

So, this time JavaFX is complaining that I'm loading a widget while the whole toolkit hasn't finished boostrapping. That's quite annoying because I'd like to just _declare_ my widgets in a _Spring Context_ so I can wire them up together with their controllers and other service implementations. These widgets are custom JavaFX classes to control their own rendering, while all the custom logic is deferred to [Delegates](https://developer.apple.com/library/mac/documentation/General/Conceptual/CocoaEncyclopedia/DelegatesandDataSources/DelegatesandDataSources.html#//apple_ref/doc/uid/TP40010810-CH11). Problem is, Spring also instantiates the darn things and the common idea I found online is to call an FMXLoader in the constructor or some other early Spring lifecycle phase such as _@PostConstruct_

* **Be @Lazy:** let's start by trying to annotate all JavaFX beans declared in the Spring context with [@Lazy](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/context/annotation/Lazy.html) and... no cookie, we still get the same exception. Back to Google in hope some poor soul has already quarreled with this angry cat and found a solution. Interestingly enough, bumping into [Tom's Blog post](http://zezutom.blogspot.nl/2014/01/spring-series-part-4-lazy-on-injection.html) made me realize: ""Aha! I can declare the beans as lazy as I want, but if Spring is still looking them up while parsing the annotations it will still cause an early instantiation unless the annotation also added to the relevant _fields_! So I tried with just one custom widget, but the application started throwing obscure NullPointerExceptions when the proxy weaved in by Spring tries to actually create the JavaFX custom widget. :'(
* **Be creative** for my next attempt the reasoning was: _I don't want to let Spring handle any of the JavaFX classes, but I still want these to access a Delegate object created by the Spring context and wired up by the framework._ In this approach I use a lambda in the widget constructors to fetch the UIDelegate from the Spring context. So far, it works and I'm not too disgusted by the solution.

{% highlight java %}
public class HeronApplication extends Application {

    @Override
    public void start(Stage primaryStage) throws Exception {

        ApplicationContext applicationContext = Main.getApplicationContext();

        try {
            final MainWindowView mainWindowView = new MainWindowView(() -> applicationContext.getBean(UIController.class));

            Scene scene = new Scene(mainWindowView);
            primaryStage.setScene(scene);
            primaryStage.setTitle("Heron");
            primaryStage.show();
        } catch (RuntimeException e) {
            logger.error(e.getMessage());
        }
    }

}

public class MainWindowView extends VBox implements DelegateAware<UIController> {

public MainWindowView(Callable<?> delegateFactory) {
...
        try (InputStream fxmlStream = MainWindowView.class.getResourceAsStream("/HeronMainWindow.fxml")) {
            this.delegateFactory = delegateFactory;
            this.delegate = (UIController) delegateFactory.call();
            FXMLLoader fxmlLoader = new FXMLLoader();
            fxmlLoader.setRoot(this);
            fxmlLoader.setController(this);
            fxmlLoader.load(fxmlStream);
        } catch (Exception e) {
            logger.error("Error in MainWindowView ctor: {}", e.getMessage());
            throw new RuntimeException(e);
        }
    }
...
}
{% endhighlight %}
