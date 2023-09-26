---
title: Singleton Pattern
type: docs
weight: 1
toc: true
sidebar:
  open: true
params:
  editURL:
---

## Simple singleton with Draconian synchronization

```java
public class Singleton {
    private static Singleton instance;
    private Singleton() {}

    public static synchronized Singleton getInstance() {
        if(instance == null) {
            this.instance = new Singleton();
        }
        return this.instance;
    }
}
```

Thread Safe. Locks will be acquired everytime the getInstance method is called. Acquiring and releasing locks unnecessarily could result in performance drawback.

## Singleton with Double Check Locking

```java
public class Singleton {
    private static volatile Singleton Instance;
    private Singletion(){}

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized(Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

Even though the double-checked locking can potentially speed things up, it has at least two issues:

- since it requires the volatile keyword to work properly, it's not compatible with Java 1.4 and lower versions
- it's quite verbose and it makes the code difficult to read

## Alternatives

1. Early Initialization

```java
public class Singleton {
    private static final Singleton INSTANCE = new Singleton();

    private Singleton() {}
    public static Singleton getInstance() {
        return INSTANCE;
    }
}
```

2. Init on Demand

```java
public class Singleton {
    private Singleton() {}

    private static class InstanceHolder {
        private static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return InstanceHolder.INSTANCE
    }
}
```