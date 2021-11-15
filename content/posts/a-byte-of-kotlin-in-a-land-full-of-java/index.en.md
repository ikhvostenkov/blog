---
title: "A Byte of Kotlin in a Land Full of Java"
date: 2021-10-25
draft: true
---
## Intro
I came to the world of programming through Java path. And during my entire career I was programming 
mostly in Java. When Kotlin became more and more popular in programming world I wanted to answer 
the question is it worth to use Kotlin in Java project or not?
And it turned out, as always in engineering, that answer is it depends.
This post is the way of a company I was working for and our experience.
So let's have a journey from island of Java to island of Kotlin together.

## How was my first time?
{{< style "img { float: right; margin: 0.5em; width: 22%; }" >}}
![Kotlin Logo](./kotlin.png)
My first acquaintance with Kotlin was when it has such a weird logo.

Back then I wrote nothing more than "Hello, World!" project in Kotlin and only followed Kotlin
community to be informed about new features and changes. But ~7 years ago I tried first time
Kotlin in enterprise project for test automation API. Project was small and went for a while.

Only in 2017 after some iterations of rejection and acceptance among other engineers we introduced
Kotlin full time in our Java components in production running project. This was not a huge project:
500K LOC, but was quite a while in production: for 12 years.
{{< /style >}}

## Main kotlin features
Kotlin is elegant, pragmatic and tool friendly language, but elegance is great, as long as
it's pragmatic. It is not a research project, but the language which was build for engineers
by engineers.
{{< style "td,th,thead,table { border: none;  background-color: transparent; text-align: center; } img {  width: 65%; }"  >}}
| ![cut](./cut.png) | ![shield](./shield.png) | ![extension](./extension.png) | ![wrench](./wrench.png) |
| --- | --- | -- | --- |
| **CONCISE** | **SAFE** | **INTEROPERABLE** | **TOOL-FRIENDLY** |
| *Drastically reduce the amount of boilerplate code* | *Avoid entire classes of errors such as null pointer exceptions* | *Leverage existing libraries for the JVM, Android and the Browser* | *Choose any Java IDE or build from the command line* |
{{< /style >}}

## Kotlin ecosystem
One of the magic of Java - is its backwards compatibility, but it's also a big bottleneck,
which limits some features and makes some features implemented in the inefficient way.
Kotlin was designed with keeping in mind all drawbacks of Java, but being 100% interoperable with it.
{{< style "img { display:block; margin: auto; }" >}}
![Kotlin Lib](./kotlin-lib.svg)
![No Kotlin SDK](./no-kotlin-sdk.svg)
{{< /style >}}

## What ∆ do we expect?
Why we decided to introduce kotlin?

In our project we had:
- A lot of Java code which was written starting from Java 1.4
- Lot of entities which are changing
- Fit enough with boilerplate code and kludges

We would like to get:
- Better code reading
- Faster feature creation
- Less routing coding
- More expressive

What does Kotlin give us:
- Code is shorter
- Null-safety
- Automatic type inference
- Great interoperability with Java
- Extension methods
- Ideal delegation

This is really all that pluses which Kotlin provides. This is what we were trying to achieve
during the migration and this is what basically we got.

## Disadvantages?
Everything looks so sweet by far. But is there a dark side?

Actually yes and I should say about it. If you like ternary operator, you should forget about it.
And there some things you should be ready for: you will be not able to assign int to long variable,
nothing will be cast automatically. You would lose some Intellij features. And you would have
some bugs. Together with less code and better readability you will also get some noise.
But still we have more advantages.

## Code style
Important preparation step to the migration is Code Style. A new code style adoption might be a very
natural process, if it starts with a new project, when there's no code formatted in the old way.
Changing formatting in an existing project is a far more demanding task, and should probably
be started with discussing all the caveats with the team.
{{< style "img { display:block; margin: auto; }" >}}
![Code Style Inspections](./code-style-inspections.png)
{{< /style >}}

### Coding conventions
There is for quite a long time like from 2018 so-called coding Kotlin conventions, but if you
started before it or before Kotlin version 1.3, then you do not have all it in your project.
But fortunately you have a special flag you can enable in maven or gradle which helps to
synchronise your team.
{{< style "img { display:block; margin: auto; }" >}}
![Code Style Inspections](./code-style-inspections.png)
{{< /style >}}
{{< labelled-highlight lang="xml" filename="pom.xml" >}}
<properties>
<kotlin.code.style>official</kotlin.code.style>
</properties>
{{</ labelled-highlight >}}

{{< labelled-highlight lang="gradle" filename="gradle.properties" >}}
kotlin.code.style=official
{{</ labelled-highlight >}}

### Check style
[KTLINT](https://github.com/pinterest/ktlint)

[DETEKT](https://github.com/detekt/detekt)

Who fails the build if the check fails?

## Mixing Kotlin and JAVA
As Kotlin compiles to Java byte code you can have both Java and Kotlin code in one project, and you
can gradually add Kotlin to your existing project. Although blending Java and Kotlin works great, you
can't write idiomatic Java nor Kotlin code, when blending. So the goal should be to move completely to
Kotlin.
{{< style "img { display:block; margin: auto; }" >}}
![Kotlin Build Process](./kotlin-build-process.svg)
{{< /style >}}

## Strategies for adoption
We can live with both languages in production more or less good, but this leads us to other question how to adopt Kotlin
on practice? On the right picture you can see module with 2 source folders Java and Kotlin respectively usually
```kotlinc``` invoked first to compile Kotlin files, then javac to compile Java on demand if there are dependency from
Kotlin to JAVA.
{{< style "td,th,thead,table { border: none;  background-color: transparent; text-align: left; } img { margin: 0.5em; } "  >}}
| ![Dependency Map](./dependency-map.png) | ![JAVA Kotlin Tree](./java-kotlin-tree.png) |
| --- | --- |
{{< /style >}}
- Use Module Dependency Graph as a map
- Build tools: first invoke ```kotlinc```, then ```javac```
- Kotlin compiler knows when to invoke ```javac```

## Conversion strategies
The next question is how to convert your code systematically:
- One more systematic way is to go random and wild, and do it along the task it hands. From the technical point of view
  this is the worst approach, but it is the best following the business value.

Here you can see to incremental systematic approaches:
- Inside-out, you take your dependency graph and first convert your innermost modules going outside.
  {{< style "img { display:block; margin: auto; }" >}}
  ![Dependency Map Inside-out](./dependency-map-inside-out.png)
  {{< /style >}}
- And outside-in you start from outside your graph and then go to the
  innermost:
  {{< style "img { display:block; margin: auto; }" >}}
  ![Dependency Map Outside-in](./dependency-map-outside-in.png)
  {{< /style >}}
