As usual, let me show you the code first.

```
#light

open System

let gravity =
    -9.81

let spring pos =
    let k = 10.0 in
        -k * pos

let drag k =
    fun speed -> -k * speed

let euler_implicit accel pos speed delta =
    let speed2 = speed + delta * accel in
        pos + delta * speed2, speed2

let rec simulate t0 t_fin delta pos0 speed0 accel_func =
    if t0 >= t_fin then pos0, speed0
    else
        let t1 = t0 + delta
        let accel = accel_func pos0 speed0
        let pos1, speed1 = euler_implicit accel pos0 speed0 delta
        simulate t1 t_fin delta pos1 speed1 accel_func

let pos = 0.0
let speed = 0.0
let t0 = 0.0
let t_fin = 10.0
let delta = 1e-3

let accel = fun pos speed -> (drag 1.0) speed + spring pos + gravity

let pos_fin, speed_fin = simulate t0 t_fin delta pos speed accel
printf "pos = %3.3f\nspeed = %3.3f\n" pos_fin speed_fin;

ignore(Console.ReadKey(true))
```

The following portions of text are new in this fourth tutorial.

```
let spring pos =
    let k = 10.0 in
        -k * pos
```

This defines a function "spring" which takes a position and returns an acceleration.
"k" is a constant denoting the stiffness of the spring. Having a constant stiffness is not very practical, but it shall do for now.

```
let drag k =
    fun speed -> -k * speed
```

The first line defines a function "drag" which takes a single argument "k", denoting the drag coefficient. The second line demonstrates one of the nice features of functional programming languages: it defines an unnamed function taking a speed and returning a speed, all in one line. This function is itself returned by the drag function. In F# functions are no different than other values, and can be returned from functions. "drag" can be seen as a "function factory". You give it a drag coefficient, and it returns a function similar in nature to the "spring" function above.

let rec simulate t0 t\_fin delta pos0 speed0 accel\_func =
Notice how the last argument was renamed from "accel" in Tutorial 3 to "accel\_func".
It is now expected that "accel\_func" should be a function. There is nothing in the syntax that tells the F# compiler about that, the "_func" suffix is just a naming convention intended to users of "simulate"._

```
let accel = accel_func pos0 speed0
```

This line calls accel\_func, passing the position and the speed. The value returned is an acceleration which is passed to "euler\_implicit".

```
let accel = fun pos speed -> (drag 1.0) speed + spring pos + gravity
```

This is just another way of defining a function. I could just as well have written "let accel pos speed = ...".
This function "accel" takes a position and a speed, and returns the sum of three accelerations:
  * a drag, with coefficient 1.0,
  * a spring (whose hidden, constant stiffness was defined to 10.0), and
  * gravity.

```
let pos_fin, speed_fin = simulate t0 t_fin delta pos speed accel
```
The "accel" function is then passed to "simulate", which will use it to sample acceleration each time step.

```
ignore(Console.ReadKey(true))
```
Notice the "ignore" function, which was not present in the previous tutorial. Since F# main programs are supposed to return nothing (which in F# is of type "unit"), and since Console.ReadKey returns something, "ignore" is used to avoid a warning by the F# compiler.

This concludes this fourth tutorial, in which we saw that functions could be created dynamically, returned from and passed to functions, just as any other value (e.g. floats).
In the next tutorial we will see an implementation of the famous Runge-Kutta 4 method of solving differential equations.