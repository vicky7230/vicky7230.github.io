---
layout: post
title: Demystifying buffer() operator in Kotlin flows
date: 2024-02-25 11:05:55 +0300
image: /assets/images/blog/flow-entities.png
destination: https://dev.to/vicky7230d/demystifying-buffer-operator-in-kotlin-flows-45jc
author: Vipin Kumar
tags: Kotlin-coroutines
---

I have been trying to understand flows and operators. During this time I cam to know about an operator called `buffer()`

The purpose of using this operator is to improve performance in case the producer is producing values faster than the consumer can consume.
Let us take an example of the below code:

```
class MainActivity : AppCompatActivity() {

    val TAG = MainActivity::class.java.simpleName

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val job = GlobalScope.launch {
            val time = measureTimeMillis {
                producer()
                    .collect {
                        delay(1500)
                        Log.d("FLOWS", "Item collected ${it.toString()}")
                    }
            }

            Log.e("FLOWS", "Took time: $time")
        }

    }

    fun producer() = flow<Int> {
        listOf(1, 2, 3, 4, 5).forEach {
            delay(1000)
            Log.d("FLOWS", "Emitting item ${it.toString()}")
            emit(it)
        }
    }

}
```

This code shows a `producer()` which is a flow that emits an integer every 1 second from a list of 5 integers.
I have added a consumer(`collect{}`) which consumes each emitted value in 1.5 seconds. 
If we run the above code we will get the below output:

```
10:47:08.429  D  Emitting item 1
10:47:09.937  D  Item collected 1
10:47:10.939  D  Emitting item 2
10:47:12.441  D  Item collected 2
10:47:13.443  D  Emitting item 3
10:47:14.945  D  Item collected 3
10:47:15.948  D  Emitting item 4
10:47:17.450  D  Item collected 4
10:47:18.453  D  Emitting item 5
10:47:19.955  D  Item collected 5
10:47:19.955  E  Took time: 12554
```

Notice that the next item is only emitted by the producer after the producer consumes the old item first. And hence the emission and consumption is a sequential operation. Since each emission and consumption of an item takes 2.5 seconds, the total time taken here is 2.5\*5 =12.5.

> Normally, flows are sequential. It means that the code of all operators is executed in the same coroutine.

So during the time when the consumer is busy consuming the item the producer cannot produce more. The producer can only produce new item after the old item has been consumed by the consumer.

### How can we improve the performance of this code:

If we can still somehow produce items during the time the consumer is consuming we can improve the performance.
Let us see how we can achieve using buffer:
Add `buffer()` operator with a capacity of 3:

```
class MainActivity : AppCompatActivity() {

    val TAG = MainActivity::class.java.simpleName

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val job = GlobalScope.launch {
            val time = measureTimeMillis {
                producer()
                    .buffer(3)
                    .collect {
                        delay(1500)
                        Log.d("FLOWS", "Item collected ${it.toString()}")
                    }
            }

            Log.e("FLOWS", "Took time: $time")
        }

    }

    fun producer() = flow<Int> {
        listOf(1, 2, 3, 4, 5).forEach {
            delay(1000)
            Log.d("FLOWS", "Emitting item ${it.toString()}")
            emit(it)
        }
    }

}
```

Now run the code and see the output:

```
10:53:36.258  D  Emitting item 1
10:53:37.265  D  Emitting item 2
10:53:37.766  D  Item collected 1
10:53:38.268  D  Emitting item 3
10:53:39.268  D  Item collected 2
10:53:39.270  D  Emitting item 4
10:53:40.272  D  Emitting item 5
10:53:40.770  D  Item collected 3
10:53:42.272  D  Item collected 4
10:53:43.774  D  Item collected 5
10:53:43.779  E  Took time: 8575
```

Observe that the items are emitted before the consumer consumes the old items. And hence we saved some time, the total execution time now is around 8.5 seconds. This is because we are using buffer.

The buffer operator creates a separate coroutine during execution for the flow it applies to .It will use two coroutines for execution of the code. A coroutine Q that calls this code is going to execute collect, and the code before buffer will be executed in a separate new coroutine P concurrently with Q. A channel is used between the coroutines to send elements emitted by the coroutine P to the coroutine Q. If the code before buffer operator (in the coroutine P) is faster than the code after buffer operator (in the coroutine Q), then this channel will become full at some point and will suspend the producer coroutine P until the consumer coroutine Q catches up. The capacity parameter defines the size of this buffer.

For more info [Reference](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/buffer.html#:~:text=If%20the%20code%20before%20buffer,the%20size%20of%20this%20buffer)!

[Originally posted here](https://dev.to/vicky7230d/demystifying-buffer-operator-in-kotlin-flows-45jc)
