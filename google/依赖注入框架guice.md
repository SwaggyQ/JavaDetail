# 谷歌的依赖注入框架guice
```
package com.basic;

/**
 * Created by sugu on 2019/2/22.
 */
import javax.inject.Singleton;

import com.google.inject.Guice;
import com.google.inject.Inject;
import com.google.inject.Injector;

@Singleton
class HelloPrinter {

    public void print() {
        System.out.println("Hello, World");
    }

}

@Singleton
public class Sample {

    @Inject
    private HelloPrinter printer;

    public void hello() {
        printer.print();
    }

    public static void main(String[] args) {
        Injector injector = Guice.createInjector();
        Sample sample = injector.getInstance(Sample.class);
        sample.hello();
    }

}
```