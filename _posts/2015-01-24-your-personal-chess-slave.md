title: Live chess with a real chessboard
date: 2015-01-24 20:03
tags: chess, java
ignore: true

Playing chess in front at a screen is so bad, all poetry is lost!
However play against a goog chess program or online is useful, so this is a simple idea: make a software to play chess with a computer with your real chessboard.

The software in outline should:
* look out the screen for you, detect the opponent moves and pronunce them;
* listen your voice, interprete what you tell and execute the move simulating your drag&drop of the piece.

This seems to be a slave, a *chesslave*.

## Dev plan

The target is clear, so let's analyze the involved components.

### The board recognizer

This component should detect a chessboard in the user desktop, observe its changes and get the moves done.

It can be divided in three sub-problems:

1. Find the region of an arbitrary chessboard displayed in the current screen.
2. Given the image of the board, analyze it extracting the images of pieces and white/black squares.

### The move reader

After have detected each opponent move the program should "read" the move for you pronouncing it on the speakers.

### The move listener

When is you turn the software should listen your voice, interprete the sounds and recognise the move.

### The move executor

After have received your move the program should execute it on the screen using simulating the use of the mouse.

### The GUI

The user interface should show the state of the program, that is:

* the graphics of the board and the pieces recognized;
* the current status of the board;
* the turn.

The user could perform the following operations:

* start a new game;
* pause the game;
* stop the game (the status is forgotten);
* force a new recognition (if the current state doesn't match with the effective state):
* force a turn switch.

### The controller

The controller orchestrates the interaction of all components, alternating the input from the user with the input from the screen.
