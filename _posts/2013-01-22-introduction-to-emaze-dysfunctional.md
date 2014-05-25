title: Introduction to Emaze Dysfunctional
date: 2013-01-22 14:30
tags: java, functional

[Emaze Dysfunctional](https://bitbucket.org/emaze/emaze-dysfunctional) is a Java library providing the support for the functional programming, released by [Emaze Networks](http://emaze.net) under BSD license. Developed in accordance with [solid principles](http://en.wikipedia.org/wiki/SOLID_%28object-oriented_design%29) of OOP, it's distinguished from other similar libraries for simplicity, robustness and high reusability of its components.

I will describe the main features of the current version 5.2 through some demonstrative examples, taken from the "classical" literature.

## Quicksort

The following fragment implements the [quicksort](http://en.wikipedia.org/wiki/Quicksort) algorithm, choosing, for simplicity, the first item as the pivot.

    :::java
    static <T> Iterator<T> quicksort(Iterator<T> items, BinaryPredicate<T, T> order) {
        if (!items.hasNext()) {
            return items;
        }
        final T head = items.next();
        final List<T> tail = Consumers.all(items);
        return Multiplexing.chain(
                quicksort(Filtering.filter(tail, Dispatching.rcurry(order, head)), order),
                Iterations.iterator(head),
                quicksort(Filtering.filter(tail, Logic.not(Dispatching.rcurry(order, head))), order));
    }

The function, in addition to the iterator with the items to sort, takes an argument of type `BinaryPredicate`, one of several [functors](http://en.wikipedia.org/wiki/Function_object) provided by the library. It belong to the predicates family, that is those functions that return a boolean value. The "binary" prefix means that it's a predicate with ariety 2, infact it receives two values and retorns *true* if these are in order.

The available functors, intended to become your best friends, in order of arity (from 0 to 3) are: 

* **Procedures**: `java.lang.Runnable` (from the standard library), `Action`, `BinaryAction` e `TernaryAction`;
* **Predicates**: `Proposition`, `Predicate`, `BinaryPredicate`, `TernaryPredicate`;
* **Generics**: `Provider`, `Delegate`, `BinaryDelegate`, `TernaryDelegate`.

As you can see a `QuadruplePredicate` doesn't exist, indeed *Dysfunctional* is developed following the *one, two, three, too many!* principle. If you need a fourth parameter, probably, or there are some design errors, or it could be the case of grouping related parameters into a container object.

<!-- more -->

Looking the function body it could observe the usage of the `Consumers` façade, that exposes useful methods to "consume" the elements of an iterator, iterable or array.
The code of `Consumers.all(items)` yields all elements remained of the iterator and inserts them in the output list.

The `Multiplexing` façade handles operations such as "many to one" or vice versa. In this case, `Multiplexing.chain(...)` returns an iterator that concatenates the three ones received.

Then, the quicksort gives back the elements lower than the pivot sorted recursively, the pivot, and the remaining elements, always sorted recursively. The selection of the elements lower than the pivot is carried out by the `Filtering.filter(tail, Dispatching.rcurry(order, head))` statement, which filters the list of objects holding the `Dispatching.rcurry(order, head)` predicate.

The `rcurry` method stands for "right curry", the [partial application ](http://www.haskell.org/haskellwiki/Partial_application) (not  [currying](http://www.haskell.org/haskellwiki/Currying)) of `head` to the second argument of the `order` predicate. In practice, it returns a unary predicate equivalent to:

    :::java
    new Predicate<T>() {
      public boolean accept(T value) {
        return order.accept(value, head);
      }
    }

After it's necessary to concatenate the pivot, and given that `chain` concatenates iterators I used `Iterations.iterator(head)` to create an iterator that provides only one element (one could achieve the same effect with `Arrays.asList(head).iterator()`, but so the code is more clear).

The third argument of the `chain` is similar to the first, but to get the remaining elements the predicate is inverted by the `Logic.not` method.

The function, finally, can be used in the following way:

    :::java
    @Test
    public void sortingANonSortedList() {
        final BinaryPredicate<Integer, Integer> ascOrder = new BinaryPredicate<Integer, Integer>() {
            public boolean accept(Integer a, Integer b) {
                return a < b;
            }
        };
        final List<Integer> sequence = Arrays.asList(5, 2, 6, 9, 3, 2, 7, 1, 8);
        final List<Integer> sorted = Consumers.all(Quicksort.sort(sequence.iterator(), ascOrder));
        Assert.assertArrayEquals(new Integer[]{1, 2, 2, 3, 5, 6, 7, 8, 9}, sorted.toArray());
    }
