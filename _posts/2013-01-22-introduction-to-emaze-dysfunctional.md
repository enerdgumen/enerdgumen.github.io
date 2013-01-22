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

## The N-queens problem

The [problem of the N-queens](http://en.wikipedia.org/wiki/Eight_queens_puzzle) is a classic puzzle that consists in placing N queens on a chessboard NxN without that they threaten each other. With languages ​​that natively support the functional constructs, like [Scala](http://www.scala-lang.org/), the problem can be solved with less than 10 lines of code (without loss of clarity), with Java of course it's a bit more tricky.
I propose the classic [backtracking algorithm](http://moritz.faui2k3.org/en/backtracking), developed with a functional object-oriented approach, using *Dysfunctional* extensively.

    :::java
    public static List<List<Integer>> queens(int n) {
        final List<Pair<Integer, Integer>> queenMoves = Arrays.asList(
                Pair.of(-1, -1),
                Pair.of(-1, 0),
                Pair.of(-1, 1));
        final Mover mover = new Mover(queenMoves);
        final IsQueenSafe isQueenSafe = new IsQueenSafe(mover);
        final PlaceNextQueens placeNextQueens = new PlaceNextQueens(isQueenSafe);
        final Iterable<Integer> columns = new Ranges(new ComparableComparator<Integer>(), new NextIntegerSequencingPolicy(), 0).rightHalfOpen(0, Maybe.just(n));
        return new PlaceQueens(Dispatching.curry(placeNextQueens, columns)).perform(n);
    }

The `queens` method receives the board size and returns the list of the possible solutions (a solution is a list of integers, representing the configuration of the queens on the board), the body doesn't directly implement the algorithm, but rather it configures properly the components involved which then delegates the searching for solutions.

* `Mover` is a binary function that, received the row and column where a queen is placed, returns a list of all the possible positions reachable from it (an iterator of pairs of row/column);
* `IsQueenSafe` is a predicate that, received the actual queens configuration and a column index, returns true if a queen placed in such column is safe (she isn't captured by any other queen);
* `PlaceNextQueens` is a binary function that, given a list of columns and the actual queens configuration, tries to add a queen on each column and returns the list of the configurations in which the new queen is safe;
* `PlaceQueens` is the function implementing the recursive search of the solutions.

### Moving the queen

    :::java
    public class Mover implements BinaryDelegate<Iterator<Pair<Integer, Integer>>, Integer, Integer> {

        // iterable of pairs where the former is the row increment, and the latter is the column increment
        private final Iterable<Pair<Integer, Integer>> moves;

        public Mover(Iterable<Pair<Integer, Integer>> moves) {
            this.moves = moves;
        }

        public Iterator<Pair<Integer, Integer>> perform(final Integer row, final Integer column) {
            return Multiplexing.chain(Applications.transform(moves, new Delegate<Iterator<Pair<Integer, Integer>>, Pair<Integer, Integer>>() {
                public Iterator<Pair<Integer, Integer>> perform(Pair<Integer, Integer> move) {
                    return new MoveIterator(move, row, column);
                }
            }));
        }
    }

The object is initialized with a sequence of pairs, each of them represents a possible move of the piece (in terms of variation of the position of row/column). For example, the pair (-1, 0) enables the vertical moviment.  
The responsability for moving the pieace is delegated to the following `MoveIterator`, that is the only component with state of the whole algorithm. The iterator applies the variation of position relative to the current one, until all the space on the board is consumed.

`Mover` uses a new function: `Applications.trasform`. It returns an iterator where the next element is obtained by applying the provided unary function to the original item, in other words it's the conventional `map` that operates over iterators in a lazy way.

The result is an iterator containing all the positions reachable from the queen.

    :::java
    public class MoveIterator extends ReadOnlyIterator<Pair<Integer, Integer>> {

        private final Pair<Integer, Integer> move;
        private int row;
        private int column;

        public MoveIterator(Pair<Integer, Integer> move, int row, int column) {
            this.move = move;
            this.row = row;
            this.column = column;
        }

        public boolean hasNext() {
            return row + move.first() >= 0
                    && column + move.second() >= 0;
        }

        public Pair<Integer, Integer> next() {
            row += move.first();
            column += move.second();
            return Pair.of(row, column);
        }
    }

Finally `MoveIterator` extends `ReadOnlyIterator`, an iterator that doesn't support the method `delete`.

### Is the queen safe?

    :::java
    public class IsQueenSafe implements BinaryPredicate<List<Integer>, Integer> {

        private final BinaryDelegate<Iterator<Pair<Integer, Integer>>, Integer, Integer> mover;

        public IsQueenSafe(BinaryDelegate<Iterator<Pair<Integer, Integer>>, Integer, Integer> mover) {
            this.mover = mover;
        }

        public boolean accept(final List<Integer> queens, final Integer column) {
            final int row = queens.size();
            final Iterator<Pair<Integer, Integer>> positions = mover.perform(row, column);
            return Filtering.filter(positions, new Predicate<Pair<Integer, Integer>>() {
                public boolean accept(Pair<Integer, Integer> position) {
                    return queens.get(position.first()) == position.second();
                }
            }).hasNext() == false;
        }
    }

To check whether a queen is safe at a given location must "move" it and see if it meets another queen, so this component uses the `mover` collaborator to get all the possible reachable positions. Informally, the queen is in position `(row, column)` when `queens[row] == column`, therefore the predicate checks this situation.  
Note the use of the `hasNext` as a result of the function, which takes advantage lazy evaluation of iterators: the function returns *false* when it detects the first queen.

### Adding a queen

    :::java
    public class PlaceNextQueens implements BinaryDelegate<List<List<Integer>>, Iterable<Integer>, List<Integer>> {

        private final BinaryPredicate<List<Integer>, Integer> isQueenSafe;

        public PlaceNextQueens(BinaryPredicate<List<Integer>, Integer> isQueenSafe) {
            this.isQueenSafe = isQueenSafe;
        }

        public List<List<Integer>> perform(Iterable<Integer> columns, List<Integer> queens) {
            final Iterator<Integer> safeColumns = Filtering.filter(columns, Dispatching.curry(isQueenSafe, queens));
            return Applications.map(safeColumns, Dispatching.curry(new AddToList<Integer>(), queens));
        }
    }

It receives the actual configuration of the queens and a list of columns on which to try to place a new queen on the next row, and returns all the possible configurations in which the new queen is safe.  
To implement it in functional way I've used the functor `AddToList`, actually unavailable in *Dysfunctional*. Given a list and an element it returns a new list with the new item added.

    :::java
    public class AddToList<T> implements BinaryDelegate<List<T>, List<T>, T> {

        public List<T> perform(List<T> list, T item) {
            final List<T> result = new LinkedList<T>(list);
            result.add(item);
            return result;
        }
    }

The list of columns is obtained generating through the façade `Ranges` an iterable from `0` to `n-1`:

    :::java
    final Iterable<Integer> columns = new Ranges(
        new ComparableComparator<Integer>(),
        new NextIntegerSequencingPolicy(),
        0
    ).rightHalfOpen(0, Maybe.just(n));

### Finding all solutions

    :::java
    public class PlaceQueens implements Delegate<List<List<Integer>>, Integer> {

        private final Delegate<List<List<Integer>>, List<Integer>> placeNextQueens;

        public PlaceQueens(Delegate<List<List<Integer>>, List<Integer>> placeNextQueens) {
            this.placeNextQueens = placeNextQueens;
        }

        public List<List<Integer>> perform(Integer n) {
            if (n == 0) {
                return Collections.singletonList(Collections.<Integer>emptyList());
            }
            final List<List<Integer>> subQueens = perform(n - 1);
            return Consumers.all(Multiplexing.flatten(Applications.map(subQueens, placeNextQueens)));
        }
    }

`PlaceQueens` is the heart of the backtracking algorithm. It delegates the adding of a new queen to the collaborator `placeNextQueens` and uses the method `Multiplexing.flatten` to flat the list of lists of boards in a list of boards.

## Conclusions

In this article I shown in a nutshell how to use *Dysfunctional* to develop real-world Java application in a purely functional style. Some may think that the proposed solutions, in addition to being less efficient, are also unnecessarily more complex than the respective imperative versions. Those who know how important it is write clear, maintainable, flexible and testable, will be greatly appreciated.  
In my opinion, the beauty in the implementation of N-queens is that you can easily test not only the whole algorithm, but also its individual components independently. It is also easy to change the behavior (within certain limits) by simply configuring the components appropriately.  
For example, to change the way in which the queen moves is sufficient to provide a different set of possible moves, or reimplement the `Mover`; to work on a rectangular board just give a different set of columns at the `PlaceNextQueens`; you can reimplementing the manner in which the piece is considered safe; etc.

Of course there are many alternative libraries, I can quote [functionaljava](http://functionaljava.org/), [guava](http://code.google.com/p/guava-libraries/), [lambdaj](http://code.google.com/p/lambdaj/), [fun4j](http://www.fun4j.org/) e [totallylazy](http://code.google.com/p/totallylazy/). 

The choice is yours!
