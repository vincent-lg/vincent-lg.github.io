---
layout: post
title: Using Python to toggle values in collections
tags: development python performance
---

Python has a lot of [builtin collections](https://docs.python.org/3/library/stdtypes.html), so many that it can be hard to choose which.  This article presents a simple use case (toggling values) and offers several collections with performance benefits.

## Python: how to toggle values in a collection?

Choosing the right collection in Python is a matter of understanding the problem.  Well, Python isn't alone here, but it's perhaps more important in high-level languages like Python for two reasons:

- Selection: there are a lot of collections in Python.  Lists, dictionaries, sets... and there are even more in the [collections](https://docs.python.org/3/library/collections.html) module!
- Performance: choosing the right tool for the job can have a lot of consequences on performances, as we'll see.

This article focuses on a rather-common need: toggling values in a collection.  The idea is rather simple:

1. You store whatever you want in a collection.  It could be entered by the user, it could be numbers, it could be other things.
2. When adding an element to the collection, if it's already in it, it should be removed.
3. Otherwise it can be added.

In other words, if you have a collection like this:

    [1, 2, 3, 4]

And if you try to toggle `2`, then you'll get:

    [1, 3, 4]

... because `2` was already present and has been removed.  But if you want to toggle `8`, you'll get:

    [1, 3, 4, 8]

Another property of these collections is that toggling the same element twice will restore the collection (it would have the same element it had before the first toggle).

Don't see a good use case?  Then just try to solve the problem as an exercise.  It's not hard... but it has several possibilities and not all of them are equal.

## Testing performances

To test our different solutions in terms of the time they require to run, we're going to use the [timeit](https://docs.python.org/3/library/timeit.html) module.  I personally don't much like inputting Python as strings and there's another option, though a bit less common:

In a new Python file (say `toggle.py`) add the following lines:

```python
from timeit import timeit
from typing import Callable

def time_function(func: Callable, *args, **kwargs):
    """Use timeit to measure a function's performance."""
    def inner():
        return func(*args, **kwargs)
    useconds = timeit(inner, number=1_000_000)
    print(f"Average run for {func.__name__!r}: {useconds:.3f} microseconds")
```

If you're not familiar with `timeit`, here's a more detailed explanation: we create a function, `time_function`, that takes a function as argument and optional arguments.  This function will create an inner function which doesn't use any argument.  This second function is useful because `timeit` can work with a callable... if it doesn't take any argument.  So to call our function with arguments, we create an inner function.

Another point worth reminding: `timeit` doesn't return the time in microseconds.  We've done a shortcut here, and it can be quite confusing, so I'll clarify (and I've spend enough time pondering over this): `timeit` returns the number in seconds.  `timeit` can run code several times (by default, one million times) and return the total execution time.  So to have the average time of one function, we should divide the result by one million.  That would give us the time (in seconds) to run one function.  But because we want to display it in microseconds (somewhat easier considering), we would multiply by one million.  In other words, we would divide by one million and multiply by one million.  So this calculation can be skipped altogether.

> How do we use that?  It doesn't look that intuitive.

Yet, I find it a bit easier to use than the native `timeit` call:

```python
time_function(print, "That's a line.")
```

Well, if you run this line... it will print "this line" one million times and then print the average time of a call to `print`.  Not very useful and it sure spams the console.

## Which collection to use?

Remember, the point of the exercise is to handle a collection of data and, in particular, the operation to "toggle" something, which will add it to the collection if not present, but will remove it if already present.

## A list

In the example in the next section I used a list to represent our collection.  Why not?  It might be your first thought anyway: let's use a list!

So let's create a list of, why, one thousand elements.  We'll put numbers in it, or rather, the numbers as strings (this is rather arbitrary, feel free to use any other element).

```python
from typing import List

# Create the list
in_list = [str(i) for i in range(1_000)]
```

That's a rather normal-looking [list comprehension](https://www.programiz.com/python-programming/list-comprehension).  If you're not familiar with this, it mostly comes down to the following lines:

```python
in_list = []
for i in range(1_000): # loop one thousand times, with i from 0 to 9999
    in_list.append(str(i))
```

Okay, so how to add an element if not present, remove it otherrise, in a list?  I offer you this first function, and that might be your first attempt too:

```python
def toggle_in_list(in_list: List[str], elt: str):
    """Add or remove the specified element from the list."""
    if elt in in_list:
        in_list.remove(elt)
    else:
        in_list.append(elt)
```

Sounds easy, right?  So how fast does it run?  Let's check with our `time_function`:

```python
time_function(toggle_in_list, in_list, "55")
```

This is roughly equivalent to: time (and print) the function `toggle_in_list(in_list, "55")` .

If you run the script, you might see something like this:

    Average run for 'toggle_in_list': 17.317 microseconds

Don't look at the number too closely.  It doesn't tell us much as is, and it will vary a lot depending on the machine you use to run the script.  Mark this number down and let's continue.

You might see a problem with the previous script: it browses the list twice: first, to find the element (`if elt in in_list:`), then to actually remove it if it's present.  Can't we do better?

We probably can and there are a few options.  Let's see one that uses the `index()` method.  `index()` returns the first index of the specified element... or raises an exception.

```python
def toggle_by_index_in_list(in_list: List[str], elt: str):
    """Toggle an element using its index in a list."""
    try:
        index = in_list.index(elt)
    except ValueError:
        in_list.append(elt)
    else:
        del in_list[index]
```

This function uses a `try/except` clause.  The advantage here is that we only browse the list once.  If we find anything we get the index (and just remove the element at this point).  If not we add the element at the end of the list.

Again, let's see what `time_function` says:

```python
time_function(toggle_by_index_in_list, in_list, "55")
```

And if you run it, you should see something like this:

    Average run for 'toggle_by_index_in_list': 12.286 microseconds

Haha, we have our first comparison!  So the second function is faster than the first by a rather wide margin (the second one runs about 30% faster.  This is not irrelevant in statistics.

Again, you might have different numbers to compare, but probably have a close result in comparison.

> I was told to avoid `try/except` blocks to optimize things?

Exceptions add overhead in Python when they're raised.  The frame (the context of this exception) has to travel backward.  This is not free.  But this is not the greatest danger altogether.

Python has an exception mechanism.  As a rule, I'd say to use it, but to try to ask (in an `if` statement) whenever possible rather than relying on an `except` catch, especially if this is to be called a lot.

In this context, notice that the `except` block could be called a lot indeed.  But also notice that `index` does raise an exception.  We might be able to speed things up by checking if `elt` is in `in_list`, but then we'll go back to the first solution.

As an example how we could make these performances much worse though, let's try to come up with a solution that doesn't involve any exception.  I warn you, it looks a bit hairy.  It uses Python generators and I'll comment on it a bit more:

```python
def toggle_in_list_with_generator(in_list: List[str], elt: str):
    """Toggle an element in a list, browsing the list in a generator."""
    indices = (i for i, already in enumerate(in_list) if already == elt)
    index = next(indices, None)
    if index is not None:
        del in_list[index]
    else:
        in_list.append(elt)
```

Don't clap too loudly, I'm blushing, though you can't see me.  As you'll see, I have no reason to be particularly proud of this function in terms of performance, but let's focus on the code first.

The first line of our function is a generator expression (between parenthesis).  The difference between list comprehensions and generator expressions might sound really irrelevant in terms of performance, but our generator (at this point) isn't consumed.  The list hasn't been browsed at all.

On the second line, we ask for the first element of our generator with `next`.  This will execute our generator but will stop at the first matching element.  Our generator should return indices of elements in the list with the same value.  And we ask `next` to return `None` if it consumes our generator and doesn't find anything.

The next part should be quite simple.  Again we remove if it exists, we add if it doesn't.

Notice that if the element doesn't exist, our entire list will be browsed in search of a matching index.

Okay, what does it say in terms of performance?

```python
time_function(toggle_in_list_with_generator, in_list, "55")
```

Run this script, get a cup of coffee and look at your inbox (if you're inbox is as full as mine, of course!):

    Average run for 'toggle_in_list_with_generator': 55.561 microseconds

WHAT?!

This beautiful function (if I say so myself) runs uncomparatively slower than the two other ones.  How come?

The reason is mostly that so far, we have used parts boosted by the C language, so we didn't exactly browse our list in Python.  That changed in this example, however, and we forced it to browse in Python rather than C.

So yes, we've got rid of exceptions, hurray!  But this function does run a lot slower.

Another try?  I'm sure there are other ways to solve this problem using lists, but lists aren't the only solution in Python and we should try to change our selection.

## A dictionary

When I first learned Python, lists seemed so beautiful, so easier to use than in other languages, so powerful and neat and with a decent syntax and methods that made total sense.  I used lists for about everything.  But here's the thing: lists aren't adapted to every task either.  In our case, lists can slow things down, as we've seen.

My second best bet would have been dictionaries.  So let's try them now.  A dictionary seems actually to make more sense, because it stores a key (a string, in our case) and a value (why not a boolean, to say if the key is in the dictionary or now)?

```python
from typing import Dict

in_dict = {str(i): True for i in range(1_000)}
```

If you were confused about list comprehensions, this might seem even more weird.  In terms of syntax however, it's pretty similar, but instead of building a list, we build a dictionary.  I find this bit of syntactic sugar to make much sense and be easy enough to understand.  Again, we create a dictionary with one thousand keys in it, each key being a number (the str-version of this number, actually), and the value being always `True`.

> Why `True`?

Well, this is just to indicate that the key is there, the toggle is on in our dictionary.  Again, you might think of better ways to do that, but let's work with dictionaries to begin with.

```python
def toggle_in_dict(in_dict: Dict[str, bool], elt: str):
    """Toggle the boolean value in a dictionary."""
    in_dict[elt] = not in_dict.get(elt, False)
```

That's one line, and that's good, but it's not a really simple line is it?  So what's happening here?

First we get the key in the dictionary, returnining `False` if not present.  Then we apply `not` to it, so that the value becomes the opposite (`True` if `False`, `False` if `True`) and finally we write it back in the dictionary.

> That sounds just like our first attempt with a list... won't we browse our dictionary twice, to get the value and write its opposite?

Nope.  Dictionaries are optimized to make access to a specific slot very quick, so no browsing is actually involved when we read or write, because we know the slot in which to read and write.

To prove it, let's run the `time_function` function again:

```python
time_function(toggle_in_dict, in_dict, "55")
```

And in your console, you might see something like this:

    Average run for 'toggle_in_dict': 0.274 microseconds

Wow!  It's not always good to compare numbers in performances, but 0.2 does seem a bit smaller than 17... or 12.  So this solution seems to be incredibly fast in comparison to a list.

As explained, when you read `in_dict['55']`, Python will not browse the entire dictionary to find the key `'55'`.  It will hash this key (which is to say, find a unique number representing `'55'`  and nothing else) and will manipulate this in all operations.  No browsing involved.  That's one reason why dictionaries can be so great in some situations.

> Why store a bool at all?  Couldn't we just remove from the dictionary if present, add if not present?

We could... but then arises the question: why use a dictionary at all?  Booleans aren't heavy in memory, but why even bother?  It's not like we need them.  What we need is a collection somewhat like a dictionary but only with keys... and no value.  Does that exist in Python?

## A set

A set wouldn't have been my first choice when I started learning Python.  And the reason was simple enough: I didn't know it ever existed.  And that's not to say "for a year I didn't know sets existed", I mean it more like "for 5 years with Python I didn't know about sets".  Sets are another collection, but due to their history (they weren't added so early in Python) and their mathematical application, they end up being less often used to solve a problem.  Which is a shame, because they're really good to solve some problems.

As a matter of fact, the first reason why I'm writing this post is I googled this very problem to see what others suggested.  Sets didn't come up in the first answers I found.

Sets are basically like dictionaries without values, only keys.  Sets are really useful to compare, well, sets of data that do not replicate.  If you add the same element twice to the set, only the first will be added.  Much like in a dictionaries, the same key cannot be added twice.  But unlike in a dictionary, keys aren't matched with values.  Sets have no values, only the presence (or absence) of the element matters.

Someone more familiar with mathematics than I (which is, admittedly, everyone or close enough) would find a lot of applications to sets right away, and would discover this collection much earlier while learning Python.  I, on the other hand, who stays away from math when there's a decent alternative found out about sets and their usefulness rather late.  But sets aren't only used for math, far from it.

To illustrate, let's go back to our example.  Let's create a set of the same elements (it will be somewhat similar to our `in_dict` dictionary, but there's no value, of course):

```python
from typing import Set

in_set = {str(i) for i in range(1_000)}
```

Even the syntax resembles dictionaries and this is source of confusion to beginners with sets.  The thing is, creating a set with, say, three elements in it, uses braces too:

```python
my_set = {1, 2, 3}
```

The difference in syntax between a dictionary and a set is that in a set, keys aren't matched with values, using a colon.  Each element is just separated from the others by a comma.

So here we use another form of list comprehension (set comprehension) to create our set.  It contains the same elements as our list and dictionary.

We have a few options to solve our problem with sets.  Let's try one first:

```python
def toggle_in_set(in_set: Set[str], elt: str):
    """Add or remove from a set."""
    if elt in in_set:
        in_set.discard(elt)
    else:
        in_set.add(elt)
```

Straightforward, easy to read and, hopefully, to understand.  If the element is in the set, we remove it, if not, we add it.  Notice that we use the `add` method (not `append`) because order doesn't matter in sets.  We don't worry too much about the order, what we want to know is whether an element is present... or not.

So let's measure this function in performance:

```
time_function(toggle_in_set, in_set, "55")
```

Running the script, you might see something like:

    Average run for 'toggle_in_set': 0.243 microseconds

Faster than dictionaries... but not by much.  As a matter of fact the difference in figures might not be relevant, statistically.  It's true than sets perform a bit faster, but the main advantage is, we don't store values (boolean values in our case), so the gain might be trivial in time and in memory usage, but the gain is here (don't create things you don't need to have, as a rule).

But there's another way to approach the same problem, and this uses a feature of sets which might sound a bit abstract at first.  Sets have [more methods](https://docs.python.org/3/library/stdtypes.html#set) than dictionaries, and some of them might sound quite cryptic.  `union`?  `intersection`?

As said, sets are useful to compare sets of data, to see what's different between two sets, what's common... and much more.  In our case there's an interesting method though.  Quoting from the documentation:

>   symmetric_difference(other)
    set ^ other
    Return a new set with elements in either the set or other but not both.

The line is short, but it says everything we need.  If in our first set, we have our current toggles, but in the other, we have the toggle to add or remove, what happens:

    >>> animals = {"dog", "cat", "rabbit", "hamster"}
    >>> animals ^ {"bat"} # It's not in the set
    {'dog', 'rabbit', 'bat', 'cat', 'hamster'}
    >>> animals ^ {"rabbit"} # It's in there already
    {'cat', 'hamster', 'dog'}
    >>>

As you can see, using the symmetric difference (`^` operator), if we mix two sets with different elements, we get a set with all elements in both sets... but if the element is present in both sets, we don't get this element in the final set.  Sounds like a good option for us!

```python
def toggle_in_set_by_symmetry(in_set: Set[str], elt: str):
    """Add or remove from a set by symmetry."""
    in_set ^= {elt}
```

And that's it!  One line!  So let's see what `time_function` says:

```python
time_function(toggle_in_set_by_symmetry, in_set, "55")
```

If you run the script you might see something like this:

    Average run for 'toggle_in_set_by_symmetry': 0.254 microseconds

Hum... not that bad but not that great.  Running it several times I actually find this last function between our first set approach and our dictionary in terms of performances.  Not that significant.

The advantage of the last method is its concision: it only needs one line.  But is it a simple line to read?

Greater performances aren't achieved because you put everything on the same line.  The shorter code isn't necessarily the fastest code.  On the other hand, this method does seem quick and short, though perhaps not as quick as the other using sets (not by much either).

## Conclusion

This article tried to present a problem and three different collections.  There are way, way more collections in Python than just three!  These all are builtin types, so you don't need to import anything to use either of them.  And Python has more in its [collections](https://docs.python.org/3/library/collections.html) module.  And that's not the end of the story.

What you should remember from this article though is quite simple:

1. Look at the problem before choosing your collection.  This will have impact both on performances and readability.  Performance isn't always a high concern, but readability should always be one, hopefully.
2. Not all collections will perform the same way to solve a problem.  The gain (or loss) in performance might be drastic.
3. The race to write less code doesn't make your code run faster, not necessarily.  Sometimes a more readable method is actually faster than a simple one-liner.

Performance is by no way static.  This article gave you numbers: don't take them at heart.  Numbers will vary on every machine.  They can be used to compare different methods... keeping in mind that they don't always tell you the whole story.  As a matter of fact, performance is a metric... but not the only one that you should worry about.  Optimized code might run faster, but if it's harder to maintain, you might pay a price in the long-run.  That's a matter of balance, as is usually the case.

## Full code

Below is the code of our Python script, with the `time_function` code and the different implementations:

```python
from timeit import timeit
from typing import Callable, Dict, List, Set

# Performances
def time_function(func: Callable, *args, **kwargs):
    """Use timeit to measure a function's performance."""
    def inner():
        return func(*args, **kwargs)
    useconds = timeit(inner, number=1_000_000)
    print(f"Average run for {func.__name__!r}: {useconds:.3f} microseconds")

# Using lists


in_list = [str(i) for i in range(1_000)]

def toggle_in_list(in_list: List[str], elt: str):
    """Add or remove the specified element from the list."""
    if elt in in_list:
        in_list.remove(elt)
    else:
        in_list.append(elt)

time_function(toggle_in_list, in_list, "55")

# Using index

def toggle_by_index_in_list(in_list: List[str], elt: str):
    """Toggle an element using its index in a list."""
    try:
        index = in_list.index(elt)
    except ValueError:
        in_list.append(elt)
    else:
        del in_list[index]

time_function(toggle_by_index_in_list, in_list, "55")

# Using a generator

def toggle_in_list_with_generator(in_list: List[str], elt: str):
    """Toggle an element in a list, browsing the list in a generator."""
    indices = (i for i, already in enumerate(in_list) if already == elt)
    index = next(indices, None)
    if index is not None:
        del in_list[index]
    else:
        in_list.append(elt)

time_function(toggle_in_list_with_generator, in_list, "55")

# Using a dictionary

in_dict = {str(i): True for i in range(1_000)}

def toggle_in_dict(in_dict: Dict[str, bool], elt: str):
    """Toggle the boolean value in a dictionary."""
    in_dict[elt] = not in_dict.get(elt, False)

time_function(toggle_in_dict, in_dict, "55")

# Using a set

in_set = {str(i) for i in range(1_000)}

def toggle_in_set(in_set: Set[str], elt: str):
    """Add or remove from a set."""
    if elt in in_set:
        in_set.discard(elt)
    else:
        in_set.add(elt)

time_function(toggle_in_set, in_set, "55")

# Using symmetry

def toggle_in_set_by_symmetry(in_set: Set[str], elt: str):
    """Add or remove from a set by symmetry."""
    in_set ^= {elt}

time_function(toggle_in_set_by_symmetry, in_set, "55")
```
