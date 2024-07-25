---
layout: post
title:  "Null-safe аннотации в java"
date:   2024-07-25 23:41:34 +0300
categories: [java]
---

# Проблема

Java изначально не предложила никакого решения для гарантии not null,
и со временем сообщество привнесло множество своих решений на основе аннотаций.

В данной статье я сравниваю три решения (`javax.annotation`, `org.jetbrains.annotations`,
`org.springframework.lang`) и выбираю наиболее подходящее.

# TL;DR
Независимо от того, пишем библиотеку или приложение, независимо от Java/Kotlin, наиболее
консистентный вариант для контрактов:
- `@javax.annotation.Nonnull` - помечаем типы и результаты методов, которые не могут быть null
- `@javax.annotation.Nonnull(when = When.MAYBE)` - помечаем типы и результаты методов, которые могут
  быть null (MAYBE предпочтительнее т.к. двойное отрицание очень запутывает)

# Результаты исследований

1. Аннотации из `org.springframework.lang` являются мета-аннотациями к аннотациям 
   `javax.annotation` (`org.google.code.findbugs:jsr305:3.0.2`). Что нужно держать в уме:
   если этими аннотациями вы размечаете свой библиотечный код, то kotlin-проекты, которые
   импортируют вашу библиотеку, должны будут включить поддержкку мета-аннотаций в build.gradle
   ```
   kotlin {
      compilerOptions {
         freeCompilerArgs.addAll("-Xjsr305=strict")
      }
   }
   ```
2. Kotlin, при вызове java-метода, классифицирует результат на три возможных типа 
   (в разрезе nullability)
   1. Not-nullable (управляется вызываемой стороной) - `T`
   2. Nullable (управляется вызываемой стороной) - `T?`
   3. Unknown (управляется вызывающей стороной) - `T!`. Тип `T!` в языке не существует, такая
   запись лишь показывает что при присваивании результата java-метода значения kotlin- типу,
   этот тип может быть либо `T` либо `T?`.
3. Аннотации из разных библиотек неочевидно влияют на распределение по типам. Пример в таблице

   | Тип | org.jetbrains.annotations | javax.annotation                                                |
   |-----|---------------------------|-----------------------------------------------------------------|
   | T   | @NotNull                  | @Nonnull(when = When.ALWAYS) <br> or @Nonnull                   |
   | T?  | @Nullable                 | @Nonnull(when = When.NEVER) <br> or @Nonnull(when = When.MAYBE) |
   | T!  | (no annotation)           | @Nonnull(when = When.UNKNOWN) <br> or (no annotation)           |

4. Kotlin умеет понимать аннотации jsr305 (javax.annotation), даже если зависимости нет в classpath
