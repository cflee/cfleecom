+++
title = "Using Java/Kotlin on LeetCode and Codeforces"
+++

Some rough notes.
WIP.

## Versions and config options

|  | [LeetCode](https://support.leetcode.com/hc/en-us/articles/360011833974-What-are-the-environments-for-the-programming-languages) | [Codeforces](https://codeforces.com/blog/entry/121114) |
|---|---|---|
| Java | 21 | 11 (32-bit), 17 (64-bit), 21 (64-bit) |
| Java JVM options | ? | `-XX:+UseSerialGC -XX:TieredStopAtLevel=1 -XX:NewRatio=5 -Xms8M -Xmx{MEMORY_LIMIT_MB}M -Xss64M -DONLINE_JUDGE=true` |
| Kotlin | 1.9 | 1.7 (64-bit), 1.9 (unclear) |
| Kotlin JVM options | ? | `-XX:NewRatio=5 -Xms8M -Xmx${MEMORY_LIMIT_MB}M -Xss64M -DONLINE_JUDGE=true -Duser.language=en -Duser.region=US -Duser.variant=US` (1.9) |

Some say that there is some negative impact to performance on certain code from moving to 64-bit, but I don't know the details of that.

## Thread stack size

If you run code locally, you should increase the JVM thread stack size from the defaults.

If you run with the default thread stack size, you may run into a StackOverflowError locally on deeply-recursive code that would execute fine on LeetCode/Codeforces.
I have run into this issue even on some LeetCode problems.

The [JDK 21 manpage for `java`](https://docs.oracle.com/en/java/javase/21/docs/specs/man/java.html) states that it defaults to 1024 KB on macOS/Linux 64-bit platforms and 2048 KB on macOS/Linux ARM 64-bit platforms.
I suppose that this could vary with JVM distribution.

Use the `-Xss64M` parameter when starting the JVM to increase it to 64 MB, this value chosen to match the Codeforces config.
I don't know what the Leetcode config is.

## Input/Output (Codeforces)

This seems to be a commonly mentioned problem on Codeforces.
I haven't run into anything that what I think are relatively 'standard' approaches can't handle.

By standard I mean the BufferedReader and InputStreamReader combo in Java, or `readln()` and `println()` in Kotlin.

But I have not worked on a problem that requires reading or writing many lines of text, so perhaps more on this in the future.

The [Kotlin docs' competitive programming tutorial](https://kotlinlang.org/docs/competitive-programming.html#functional-operators-example-long-number-problem)'s suggested helper functions are convenient.
I usually only have to supplement them with a `readLongs()`:

```kt
private fun readString() = readln()
private fun readInt() = readString().toInt()
private fun readStrings() = readString().split(" ")
private fun readInts() = readStrings().map { it.toInt() }
private fun readLongs() = readStrings().map { it.toLong() }
```

## Defending against slow sort of primitive arrays

In JDK 7 to 13, there was an edge case where under some circumstances sorting primitive arrays (e.g. `int[]`, `long[]`, `double[]`) could devolve to `O(n^2)` quadratic time due to a switch to quicksort.
See e.g. this Codeforces blog post "[Avoid getting TLE in Java while sorting Arrays](https://codeforces.com/blog/entry/104550)", which suggests as a workaround to make `Integer[]` arrays instead to get a mergesort done.

This has been fixed with an implementation change in JDK 14, where there were improvements to the Dual-Pivot Quicksort algorithm, and in particular switching to heapsort instead of quicksort when the execution time is become quadratic, guaranteeing `O(n log n)` time.
See [JDK-8226297](https://bugs.openjdk.org/browse/JDK-8226297) and the corresponding [commit on the github repo mirror](https://github.com/openjdk/jdk14/commit/de54eb15130848d4efc266891e41b978f444f9f3).

Just always use JDK 17 or 21 and you're all set for this.

It's no longer necessary to shuffle the arrays before sorting to avoid the pathological cases.

I haven't run into this issue myself, but it's almost folklore on Codeforces at this point and often cited in the "why not to use Java" posts or comments, so this is useful info to know.

## Defending against hash table collisions

There seems to be a bit of a sport on Codeforces of hacking (adverserially crafting test inputs to make others' solutions fail time limits) hash table based solutions with specially crafted inputs that cause all the data to land on the same hash bucket in a hash table.
This makes working with entries in the bucket take `O(n)` time instead of the `O(log n)` that you're expecting.
You'll often see this mentioned as the `unordered_map` problem in C++, see e.g. this classic post "[Blowing up unordered_map, and how to stop getting hacked on it](https://codeforces.com/blog/entry/62393)".

In Java, these are data structures like `HashSet`, `HashMap`.
In Kotlin, the default `Set` and `Map` implementations on JVM appear to be the Java [`LinkedHashSet` and `LinkedHashMap`](https://discuss.kotlinlang.org/t/listof-arraylistof-setof-etc/1475/5) and have all the same issues.

This has been improved since JDK 8, in [JEP-180: Handle Frequent HashMap collisions with Balanced Trees](https://openjdk.org/jeps/180), where buckets exceeding the default of 8 entries will be switched from a linked list to a balanced tree, and improve the worst-case time to `O(log n)`.

However, the requirement is that the _hash table keys must implement Comparable_.
This is fine if you're using Integers or Ints or Strings, but you must add it for your own Java classes or Kotlin data classes that are used as keys to benefit from this improved implementation.

Example of implementing `Comparable<T>` on an data class in Kotlin:

```kt
data class IntPair(val a: Int, val b: Int) : Comparable<IntPair> {
    override fun compareTo(other: IntPair) =
        compareValuesBy(this, other, { it.a }, { it.b })
}
```

Alternatively, you could always use the `TreeSet`/`TreeMap` type of data structures instead, even when you don't need the key ordering, but you will still have to implement `Comparable<T>` for the keys anyway as a prerequisite.

## Memoizing in HashMap vs arrays

I often reach for the pattern of using an object as key in a HashMap, to make a composite key for memoizing a recursive function, as a shortcut compared to wiring up the storage and retrieval through multiple layers of HashMaps.

Example in Kotlin:

```kt
data class Record(val a: Int, val b: Int, val c: Int)
val memo = mutableMapOf<Record, Int>()
fun recurse(a: Int, b: Int, c: Int): Int {
    return memo.getOrPut(Record(a, b, c)) {
        // do stuff
    }
}
```

However, this obviously involves a ton of object instantiations, even on the function invocations that are to be retrieved from the cache.
Much of the time it works out fine, but I have run into problems where this is simply too slow.
(Not that you would know how close you are from a Time Limited Exceeded error... but sometimes you gotta guess.)

In that case, it's necessary to store the memoized values in something else, like multi-dimensional arrays.
(Perhaps layers of lists/maps could work too, but in this case I went directly to arrays.)

Example in Kotlin:

```kt
val memo = Array(n) { Array(2) { IntArray(3) { -1 } } }
fun recurse(a: Int, b: Int, c: Int): Int {
    // array bounds checks
    if (memo[a][b][c] != -1) return memo[a][b][c]
    val result = // do stuff
    memo[a][b][c] = result
    return memo[a][b][c]
}
```
