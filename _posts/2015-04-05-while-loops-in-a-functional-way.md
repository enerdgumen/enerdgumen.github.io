title: While loops in a functional way
date: 2015-03-24 20:03
tags: java, functional

The following code shows a technique to implement while loops taking advantage of java streams and functional programming.

## Do/while loops

This is a classic do/while loop where the body is essentially a supplier of states:

    :::java
    T state;
    do {
    	state = body();
    } while (predicate(state));

It is equivalent to the following code:

    :::java
    final T endState = Stream.generate(body)
    	.filter(predicate.negate())
    	.findFirst()
    	.get();

The stream will terminate when the current generated value hold the predicate in the filter clause, therefore such clause - used in this way - behaves as an "until" condition and must be negated.

The stream is consumed invoking `findFirst`, at that point can happend two things:

1. the stream terminates: `findFirst` returns an `Optional` containing the last state checked by the predicate, so the last state is always available;
2. the stream doesn't terminate: unfortunately there's an infinite loop!

## Stateful while loops

Sometimes the current state of a while loop depends from the previous state:

    :::java
    T state = initial;
    while (predicate(state)) {
    	state = body(state);
    }

The equivalent functional code is similar to the previous but it uses `iterate` to generate a chain of dependent states:

    :::java
    final T endState = Stream.iterate(initial, body)
    	.filter(predicate.negate())
    	.findFirst()
    	.get()

## A little example

As a demonstration, suppose you have started a task to a remote service and you want monitor the operation getting the task status in polling. You want read the status with 10 tentatives before to declare the operation failed too.

An imperative solution:

	:::java
	Outcome run() {
		State state;
		do {
			state = getTaskState();
		} while (!isCompleted(state));
		return state.success ? SUCCESS : FAILURE;
	}

	State getTaskState() {
		int attempt = 0;
		do {
			Thread.sleep(1000);
			try {
				return readEffectiveTaskStatus(...);
			} catch (Exception ex) {
				// log
			}
			attempt += 1;
		} while (attempt < 10);
		return State.failure();
	}

A functional alternative:

	:::java
	Outcome run() {
		final State state = Stream.generate(this::getTaskState)
			.filter(this::isCompleted)
			.findFirst()
			.get();
		return state.success ? SUCCESS : FAILURE;
	}

	State getTaskState() {
		return Stream.generate(this::readEffectiveTaskStatus)
			.limit(10)
			.filter(Optional::isPresent)
			.map(Optional::get)
			.findFirst()
			.orElse(State.failure());
	}

	Optional<State> readEffectiveTaskStatus() {
		try {
			Thread.sleep(1000);
			return Optional.of(...);
		} catch (Exception ex) {
			return Optional.empty();
		}
	}