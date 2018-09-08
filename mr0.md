---
layout: doc
title: "Map Reduce 0 - Lab Edition"
submissions:
- title: Entire Assignment
  due_date: 10/26 11:59pm
  graded_files:
  - mr0.c
learning_objectives:
  - Interprocess Communication (IPC)
  - Pipes
  - Files and File Descriptors
  - MapReduce
  - Jeff Dean
wikibook:
  - "Pipes, Part 1: Introduction to pipes"
  - "Pipes, Part 2: Pipe programming secrets"
---

## MapReduce

In 2004, Google released a general framework for processing large data sets on clusters of computers.
We recommend you read [this link](http://en.wikipedia.org/wiki/MapReduce) on Wikipedia for a general understanding of MapReduce.
Also, [this paper](http://static.googleusercontent.com/media/research.google.com/zh-CN/us/archive/mapreduce-osdi04.pdf) written by Jeffrey Dean and Sanjay Ghemawat gives more detailed information about MapReduce.
However, we will explain everything you need to know below.

To demonstrate what MapReduce can do, we'll start with a small dataset--three lines of text:

```
Hello
there
class!
```

The goal of this MapReduce program will be to count the number of occurrences of each letter in the input.

MapReduce is designed to make it easy to process large data sets, spreading the work across many machines. We'll start by splitting our (not so large) data set into one chunk per line.

|           | Chunk #1 | Chunk #2 | Chunk #3 |
| --------- | -------- | -------- | -------- |
| **Input** | "Hello"  | "there"  | "class!" |


**Map**. Once the data is split into chunks, `map()` is used to convert the input into `(key, value)` pairs.
In this example, our `map()` function will create a `(key, value)` pair for each letter in the input, where the key is the letter and the value is 1.

|            | Chunk #1 | Chunk #2 | Chunk #3 |
|------------|----------|----------|----------|
| **Input**  | "Hello"  | "there"  | "class!" |
| **Output** | `(h, 1)` | `(t, 1)` | `(c, 1)` |
|            | `(e, 1)` | `(h, 1)` | `(l, 1)` |
|            | `(l, 1)` | `(e, 1)` | `(a, 1)` |
|            | `(l, 1)` | `(r, 1)` | `(s, 1)` |
|            | `(o, 1)` | `(e, 1)` | `(s, 1)` |

**Reduce.** Now that the data is organized into `(key, value)` pairs, the `reduce()` function is used to combine all the values for each key.
In this example, it will "reduce" multiple values by adding up the counts for each letter.
Note that only values for the same key are reduced.
Each key is reduced independently, which makes it easy to process keys in parallel.


|            | Chunk #1 | Chunk #2 | Chunk #3 |
|------------|----------|----------|----------|
| **Input**  | `(h, 1)` | `(t, 1)` | `(c, 1)` |
|            | `(e, 1)` | `(h, 1)` | `(l, 1)` |
|            | `(l, 1)` | `(e, 1)` | `(a, 1)` |
|            | `(l, 1)` | `(r, 1)` | `(s, 1)` |
|            | `(o, 1)` | `(e, 1)` | `(s, 1)` |
| **Output** | `(a, 1)` |          |          |
|            | `(c, 1)` |          |          |
|            | `(e, 3)` |          |          |
|            | `(h, 2)` |          |          |
|            | `(l, 3)` |          |          |
|            | `(o, 1)` |          |          |
|            | `(r, 1)` |          |          |
|            | `(s, 2)` |          |          |
|            | `(t, 1)` |          |          |

MapReduce is useful because many different algorithms can be implemented by plugging in different functions for `map()` and `reduce()`.
If you want to implement a new algorithm you just need to implement those two functions.
The MapReduce framework will take care of all the other aspects of running a large job: splitting the data and CPU time across any number of machines, recovering from machine failures, tracking job progress, etc.

## The Lab

For this Lab, you have been tasked with building a simplified version of the MapReduce framework.
It will run multiple processes on one machine as independent processing units and use IPC mechanisms to communicate between them.
`map()` and `reduce()` will be programs that read from standard input and write to standard output.
The input data for each mapper program will be lines of text.
Key/value pairs will be represented as a line of text with ": " between the key and the value:

```
key1: value1
key two: values and keys may contain spaces
key_3: but they cannot have colons or newlines
```


## Version 0 - one mapper, one reducer

For your initial implementation, start with one mapper process and one reducer.

    input_file
        |
       MAP
        |
     REDUCER
        |
    output_file


Command line:

    mr0 <input_file> <output_file> <mapper_executable> <reducer_executable>

Sample Usage:

```
% ./mr0 test.in test.out my_mapper my_reducer
my_mapper exited with status 1
my_reducer exited with status 2
output pairs in test.out: 9
```

You won't implement your own `map()` or `reduce()` function--your program will take the names of a map program and a reduce program on the command line and run those.

### Close all unused file descriptors!

The mapper and reducer processes won't exit until they reach the end of their input file.
An `EOF` won't be triggered on their input file until all processes that have a copy of their input file descriptor close that file descriptor.

For example, if the main process doesn't close its copy of the write end of the pipe that the reducer is reading from, the reducer will never see an EOF and will never exit.
In each child process created with `fork()` you'll also need to close all unused file descriptors inherited from the parent process.
To aid you in this, we've provided a set of functions, declared in `common.h`, that make it easy to keep track of all the additional file descriptors you create, and to close them all.

```
void descriptors_add(int fd);
void descriptors_closeall();
void descriptors_destroy();
```

You don't have to use these, but they may make the project easier.

You also should consider using the function `pipe2()` to create your pipes.
If you use `pipe2()`, you can set a flag `O_CLOEXEC` which will instruct the system to close both ends of the pipe if a call to `exec` is made.
See the man pages for `pipe2()` and `open()` for more information.

Since most of the child processes in this program have their `stdin` and `stdout` redirected, you may wish to create a function for that.

### "Reference Implementation"

You can implement the equivalent of this program in a Unix shell quite easily:
```
% my_mapper < test.in | my_reducer > test.out
```


### Files used for grading:

  * mr0.c

### Things we will be testing for:
* Inputs of varying size
* Different types of mapper and reducer tasks
* No memory leaks and memory errors when running application

### Things that will not be tested for:
* Illegal inputs for either the mapper or reducer (Input data in a format other than as described above)
* Invalid mapper or reducer code (mappers or reducers that do not work)

##  Building and Running

### Building
This Lab has a very complicated `Makefile`, but, it defines all the normal targets.

```
make # builds provided code and student code in release mode
make debug # builds provided code and student code in debug mode
```

### Input Data
To download the example input files (books from [Project Gutenberg](https://www.gutenberg.org/)), use the `Makefile`:

```
make data
```

You should now see `data/dracula.txt` and `data/alice.txt` in your Lab folder

### Running Your Code
We have provided the following mappers:

  * `mapper_wordcount`
  * `mapper_lettercount`
  * `mapper_asciicount`
  * `mapper_wordlengths`

These each be used anywhere we specify `my_mapper` in these docs.

And the following reducers:

  * `reducer_sum`

These each be used anywhere we specify `my_reducer` in these docs.

For example, if you wanted to count the occurrences of each word in Alice in Wonderland, you can run and of the following

    ./mr0 data/alice.txt test.out ./mapper_wordcount ./reducer_sum

## Notes

* You may find `dup` and/or `dup2` helpful.
* You may **NOT** use `system()`
