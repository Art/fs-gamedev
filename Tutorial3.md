The code for this tutorial is similar to the previous'.

```
#light

open System

let gravity = -9.81

let euler_implicit accel pos speed delta =
    let speed2 = speed + delta * accel
    pos + delta * speed2, speed2

let rec simulate t0 t_fin delta pos0 speed0 accel =
    if t0 >= t_fin then pos0, speed0
    else
        let t1 = t0 + delta
        let pos1, speed1 = euler_implicit accel pos0 speed0 delta
        simulate t1 t_fin delta pos1 speed1 accel

let pos = 0.0
let speed = 0.0
let t0 = 0.0
let t_fin = 10.0
let delta = 1e-3

let pos_fin, speed_fin = simulate t0 t_fin delta pos speed gravity in
    printf "pos = %3.3f\nspeed = %3.3f\n" pos_fin speed_fin;

Console.ReadKey(true)
```

Here come the explanations, focused on the new function "simulate":

```
let rec simulate t0 t_fin delta pos0 speed0 accel =
```
This defines a recursive function named "simulate". Recursion is used as a substitute for looping. I will not explain recursion here. It is simple in essence, but can be hard to grasp at first for someone used to think in C. Note that "rec" in "let rec simulate" is used to tell the compiler that "simulate" calls itself. Follows a list of parameters:
  * t0 is the initial time when the simulation starts,
  * t\_fin is the final time when the simulation finishes,
  * pos0 is the initial position,
  * speed0 is the initial speed,
  * accel is the constant acceleration.

```
if t0 >= t_fin then pos0, speed0
```
If the initial time is after the final time, the function directly returns the initial position and speed.

```
else
    let t1 = t0 + delta in
    let pos1, speed1 = euler_implicit accel pos0 speed0 delta in
```
Otherwise, the integration function is called to compute the position and the speed at t0 + delta...

```
    simulate t1 t_fin delta pos1 speed1 accel
```
... and "simulate" is called recursively, with the initial time, position and speed set to the values that were just computed.

Here is the output of the program:


pos = -490.647
speed = -98.110



We have seen how to simulate loops using recursion and if-then-else conditions. This is an alternative to using a while loop. Notice that F# has no problem handling the deep recursion. Thanks to the optimizations supported by the F# compiler, recursion can be used heavily by programmers without having to worry about running out of stack space. There is a catch, though: the recursive call must be the last instruction in the function.

In the next tutorial, we will see how to simulate spring and dampers. This will take advantage of the fact that functions in F# are "first-class citizen" and can be used as any other kind of parameter.