---
title: Why is Spring much slower than NestJs?
description: Comparing SpringBoot to NestJS
author: nasagong
date: 2025-02-24 14:10:00 +0800
categories: [Misc]
tags: [Spring, NestJs]
render_with_liquid: false
---

## Target Audience
> - Those who have experience with Spring Boot or NestJS  
> - Those who have a basic understanding of Dependency Injection
> - Those who are interested in how frameworks work under the hood

## 0. Intro

My first DI framework was NestJS, which I stumbled upon during my military service.  
At the time, I had just taken my first steps into the web development world through React, and a friend recommended that I give Nest a try.  
He also mentioned that it’s pretty similar to Spring, which is almost inevitable to encounter if you're a web developer based in Korea.  
If you've worked with both frameworks, you probably noticed pretty quickly how similarly they're abstracted.


Nest was a really helpful friend that made Spring's learning curve way easier for me.  
But as time went on and I got more comfortable with Spring, a question started to pop up in my mind.


### Why is Spring's cold start so slow?

Both frameworks are supposed to make DI easy, and while their underlying philosophies might differ a bit, from a user's perspective they feel really similar in how they work.  
But when it comes to startup time, the gap between them is huge.  
I couldn't find an exact benchmark, but based on my experience, Spring builds usually take anywhere from 10 seconds to even tens of seconds, while Nest rarely takes more than 5 seconds.  
Why is that?  
There are a lot of factors that come to mind, but if someone asked me to give a precise answer... honestly, I'd find it pretty hard to explain.


...

So, starting from this personal curiosity, I ended up doing some digging, and I’d like to briefly share a few things I learned along the way.




## 1. Language Limitations

I actually debated whether I should even write about this since it feels kind of obvious,  
but in a way, I think it’s the biggest factor — so I’ll try to keep it as simple as possible.


### 1) In the case of Spring

![](/assets/img/2025-02-25-21-39-29.png)

First off, when you’re running a JVM-based language, it’s unavoidable that your source has to be compiled down to bytecode.  
Why do we even need bytecode? Pretty simple — because it has to be loaded onto the JVM to run at all.  
And the JVM tends to have relatively high memory usage.  
On top of that, when the JVM boots up, it has to go through class loading, garbage collector initialization, and other processes,  
which naturally adds a ton of overhead.


### 2) In the case of NestJS

NestJS runs on Node, obviously, which means it’s sitting on top of V8 — way lighter than the JVM.  
Other than that, the only somewhat "heavy" task would be transpiling TypeScript down to JavaScript,  
but beyond that, there isn’t any extra complicated build process going on.

---
\
So, is this difference the main reason for the huge gap in startup times?  
If that were the case, the title of this post would've been something like *"JVM vs V8"* instead.  
In reality, there’s more to it — and the second reason I’m about to get into plays a pretty big role too.



<br/>

## 2. Difference in the Scope of Reflection

If you've ever taken even a slight interest in how Spring works under the hood,  
or if you've tried to inspect an object's information at runtime in other languages or frameworks,  
then the concept of Reflection probably feels pretty familiar to you.  
Before diving deeper, let’s quickly take a look at how Reflection is actually being used internally.


### In the case of Spring

![](/assets/img/2025-02-25-22-38-57.png)

Let’s take a quick look inside the `@SpringBootApplication` annotation you first encounter when creating a Spring Boot project.<br/>

Just because you add something like `@Controller` or `@Service` on top of a class  
doesn’t mean it automatically gets picked up and registered into the container.  
To actually use dependency injection — the core feature of Spring —  
the framework needs to scan ahead of time and gather the objects that will be injected.  
In simple terms, during the main method’s startup phase, it scans all classes to find which ones have certain annotations.<br/>

You could register beans at compile time using `@Bean`,  
but most of the time, for the sake of convenience, the decision about which beans to inject happens at runtime.  
Now, let’s take a look at the internals of the `@ComponentScan` annotation.



![](/assets/img/2025-02-25-23-08-09.png)

The code you can see in your project window is just an interface,  
so to really see where Reflection is being used, you have to dive into the actual Spring core code.  
You can check it [here](https://github.com/spring-projects/spring-framework/blob/main/spring-core/src/main/java/org/springframework/core/type/StandardAnnotationMetadata.java#L166).  
For space reasons, I’m only attaching one method, but internally, methods like the one above use Reflection to gather annotation information from classes,  
and they even explore inherited annotations by using the `SearchStrategy.INHERITED_ANNOTATIONS` option.

I haven’t looked at every single case, but overall, a big part of Spring DI’s core logic is built on top of Reflection.  
Even just thinking about it simply — Reflection has to load all the class information into memory at runtime,  
so naturally, it’s a very heavy operation.  
You could try to limit the scope using a base package setting or something,  
but fundamentally, it's not easy to dramatically reduce Reflection usage.

So then, how does NestJS handle this?


## In the case of NestJS

![](/assets/img/2025-02-25-23-29-45.png)

First, here’s a simple controller implementation from NestJS.  
The `@Controller()` decorator directly maps to what Spring’s `@Controller` does,  
and the `@Get()` decorator on the `findAll` method corresponds to Spring’s `@GetMapping`.  
Looking at the code, the structure feels really similar to Spring.  
But actually, there’s a mistake in the explanation above.

Nest’s `@Controller()` isn’t technically an annotation — it’s a **decorator**.  
And it’s not like they just slapped a different name on the same feature;  
there’s a pretty big difference between the two.

Let’s dive into the core code again to see what’s going on.

![](/assets/img/2025-02-25-23-36-51.png)
(You can find the actual code in `node_modules/@nestjs/core/scanner.ts`)

Obviously, you can’t figure out everything just by looking at this one line,  
but to put it simply, it’s a method that selectively pulls specific metadata using Reflection.

![](/assets/img/2025-02-25-23-40-50.png)

If you dig a little deeper, you’ll see that it’s set up to only collect metadata from ***specific decorators*** like `@Injectable()` and `@Controller()`.

When we looked at Spring earlier:

1. It scanned all classes first,  
2. Then checked if any of them had certain annotations before registering them in the container.

It’s a pretty natural flow.  
I mean, how could you claim to have discovered all target classes without brute-forcing through every class under the base package?  
Is TypeScript pulling off some kind of magic trick here?

This is where the difference between annotations and decorators really starts to stand out.


### Decorators are Module-Scoped

If Reflection is the main tool Spring uses for DI, then in NestJS, dependency resolution is based around modules.  
By combining decorators with its module system, NestJS can pinpoint target classes pretty precisely.

```ts
@Module({
  controllers: [CatsController],
  providers: [CatsService]
})
export class CatsModule {}
```

You explicitly register components inside a module file ahead of time,  
and when the app first boots up, all these modules get loaded into the root module — usually the AppModule.  
During this process, the decorators execute, and the metadata gets automatically registered as well.  
In short, decorators only operate on classes that have already been registered to a module,  
which means they’re module-scoped by design.

Because metadata is managed module-by-module like this,  
NestJS doesn’t have to scan everything with Reflection from beginning to end like Spring does.  
It can just narrow down the search range super fast by using the data that’s already been registered.<br/>

<br/>


## 3. Difference in AOP Approach

We also need to touch on AOP, one of Spring’s core features.  
Since the post is already getting long, I’ll skip the deep conceptual explanations and jump straight into the comparison.

```java
@Transactional
public void someMethod() {
   // ... 
}
```

One of the most iconic examples when explaining Spring AOP is `@Transactional`.  
It’s a super useful feature that lets you handle complex transaction management just by throwing an annotation on a method.  
But of course, for Spring AOP to actually catch methods marked with annotations like this, it needs to do a bunch of work under the hood.  
As part of that, [dynamic proxy](https://huisam.tistory.com/entry/springAOP) objects get created,  
and inside those proxies, yep — Reflection is used again.  
Naturally, it’s a pretty heavy operation.

```ts
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    // ...
    return next.handle(); // chaining to the next interceptor in the lifecycle
  }
}
```

On the other hand, NestJS skips the traditional AOP style and instead uses an `Interceptor` system based on function chaining to achieve separation of concerns.  
It explicitly registers interceptors without relying on Reflection,  
which cuts down a lot of overhead.

```ts
@UseInterceptors(LoggingInterceptor)
export class UserController {
  @Get()
  findAll() {
    return "someone..";
  }
}
```

By explicitly specifying interceptors like this,  
NestJS avoids the need to create dynamic proxies and do heavy Reflection work,  
giving it a performance advantage even in AOP scenarios.<br/>
<br/>


## 4. Closing Thoughts

Now that I look at what I wrote, it kind of sounds like I painted NestJS as this insanely fast, way superior tool compared to Spring.  
But like I mentioned at the beginning, the point of this post was just to satisfy my curiosity about *why* Spring's cold start feels so slow.

As with pretty much all software, most performance differences come down to trade-offs based on design goals.  
Spring is built for enterprise-level service development,  
so it uses things like `@ComponentScan` to provide broad flexibility —  
making sure developers don’t have to manually wire up the entire context themselves.  
And also, it's really just the cold start that's slow;  
once the runtime gets going, Spring internally optimizes itself over time,  
making it a pretty stable choice for running large-scale services.

On the flip side, Nest focuses on enabling faster development cycles and lightweight execution environments by leveraging the JS/TS ecosystem.  
That’s why it has faster cold starts,  
and it naturally shines in environments like serverless, where cold starts happen all the time.  
From my limited point of view, I feel like this advantage might be one of the reasons why startups —  
who usually need fast development cycles — often end up picking Node.

## References

- [https://docs.nestjs.com/modules](https://docs.nestjs.com/modules)

- [spring core](https://github.com/spring-projects/spring-framework/blob/main/spring-core/src/main/java/org/springframework/core/type/StandardAnnotationMetadata.java#L166)

- [https://docs.nestjs.com/interceptors](https://docs.nestjs.com/interceptors)

- [https://huisam.tistory.com/entry/springAOP](https://huisam.tistory.com/entry/springAOP)