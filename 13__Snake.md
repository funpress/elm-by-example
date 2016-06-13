---
title: Snake
---

The *[Snake.elm](Snake.elm)* program is a game, in which the player
uses the keyboard arrows to choose the direction that the snake goes
to. The snake should eat food represented as green rectangles. When
the snake eats (covers it with its head), it also grows. The game
ends, when the snake collides with itself or a wall. Before
continuing, try the game [here](Snake.html), to have an idea of how it
works.

The code is divided into several modules:

 * `SnakeModel`
 * `SnakeView`
 * `SnakeSignal`
 * `SnakeState`
 * `Snake`

We start our analysis with the `SnakeModel` module defined in the
*[SnakeModel.elm](SnakeModel.elm)* file. The module starts with the
usual module declaration and imports.

% SnakeModel.elm
      module SnakeModel where


      import List ((::), head, isEmpty, map, reverse, tail)
      import Set

Then it defines the following type aliases:
% SnakeModel.elm

      type alias Position =
          { x: Int
          , y: Int
          }


      type alias Delta =
          { dx: Int
          , dy: Int
          }


      type alias Snake =
          { front: List Position
          , back: List Position
          }


      type alias SnakeState =
          { snake: Snake
          , delta: Delta
          , food: Maybe Position
          , ticks: Int
          , gameOver: Bool
          }

`SnakeState` represents the state of the game. The `snake` member, of
type `Snake`, contains the snake positions (the `Position` type) on
the board, stored in two lists (it will be explained below why it is
convenient to store it that way). The `delta` member, of type `Delta`,
stores the current direction of the snake. At any given point in time,
one of its members: `dx` or `dy` is set to `1` or `-1`, and the other
is set to `0`. The `food` represents the position of the food wrapped
in `Maybe`. The `ticks` member is a counter of `Tick` events (more on
it below). The `gameOver` member is a boolean value indicating whether
the game has been finished or not.

The `initialState` function creates the initial game state:
% SnakeModel.elm

      initialSnake =
          { front = [{x = 0, y = 0}, {x = 0, y = -1}, {x = 0, y = -2}, {x = 0, y = -3}]
          , back = []
          }


      initialDelta = { dx = 0, dy = 1 }


      initialFood = Nothing


      initialState =
          { snake = initialSnake
          , delta = initialDelta
          , food = initialFood
          , ticks = 0
          , gameOver = False
          }

The game state is changed in reaction to events represented by the
following data type:
% SnakeModel.elm

      type Event = Tick Position | Direction Delta | NewGame | Ignore

The `Tick` event is periodically generated based on a time signal and
contains a potential, new, randomly-generated position of the
food. The `Direction` event represents the new snake direction
generated based on a keyboard signal. The `NewGame` event is used for
starting the game from the beginning. Finally, there is the `Ignore`
event, that will be, well, ignored.

The game logic depends on certain constants:
% SnakeModel.elm

      boardSize = 15


      boxSize = boardSize + boardSize + 1


      velocity = 5

The `boxSize` is the size of the game board, calculated based on the
`boardSize` value. The size is expressed in logical units. One unit is
equivalent to a square drawn on the screen. The size of the square is
specified elsewhere. The `velocity` indicates how many `Tick` signals
are needed for one snake move.

The `nextPosition` function calculates the next position on the board
that the snake’s head will move into.
% SnakeModel.elm

      nextPosition : Snake -> Delta -> Position
      nextPosition snake {dx,dy} =
          let headPosition = head snake.front
          in  { x = headPosition.x + dx, y = headPosition.y + dy }

The `moveSnakeForward` function calculates the snake positions after
the move.
% SnakeModel.elm

      moveSnakeForward : Snake -> Delta -> Maybe Position -> Snake
      moveSnakeForward snake delta food =
          let next = nextPosition snake delta
              tailFunction =
                  case food of
                      Nothing -> tail
                      Just f -> if next == f then identity else tail
          in
              if isEmpty snake.back
              then { front = [next]
                   , back = (tailFunction << reverse) snake.front }
              else { front = next :: snake.front
                   , back = tailFunction snake.back }

The snake positions are stored in two lists. That way of
representation is chosen, because to move the snake, we need to
operate on its both ends: we need to add a position to the front, and
we need to remove one from its back (unless the snake has just eaten
food, in which case we leave the tail as it was). Thus `Snake.front`
represents the snake positions from its head backward, while
`Snake.back` represents its positions from the back forward (thus the
positions are stored in the opposite direction). Since the new
positions are added to the `front` and removed from the `back`, from
time to time the `back` member may be exausted and become empty. In
such a situation, the `front` list is reversed and assigned to `back`
(possibly with one element removed). The `tailFunction` function
returns the tail of the list given to it as argument, or returns the
list unchanged, based on whether the next position to be occuppied by
the snake is equal to the food position.

The `isInSnake` function verifies whether the snake contains a given
position. The function builds sets from both lists containing the
snake positions (using the `Set.fromList` function) and uses the
`Set.member` function to verify the membership.
% SnakeModel.elm

      isInSnake : Snake -> Position -> Bool
      isInSnake snake position =
          let frontSet = Set.fromList <| map toString snake.front
              backSet = Set.fromList <| map toString snake.back
          in
              Set.member (toString position) frontSet || Set.member (toString position) backSet

The `collision` function detects the collision state, that is a state
in which the next position of the snake belongs to the snake or is outside
the board.
% SnakeModel.elm

      collision : SnakeState -> Bool
      collision state =
          let next = nextPosition state.snake state.delta
          in
              if abs next.x > boardSize || abs next.y > boardSize || isInSnake state.snake next
              then True
              else False

The `SnakeView` module, defined in the
*[SnakeView.elm](SnakeView.elm)* file, contains functions responsible
for drawing the game. It begins with the module declaration and a
block of imports.

% SnakeView.elm
      module SnakeView where


      import Color (Color, black, blue, green, red, white)
      import Graphics.Collage (Form, collage, filled, move, rect)
      import Graphics.Element (Element, container, empty, midBottom, middle, layers)
      import List (map)
      import Maybe
      import SnakeModel (..)
      import Text

The snake and the food are drawn using filled squares. The actual
size of the squares and the size of the board boundaries are
calculated by the following functions:
% SnakeView.elm

      unit = 15


      innerSize = unit * boxSize


      outerSize = unit * (boxSize+1)

The `box` function draws the board boundaries by drawing two
rectangles: a bigger black one, and a smaller white one on top.
% SnakeView.elm

      box =
          collage outerSize outerSize [
              filled black <| rect outerSize outerSize,
              filled white <| rect innerSize innerSize ]

The `drawPosition` function draws a single square on a given position.
% SnakeView.elm

      drawPosition : Color -> Position -> Form
      drawPosition color position =
          filled color (rect unit unit) |>
          move (toFloat (unit*position.x), toFloat (unit*position.y))

The `drawPositions` function draws squares representing positions from a list.
% SnakeView.elm

      drawPositions : Color -> List Position -> Element
      drawPositions color positions =
          collage outerSize outerSize (map (drawPosition color) positions)

The `drawFood` function draws a green square representing food.
% SnakeView.elm

      drawFood : Position -> Element
      drawFood position = drawPositions green [position]

The `gameOver` function draws the text informing the user that the
game is over.
% SnakeView.elm

      gameOver : Element
      gameOver =
          Text.fromString "Game Over"
              |> Text.color red
              |> Text.bold
              |> Text.height 60
              |> Text.centered
              |> container outerSize outerSize middle

The `instructions` function shows the game instructions below the
board.
% SnakeView.elm

      instructions : Element
      instructions =
          Text.plainText "Press the arrows to change the snake move direction.\nPress N to start a new game."
              |> container outerSize (outerSize+3*unit) midBottom

The `view` function combines the above functions into one that draws
the whole game based on the state given in the argument.
% SnakeView.elm

      view state =
          layers [ box
                 , instructions
                 , drawPositions blue state.snake.front
                 , drawPositions blue state.snake.back
                 , Maybe.withDefault empty <| Maybe.map drawFood state.food
                 , if state.gameOver then gameOver else empty
                 ]

The `empty` function returns an element that is, well, empty. It is
not showed on the screen. That element is used if there is no food to
be drawn.

The `main` function is for testing purposes. The module can be
compiled and the resulting page opened, showing the game initial
state.
% SnakeView.elm

      main = view initialState

You can verify what it shows [here](SnakeView.html).

The `SnakeSignals` module creates several of the game signals. The
following figure presents how the individual signals are combined
together to produce the main game signal. The `SnakeSignals` module
defined the functions showed on the figure, except for the
`stateSignal` and `main` functions, which are defined in different
modules.

![](i/snake1.png)

<!--
"""

sigBox a b c w x line = signalFunctionBox 14 18 50 a b c w x (line*100-300)
sigVertU line x = sigVerticalLine 25 x (line*100-238)
sigVertD line x = sigVerticalLine 25 x (line*100-238-25)
sigVert line x = sigVerticalLine 50 x (line*100-250)
sigHoriz w line x = sigHorizontalLine w x (line*100-250)
sigArr line x = sigDownwardArrow x (line*100-265)
sigVertArr line x = group [sigVert line x, sigArr line x ]

picture1 = collage 600 510
  [ sigBox "Signal Int" "timeSignal" "fps" 100 0 5

  , sigVertArr 4 0

  , sigBox "Signal Event" "directionSignal" "Keyboard.arrows" 170 -200 4
  , sigBox "Signal Event" "tickSignal" "" 170 0 4
  , sigBox "Signal Event" "newGameSignal" "Keyboard.isDown, keepIf" 170 200 4

  , sigVertU 3 -200
  , sigVertArr 3 0
  , sigVertU 3 200
  , sigHoriz 400 3 0

  , sigBox "Signal Event" "eventSignal" "merges" 120 0 3

  , sigVertArr 2 0

  , sigBox "Signal SnakeState" "stateSignal" "foldp" 140 0 2

  , sigVertArr 1 0

  , sigBox "Signal Element" "main" "" 140 0 1
  ]

{-

% SnakeSignals.elm
      module SnakeSignals where


      import Char
      import Keyboard
      import Random
      import Signal ((<~), Signal, keepIf, mergeMany)
      import SnakeModel (..)
      import SnakeView (..)
      import Time (Time, fps)

-}
-->


The `timeSignal` function uses the `fps` function to produce a signal
of `Time` values ticking with the approximate rate of 50 events per
second.

% SnakeSignals.elm

      timeSignal : Signal Time
      timeSignal = fps 50

% SnakeSignals.elm

The `makeTick` function creates a `Tick` event given a `Time`
value. Each such event carries a `Position` value representing the
potential new food position.

% SnakeSignals.elm

      makeTick : Time -> Event
      makeTick time =
          let seed1 = Random.initialSeed (round time)
              (x,seed2) = Random.generate (Random.int -boardSize boardSize) seed1
              (y,_) = Random.generate (Random.int -boardSize boardSize) seed2
          in
              Tick { x = x, y = y }

The `tickSignal` function maps `makeTick` over the time signal,
producing a signal of `Tick` events.

% SnakeSignals.elm

      tickSignal : Signal Event
      tickSignal = makeTick <~ timeSignal

The `directionSignal` function uses the `Keyboard.arrows` function and
produces a signal of the directions the snake should move to.
% SnakeSignals.elm

      directionSignal : Signal Event
      directionSignal =
        let arrowsToDelta {x,y} =
                if  | x == 0 && y == 0 -> Ignore
                    | x /= 0 -> Direction { dx = x, dy = 0 }
                    | otherwise  -> Direction { dx = 0, dy = y }          
        in
            arrowsToDelta <~ Keyboard.arrows

The `newGameSignal` function produces a signal of `NewGame`
events. The events are generated when the player presses the “N” key
on the keyboard. The `Keyboard.isDown` function is used for detecting
the key events, while the `keepIf` function is used to filter-out the
events related to releasing the button. The `always` function always
returns its argument (`NewGame`) regardles of its input.
% SnakeSignals.elm

      newGameSignal : Signal Event
      newGameSignal =
          always NewGame <~ (keepIf identity False <| Keyboard.isDown (Char.toCode 'N'))

The `eventSignal` function merges the signals produced by
`tickSignal`, `directionSignal` and `newGameSignal`. Notice that all
the input signals have the same signature.
% SnakeSignals.elm

      eventSignal : Signal Event
      eventSignal = mergeMany [tickSignal, directionSignal, newGameSignal]

The `stateSignal` function is defined in the `StateState` module. The
module obviously starts with the module declaration and imports.

% SnakeState.elm
      module SnakeState where


      import List (head)
      import Signal (Signal, foldp)
      import SnakeModel (..)
      import SnakeSignals (..)

The `stateSignal` function uses the `foldp` function and produces a
signal of `SnakeState`.
% SnakeState.elm

      stateSignal : Signal SnakeState
      stateSignal = foldp step initialState eventSignal

The `foldp` function takes three arguments. The first one is the
`step` function — it is a function that takes two arguments (the
current event and the current state) and produces the new state. The
second argument is the initial state, returned by the `initialState`
function. The third one is the signal of input events (returned by
`eventSignal`).

The `step` function is producing the next game state based on the
event received and the current state.
% SnakeState.elm

      step : Event -> SnakeState -> SnakeState
      step event state =
          case (event,state.gameOver) of
              (NewGame,_) -> initialState
              (_,True) -> state
              (Direction newDelta,_) ->
                  { state | delta <- if abs newDelta.dx /= abs state.delta.dx
                                     then newDelta
                                     else state.delta }
              (Tick newFood, _) ->
                  let state1 = if state.ticks % velocity == 0
                               then { state | gameOver <- collision state }
                               else state
                  in if state1.gameOver
                     then state1
                     else let state2 = { state1
                                        | snake <-
                                            if state1.ticks % velocity == 0
                                            then moveSnakeForward state1.snake state1.delta state1.food
                                            else state1.snake
                                        }
                              state3 = { state2
                                       | food <-
                                             case state2.food of
                                                 Just f ->
                                                     if state2.ticks % velocity == 0 &&
                                                        head state2.snake.front == f
                                                     then Nothing
                                                     else state2.food
                                                 Nothing ->
                                                     if isInSnake state2.snake newFood
                                                     then Nothing
                                                     else Just newFood
                                       }
                           in { state3 | ticks <- state3.ticks + 1 }
              (Ignore,_) -> state

The function verifies certain conditions and reacts to the first one
that is true. It first verifies whether the event received is `NewGame`,
in which case the function returns the initial state, regardless of
what is the current state.

If the `gameOver` member of the current state is true, then the state
is returned unchanged.

When a `Direction` event is received, the `delta` member is updated, but
only if the new direction does not cause the snake to turn back (what
would cause an immediate collision).

When the `Tick` event is received, several changes to the state are
performed. The changes are performed one by one, producing
intermediate states (`state1`, `state2`, `state3`). First, the
`collision` function verifies whether a collision can be detected, in
which case the game is over. The result is stored in the `gameOver`
member of `state1`. If the game is over, `state1` is returned and
no further state updates are needed. Otherwise, the snake moves
forward but only if the `ticks` modulo the velocity is equal to
zero.

The next step is to update the `food` member. If there is food on the
board, the snake has just moved, and its head’s position is equal to
the position of the food, it means the snake has just eaten the food,
in which case we set the `food` member to `Nothing`. If there is no
food on the board (which happens after it has been eaten or at the
beginning of the game), we verify whether the `newFood` can safely be
put on the board, that is whether its position is not equal to the
position of any segment of the snake. If it is safe, we update
`food` to have the new food postion (wrapped in `Just`).

Finally the `ticks` member is incremented and the new state returned.

The `Ignore` event is ignored, that is it does not cause any state
change.

The `Snake` module implements the `main` function.

% Snake.elm
      module Snake where


      import Graphics.Element (Element)
      import Signal ((<~), Signal)
      import SnakeState (..)
      import SnakeView (..)


      main : Signal Element
      main = view <~ stateSignal

The `main` function returns a `Signal Element` signal which is
interpreted by Elm’s runtime, rendering the application.

In order to transform the game state, we have used a monolitic `step`
function, that reacts to each possible combination of input event and
current state. The solution works, but it has the disadvantage that
the function which transforms the state may become big and difficult
to maintain for larger programs. The
next chapter presents an alternative
solution for implementing the same game.
