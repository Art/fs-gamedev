Now that we know how to print text, we will see in this second tutorial how to compute things worth printing.
```
#light

open System

let gravity = -9.81

let euler_explicit accel pos speed delta =
    pos + delta * speed, speed + delta * accel

let euler_implicit accel pos speed delta =
    let speed2 = speed + delta * accel
    pos + delta * speed2, speed2

let pos, speed = euler_explicit gravity 0.0 0.0 1.0 in
    printf "pos = %f speed = %f\n" pos speed

let pos, speed = euler_implicit gravity 0.0 0.0 1.0 in
    printf "pos = %f speed = %f\n" pos speed

Console.ReadKey(true)
```

Now come the explanations for each new line in this tutorial.

```
let gravity = -9.81
```
Declares a constant representing the acceleration due to gravity.

```
let euler_explicit accel pos speed delta =
    pos + delta * speed, speed + delta * accel
```
A first version of Euler integration. A C programmer would have written it:
```
pos += delta * speed;
speed += delta * accel;
```
The F# and the C version differ notably in the way they handle the newly computed data. The C code uses variables which are changed during the computation, while the F# version computes new values which are returned, leaving the original values unchanged. Neither F# nor C force you to do it that way, but this is the natural way to do it in these languages.
The functional way demonstrated in the F# code is more flexible, because the old values are not lost. Should we later realize we need them, we will be able to use them without refactoring the code. It also clarifies the role of each variable, since the content of each variable never changes.
The procedural way, shown in the C snipplet, is likely more efficient, at least on current processors. Side-effect-free computing easily takes advantage of concurrency, meaning that the F# way may well become more efficient in practice when (if?) we get processors with a large number of cores.

Back to the F# code:
```
let euler_implicit pos speed delta =
    let speed2 = speed + delta * accel
    pos + delta * speed2, speed2
```
A variant of Euler integration popular among game developers. The position is updated using the speed at the next time step. Notice the definition of "speed2", which is used as a shorthand for the speed at the next time step. The scope of "speed2" is limited to the expression following "in". Again, the C counterpart would be:
```
speed += delta * accel;
pos += delta * speed;
```

Continuing with the F# code:
```
let pos, speed = euler_explicit gravity 0.0 0.0 1.0 in
    printf "pos = %f speed = %f\n" pos speed

let pos, speed = euler_implicit gravity 0.0 0.0 1.0 in
    printf "pos = %f speed = %f\n" pos speed
```
Finally, these lines call each integration method with null initial positions and speeds. The value of the time step is quite large in this example (1 second), we will see in the next tutorial how to incrementally compute the position and speed using a smaller time step.

The output of the program should look like this:


pos = 0.000000 speed = -9.810000

pos = -9.810000 speed = -9.810000


Note that the position is not quite correct, but that was expected, given the large integration step.

We have just learned how to define and use constants and functions. Note that the syntax is the same in both cases, which is not shocking. Indeed, constants can be seen as a functions of zero parameters. In the next tutorial we will look at a few constructs to control the flow of the program.