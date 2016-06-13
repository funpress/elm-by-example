---
title: Circles
---

The next example, *[Circles.elm](Circles.elm)*, is a program which
maintains state. Initially, the program only shows an empty
square. However, after you click inside it, a colorful circle is
created and starts moving inside the square. After each subsequent
click, another circle is created. Before continuing, take a look at
the working program [here](Circles.html), to have an idea of how it
works.

The code is divided into three modules:

 * `CirclesModel`
 * `CirclesView`
 * `Circles`

We start our analysis with the `CirclesModule` module, defined in the
*[CirclesModel.elm](CirclesModel.elm)*, which starts with the usual
module declaration followed by imports and three type declarations:

% CirclesModel.elm
      module CirclesModel where


      import Color (Color, rgb)
      import Time (Time)
      import Random (generate, int, initialSeed)


      type alias Position = { x: Int, y: Int }


      type alias CircleSpec = {
               radius: Int,
               xv: Int,
               yv: Int,
               col: Color,
               creationTime: Time
           }


      type alias Circle = {
               position: Position,
               circleSpec: CircleSpec
           }

The module defines three data types, that we will use in our
program. Their definitions start with the `type alias` keywords
followed by the type name. The `type alias` statement creates, a type
alias, that is, the type on the right hand side of the equals sign
acquires a new name.

All of the data types are records. A record is a data structure
consisting of one or more values. Each value in a record has a name
and a type. For example, `Position` is an alias for a record type
consisting of two values, `x` and `y`, both of type `Int`.

We can create record values by providing field names followed by the
equal sign and the field values, separated by commas and enclosed in
curly braces. Here is an example:

      > myRecord = { x = round 1.1, y = round 4.9 }
      { x = 1, y = 5 } : {x : Int, y : Int}

We can reference record fields by appending a dot and the field name
to the record name. The following function converts our record to a
string:

      > showRecord rec = concat [show rec.x, " ", show rec.y]
      <function> : {c | y : b, x : a} -> String
      > showRecord myRecord
      "1 5" : String

We can also *pattern match* on record field. We do it by providing a
pattern consisting of field names separated by commas and enclosed in
curly braces. Here is another version of the `showRecord` function,
which uses pattern matching in the `let` expression:

      > showRecord rec = let {x,y} = rec in concat [show x, " ", show y]
      <function> : {c | x : a, y : b} -> String
      > showRecord myRecord
      "1 5" : String

Pattern matching can also be used directly in the function parameters,
as will be shown in a moment.

Going back to our data types, the `CircleSpec` records contain data
specifying a circle: its radius, its vertical and horizontal velocity
(`xv` and `yv`), its color and the time when it was created. The
`Circle` data type contains the circle specification (`CircleSpec`)
and its position.

The `makeCircleSpec` function creates a new `CircleSpec` record given
a `Time` value.

% CirclesModel.elm

      makeCircleSpec : Time -> CircleSpec
      makeCircleSpec time =
          let seed1 = initialSeed (round time)
              (radius,seed2) = generate (int 10 30) seed1
              (xv,seed3) = generate (int 10 50) seed2
              (yv,seed4) = generate (int 10 50) seed3
              (r,seed5) = generate (int 10 220) seed4
              (g,seed6) = generate (int 10 220) seed5
              (b,_) = generate (int 10 220) seed6
          in
              { radius = radius
              , xv = xv
              , yv = yv
              , col = rgb r g b
              , creationTime = time
              }

The `makeCircleSpec` function is using several functions from the
`Random` module, which contains functions related to generating random
values. The function generates random values using the `generate`
function, which has the following type:

      generate : Generator a -> Seed -> (a, Seed)

Its first argument is generator, which produces random values of a
certain type `a`. The `makeCircleSpec` function uses generators
created by the `int` function, which takes two `Int` arguments
defining a range of values and returns a generator of `Int` values
from that range.

      int : Int -> Int -> Generator Int

Besides a generator, the `generator` function needs a `Seed` value to
produce a result. A `Seed` value can be created by the `initialSeed`
function, which takes an `Int` value.

      initialSeed : Int -> Seed

The `generate` function returns a random value produced by the
generator and a new seed. If we try to call the `generate` function
with the same generator and seed, we will always get the same return
value (the REPL output is a little verbose, showing the internals of
the seed):

      > generate (int 10 18) (initialSeed 12345)
      (12,{ next = <function>, range = <function>, split = <function>, state = State 494012844 40692 })
          : ( Int
            , { next : Random.State -> ( Int, Random.State )
              , range : Random.State -> ( Int, Int )
              , split : Random.State -> ( Random.State, Random.State )
              , state : Random.State
              }
            )
      > generate (int 10 18) (initialSeed 12345)
      (12,{ next = <function>, range = <function>, split = <function>, state = State 494012844 40692 })
          : ( Int
            , { next : Random.State -> ( Int, Random.State )
              , range : Random.State -> ( Int, Int )
              , split : Random.State -> ( Random.State, Random.State )
              , state : Random.State
              }
            )

That is not very useful if we really need randomness. We can, however,
use the seed returned by `generate` as the input seed in the next call
to `generate`:

      > result = generate (int 10 18) (initialSeed 12345)
      (12,{ next = <function>, range = <function>, split = <function>, state = State 494012844 40692 })
          : ( Int
            , { next : Random.State -> ( Int, Random.State )
              , range : Random.State -> ( Int, Int )
              , split : Random.State -> ( Random.State, Random.State )
              , state : Random.State
              }
            )
      > generate (int 10 18) (snd result)
      (18,{ next = <function>, range = <function>, split = <function>, state = State 1991225964 1655838864 })
          : ( Int
            , { next : Random.State -> ( Int, Random.State )
              , range : Random.State -> ( Int, Int )
              , split : Random.State -> ( Random.State, Random.State )
              , state : Random.State
              }
            )

The `makeCircleSpec` function uses a time value (recall that `Time` is
an alias of `Float`) to get the initial seed. It then calls `generate`
several times in order to calculate the circle specification elements,
passing previously calculated seeds as input to the subsequent
`generate` calls. In order to calculate the circle color, it uses the
`rgb` function, which takes three `Int` values representing the
primary colors: red, green and blue.

      > makeCircleSpec 12345.0
      { col = RGBA 34 33 99 1, creationTime = 12345, radius = 18, xv = 10, yv = 34 }
          : { col : Color.Color
            , creationTime : Float
            , radius : Int
            , xv : Int
            , yv : Int
            }

We will now turn our attention to the `CirclesView` module. It is
defined in the *[CirclesView.elm](CirclesView.elm)* file and it starts
as usual with the module declaration and a list of imports.

% CirclesView.elm
      module CirclesView where


      import CirclesModel (Circle, CircleSpec, Position)
      import Color (black, red, green)
      import Graphics.Collage (circle, collage, filled, move, outlined, rect, solid)
      import Graphics.Element (layers)
      import List (map)

The `boundingBox` function draws a square of a given width and
height. The `outlined` function draws the square border using the line
specification provided as its first argument. In our case we want a
solid black line. The `solid` function takes a color and returns a
`LineStyle` value representing the line style to be used.
% CirclesView.elm

      boundingBox w h =
          collage
              w
              h
              [
                  outlined (solid black) <| rect (toFloat w) (toFloat h),
                  outlined (solid black) <| rect (toFloat (w-2)) (toFloat (h-2))
              ]

The `drawCircle` function draws a circle. It takes as arguments the
width and height of the bounding box, and the information about the
circle, of type `Circle`. The third argument is not specified in the
usual way — as an indentifier. Instead, a *pattern* is used, which
enumerates the member names of the `Circle` data type. The members are
separated by a comma and enclosed in curly braces. This is *pattern
matching* in action again. The function body can directly reference
the members of the `Circle` record passed as the function
argument. Although the `drawCircle` function enumerates all members of
the `CircleSpec` data type in the pattern, in general we do not have
to enumerate all of them if not all of them are used in the function
body.
% CirclesView.elm

      drawCircle w h {position, circleSpec} =
          filled circleSpec.col (circle (toFloat circleSpec.radius))
              |> move (toFloat position.x - (toFloat w)/2, (toFloat h)/2 - toFloat position.y)

The circle position is adjusted, because we want to specify the
position using the same coordinate system that is used by Elm to
represent the mouse clicks positions. That coordinate system has its
origin (the point with the (0,0) coordinates) in the left-upper
corner. The *x* coordinates increase to the right, and the *y*
coordinates increase downwards. However, the coordinate system used
for drawing forms in a collage is different: the origin is in the
middle and the coordinates increase when going to the right and
upwards.

The `drawCircles` function draws circles from a given list by mapping
the `drawCircle` function over the list. The `view` function draws the
complete view by drawing the bounding box first and the circles on top
of it. To draw two (or more) elements on top of each other, the
`layers` function from the `Graphics.Element` module is used.
% CirclesView.elm

      drawCircles w h circles = collage w h (map (drawCircle w h) circles)


      view w h circles = layers [ boundingBox w h, drawCircles w h circles ]

The `main` function let us test the `view` function with sample
data. Notice, that since the drawing functions do not use all of the
fields from the records defined in `CirclesModel`, we do not need to
provide all of them when constructing the sample data. With our test
data, we should see a red circle in the left-upper corner, and a green
one in the center of the bounding box. You can verify whether that is
what really happens [here](CirclesView.html).
% CirclesView.elm

      main =
          view
              400
              400
              [
                  { circleSpec = { col = red, radius = 26 } , position = { x = 0, y = 0 } },
                  { circleSpec = { col = green, radius = 43 } , position = { x = 200, y = 200 } }
              ]

The `Circles` module defines the signals used in our program. Here is
its beginning:

% Circles.elm
      module Circles where


      import CirclesModel (..)
      import CirclesView (..)
      import List ((::), map)
      import Mouse
      import Signal (Signal, (<~), (~), foldp, keepIf, sampleOn)
      import Time (Time, fps, timestamp)

The module creates several signals and combines them together. The
following figure presents the relations between the individual
signals.

![](circles1.png)


<!--
"""

sigBox a b c w x line = signalFunctionBox 14 18 50 a b c w x (line*100-300-50)
sigVertU line x = sigVerticalLine 25 x (line*100-238-50)
sigVertD line x = sigVerticalLine 25 x (line*100-238-25-50)
sigVert line x = sigVerticalLine 50 x (line*100-250-50)
sigHoriz w line x = sigHorizontalLine w x (line*100-250-50)
sigArr line x = sigDownwardArrow x (line*100-265-50)
sigVertArr line x = group [sigVert line x, sigArr line x ]

picture1 = collage 900 810
  [ sigBox "Signal (Int,Int)" "clickPositionsSignal" "sampleOn Mouse.clicks Mouse.position" 240 -150 7

  , sigVertArr 6 -150

  , sigBox "Signal (Int,Int)" "inBoxClickPositionsSignal" "Signal.keepIf" 200 -150 6
  , sigBox "Signal Time" "clockSignal" "Time.timestamp, Time.fps" 200 150 6

  , sigVertU 5 -150
  , sigVertU 5 150
  , sigHoriz 100 5 -100
  , sigHoriz 100 5 100
  , sigVertD 5 -50, sigArr 5 -50
  , sigVertD 5 50, sigArr 5 50

  , sigBox "Signal Time" "creationTimeSignal" "Signal.sampleOn" 170 0 5

  , sigVertArr 4 0
  , sigVerticalLine 200 -150 (4*100-250-50)

  , sigBox "Signal CircleSpec" "newCircleSpecSignal" "" 200 0 4

  , sigVertArr 3 0
  , sigHoriz 100 3 -100
  , sigVertD 3 -50, sigArr 3 -50
  , sigVerticalLine 400 150 (3*100-250-50)

  , sigBox "Signal Circle" "newCircleSignal" "" 200 0 3

  , sigVertArr 2 0

  , sigBox "Signal [Circle]" "allCirclesSpecSignal" "foldp" 200 0 2

  , sigVertArr 1 0
  , sigHoriz 100 1 100
  , sigVertD 1 50, sigArr 1 50

  , sigBox "Signal [Circle]" "circlesSignal" "" 200 0 1

  , sigVertArr 0 0

  , sigBox "Signal Element" "main" "" 200 0 0

  ]
-->


The first signal from the `Circles` module is created by the
`clockSignal` function. It periodically outputs a timestamp. The rate
of events is established by the `fps` function (fps means frames per
second) from the `Time` module.
% Circles.elm

      clockSignal : Signal Time
      clockSignal = fst <~ timestamp (fps 50)

The signal created by the `clickPositionsSignal` function outputs the mouse
pointer position on every click.
% Circles.elm

      clickPositionsSignal : Signal (Int, Int)
      clickPositionsSignal = sampleOn Mouse.clicks Mouse.position

The `inBoxClickPositionsSignal` function takes the width and height as
arguments and creates a signal, that filters the events from the
`clickPositionsSignal` to only output those that represent positions
inside the bounding box of the given width and height. The `keepIf`
function from the standard `Signal` module is used for filtering the
events.
% Circles.elm

      inBoxClickPositionsSignal : Int -> Int -> Signal (Int, Int)
      inBoxClickPositionsSignal w h =
          let positionInBox pos = fst pos <= w && snd pos <= h
          in
              keepIf positionInBox (0, 0) clickPositionsSignal

The `creationTimeSignal` function produces a signal representing the
creation times of the circles. It is created by sampling (using the
`sampleOn` function) the signal produced by the `clockSignal` function
on the events carried on by the signal from the
`inBoxClickPositionsSignal` function.
% Circles.elm

      creationTimeSignal : Int -> Int -> Signal Time
      creationTimeSignal w h =
          sampleOn (inBoxClickPositionsSignal w h) clockSignal

The `newCircleSpecSignal` function combines several signals to produce
a signal representing the specifications of the new circles.
% Circles.elm

      newCircleSpecSignal : Int -> Int -> Signal CircleSpec
      newCircleSpecSignal w h =
          makeCircleSpec <~ creationTimeSignal w h

A full circle representation consists of its specification and its
position. The `newCircleSignal` function produces a signal
representing circles by combining the `newCircleSpecSignal`
(representing the specifications) and the `inBoxClickPositionsSignal`
(representing the positions — the initial position of a circle is the
position of where the user clicked inside the bounding box) signals.
% Circles.elm

      newCircleSignal : Int -> Int -> Signal Circle
      newCircleSignal w h =
          let makeCircle (x,y) spec = { position = { x = x, y = y }, circleSpec = spec }
          in
              makeCircle
                  <~ inBoxClickPositionsSignal w h
                  ~ newCircleSpecSignal w h

We have arrived to a moment when we want to maintain state in our
program. We want the program to remeber the circles created each time
the user clicked inside the bounding box. After each such click, we
want to add the new circle to a list of previous circles (our list is
empty at the beginning). New circles are represented by a signal of
type `Signal Circle`. We now want to have a signal of type `Signal
(List Circle)`, which contains a list of all circles created so far.

We can use the `foldp` function from the `Signal` module to maintain
state in Elm programs. The signature of that function looks as follows:

      foldp : (a -> b -> b) -> b -> Signal a -> Signal b

It takes three arguments. It transforms a signal of values of some
type `a` (given as the third argument) into a signal of values of some
type `b`. The initial value of the output signal is provided as the
second argument. The first argument is a function that takes two
arguments — one of type `a` and one of type `b` — and returns a value
of type `b`. Let’s call that function a *transformation* function. For
each event of the input signal (the signal of values of type `a`) the
transformation function is called with the value of the new signal,
and the current value of the output signal as arguments. The result
of calling the transformation function becomes the new value of the
output signal. It will also be the value given as the second argument
in the subsequent call to the transformation function — whenever a new
event from the input signal is going to be processed. Thus the `b`
values represent the state that is maintained by the `foldp` function
and also the results emitted by the signal produced by that function.

The `allCirclesSpecSignal` function uses `foldp` to maintain a list of
circles — which is our state. The initial value of the state is the
empty list, and this is the value of the second argument given to the
`foldp` function. The input signal provided as the third argument is
produced by the `newCircleSignal`. The first argument is simply a
function that appends the new circle to the current list of
circles.

We do not need to write that function ourselves, since it already
exists in Elm. It is called `::` and it takes two arguments — a value
and a list — and returns a new list consisting of the new value
prepended to the list.

      > 1 :: []
      [1] : List number
      > 2 :: [1]
      [2,1] : List number
      > 3 :: [2,1]
      [3,2,1] : List number

We can thus use the `::` operator as the first argument to our `foldp`
call:
% Circles.elm

      allCirclesSpecSignal : Int -> Int -> Signal (List Circle)
      allCirclesSpecSignal w h =
          foldp (::) [] (newCircleSignal w h)

Since `::` is an operator — its name is not alphanumeric, but it
contains other symbols — we have to enclose its name in
parenthesis. Otherwise the Elm compiler would complain.

Having a list of circles is not enough for our purposes. We want the
circles to move on the screen. We thus have to update their
coordinates as a function of time. We will use the `computeCoordinate`
function to calculate the circle coordinates — each of the *x* and *y*
coordinates separately. The `computeCoordinate` function takes four arguments:

 * the coordinate of the starting position
 * the size of the box inside which the circle is to move
 * the circle velocity
 * the time that passed since the circle was created

It computes the distance the circle has travelled since its
creation. It then calculates the coordinate based on that distance. If
the distance divided by the box size is even, the circle should move
to the right or downwards and the coordinate is taken as the distance
modulo the box size. If it is odd, the circle should move to the left
or upwards, and the coordinate is the box size minus the distance
modulo the box size.
% Circles.elm

      computeCoordinate : Int -> Int -> Float -> Float -> Int
      computeCoordinate startingPointCoordinate boxSize velocity time =
          let distance = startingPointCoordinate + round(velocity * time / 1000)
              distanceMod = distance % boxSize
              distanceDiv = distance // boxSize
        in
            if (distanceDiv % 2 == 0)
                then distanceMod
                else boxSize - distanceMod

The `positionedCircle` function transforms a `Circle` value
representing its initial state, into a new `Circle` value with its
position updated.
% Circles.elm

      positionedCircle : Int -> Int -> Float -> Circle -> Circle
      positionedCircle w h time circle =
          let {position, circleSpec} = circle
              {radius, xv, yv, creationTime} = circleSpec
              relativeTime = time - creationTime
              boxSizeX = w - radius*2
              boxSizeY = h - radius*2
              x = radius + computeCoordinate (position.x-radius) boxSizeX (toFloat xv) relativeTime
              y = radius + computeCoordinate (position.y-radius) boxSizeY (toFloat yv) relativeTime
          in
              { position = { x=x, y=y }, circleSpec = circleSpec }

We use two different techniques to access individual members of the
`Circle` record. The first line after the `let` keyword use pattern
matching to extract individual members of the `circle`. The second
line also uses pattern matching to extract the members of the
`circleSpec` value. We do not perform the similar extraction for the
`position` members. Instead, the `x` and `y` coordinates are accessed
using the dot notattion. We calculate the circle position using the
`computeCoordinate` function separately for *x* and *y*
coordinates. The size of the box inside which the circle is to move is
calculated as the widht or height of the bounding box minus twice the
circle radius. We substract the radius, because the position
represents the center of the circle, but we want the circle to change
its direction as soon as its boundary touches the border of the
bounding box. Thus the real bounding box for the center of a given
circle is smaller than the visible bounding box by twice the circle
radius — we substract the radius once for each side of the box. (There
is a small problem with this design, which the implementation silently
ignores. What happens when the user clicks near the border of the
bounding box? The center of the circle, after taking the modulo
operation, may not be in the same place where the user clicked.)

The `positionedCircles` function maps the `positionedCircle` function
partially applied with the appropriate parameters, through a list of
`Circle` values.
% Circles.elm

      positionedCircles : Int -> Int -> Float -> List Circle -> List Circle
      positionedCircles w h time circles =
          map (positionedCircle w h time) circles

Notice how we have used the `positionedCircle` function. That function
takes a `Circle` as its last argument. The `map` function requires a
one-argument function and in our case the argument should have the
type of `Circle`. By partially applying all but last arguments of the
`positionedCircle` function, we obtain a one-argument function
suitable to be passed as the first argument to `map`.

We can now use `positionedCircles` to create a signal of circles whose
positions change while time passes. This is the job of the
`circlesSignal` function, which combines the signals produced by
`clockSignal` and `allCirclesSpecSignal`.
% Circles.elm

      circlesSignal : Int -> Int -> Signal (List Circle)
      circlesSignal w h = positionedCircles w h <~ clockSignal
                                                 ~ allCirclesSpecSignal w h

The `main` function uses the values of the `circlesSignal` as input to
the `view` function from the `CirclesView` module, producing the main
program signal, that is rendered by Elm’s runtime.
% Circles.elm

      main =
          let main' w h = view w h <~ circlesSignal w h
          in
              main' 400 400


So far we have used signals from the `Mouse` module to get mouse
related input. The next chapter presents an
alternative way of processing mouse events.
