This tutorial shows an implementation of the Runge-Kutta (order 4). The entire code follows below, followed by explanations on the new features since Tutorial 4.

```
#light

open System

let gravity =
    -9.81

let spring pos =
    let k = 10000.0
    -k * pos

let drag k =
    fun pos speed -> -k * speed

let euler_implicit accel_func pos speed delta =
    let speed2 = speed + delta * accel_func pos speed
    pos + delta * speed2, speed2

let adapt (f: 'num -> 'num -> 'num) =
    fun pos speed ->
        let accel = f pos speed
        speed, accel

let runge_kutta_4 accel_func pos speed delta =
    let half_delta = 0.5 * delta
    let deriv_func = adapt accel_func
    let s1, a1 = deriv_func pos speed
    let s2, a2 = deriv_func (pos + half_delta * s1) (speed + half_delta * a1)
    let s3, a3 = deriv_func (pos + half_delta * s2) (speed + half_delta * a2)
    let s4, a4 = deriv_func (pos + delta * s3) (speed + delta * a3)
    let pos1 = pos + delta/6.0*(s1 + 2.0*s2 + 2.0*s3 + s4)
    let speed1 = speed + delta/6.0*(a1 + 2.0*a2 + 2.0*a3 + a4)
    pos1, speed1

let simulate intg_func t0 t_fin delta pos0 speed0 =
    let rec repeat t0 t_fin pos0 speed0 =
        if t0 >= t_fin then pos0, speed0
        else
            let t1 = t0 + delta
            let pos1, speed1 = intg_func pos0 speed0 delta
            repeat t1 t_fin pos1 speed1
    repeat t0 t_fin pos0 speed0


let pos = 0.0
let speed = 0.0
let t0 = 0.0
let t_fin = 500000.0
let delta = 0.025

let accel = fun pos speed -> (drag 100.0) pos speed + spring pos + gravity

let time0 = System.DateTime.Now
let (pos_fin, speed_fin) = simulate (euler_implicit accel) t0 t_fin delta pos speed
let diff0 = System.DateTime.Now - time0
printfn "%f pos = %f speed = %f" diff0.TotalSeconds pos_fin speed_fin;

let time0 = System.DateTime.Now
let (pos_fin, speed_fin) = simulate (runge_kutta_4 accel) t0 t_fin delta pos speed
let diff0 = System.DateTime.Now - time0
printfn "%f pos = %f speed = %f" diff0.TotalSeconds pos_fin speed_fin

ignore(Console.ReadKey(true))
```

Before diving into the explanations, let us have a look at the definition of Runge-Kutta 4, according to wikipedia:

> Let an initial value problem be specified as follows.

> ![http://upload.wikimedia.org/math/7/3/6/73654ee739d62e6612a2357db9d044f6.png](http://upload.wikimedia.org/math/7/3/6/73654ee739d62e6612a2357db9d044f6.png)

> Then, the RK4 method for this problem is given by the following equations:

> ![http://upload.wikimedia.org/math/0/4/9/049a79910042e7b201b0c458ae649a8e.png](http://upload.wikimedia.org/math/0/4/9/049a79910042e7b201b0c458ae649a8e.png)

> where yn + 1 is the RK4 approximation of y(tn + 1), and

> ![http://upload.wikimedia.org/math/6/c/4/6c44aec9cdaee43dd8f2cd50c2b41afa.png](http://upload.wikimedia.org/math/6/c/4/6c44aec9cdaee43dd8f2cd50c2b41afa.png)

In the context of this tutorial, function "f" can be seen as a function computing the speed and the acceleration, depending on the current time "t" and the current position and speed "y". Actually, in this tutorial, the forces do not depend on the current time, and the first parameter of "f" is therefore omitted.
Using a functional language like F# to implement Runge-Kutta 4 (RK4 for short) is particularly fitting, as the function "f" is an input to the algorithm. It seems easy to implement this algorithm using a function taking a function "f", and indeed it is:
```
let runge_kutta_4 accel_func pos speed delta =
    let half_delta = 0.5 * delta
    let deriv_func = adapt accel_func
    let s1, a1 = deriv_func pos speed
    let s2, a2 = deriv_func (pos + half_delta * s1) (speed + half_delta * a1)
    let s3, a3 = deriv_func (pos + half_delta * s2) (speed + half_delta * a2)
    let s4, a4 = deriv_func (pos + delta * s3) (speed + delta * a3)
    let pos1 = pos + delta/6.0*(s1 + 2.0*s2 + 2.0*s3 + s4)
    let speed1 = speed + delta/6.0*(a1 + 2.0*a2 + 2.0*a3 + a4)
    pos1, speed1
```

There is a small catch here: The main program builds an "accel\_func" which returns a single number denoting the acceleration, where "runge\_kutta\_4" expects a function returning a pair denoting the speed and the acceleration. This small problem is solved using the "adapt" function:

```
let adapt (f: 'num -> 'num -> 'num) =
```

"adapt" is a function taking a parameter "f". To make things easier to understand for the reader (and for the writer too:), a type annotation is used. The type annotation means "a function taking some type num, and returning a function taking the same type num, which returns a value of type num", or in other words, "a function taking two parameters of type num and returning a value of type num". The single quote before "num" means that num is a type variable, or in C++ terminology, a template parameter. In these tutorials I have used one-dimensional floats, but a typical 3d game would use a 3-dimensional vector. "float -> float -> float" would also have worked, but would be less generic. I could also have left out the type annotation altogether, and let the F# compiler figure out the types.

```
fun pos speed ->
    let accel = f pos speed
    speed, accel
```

I hope no one is getting headaches yet :) "adapt" returns a function taking a position and a speed and returning the speed and the acceleration which is computed by calling the "f" function. "adapt" is indeed an adapter, but instead of working with values, as one would probably do in non-functional languages, "adapt" manipulates functions directly.

```
let euler_implicit accel_func pos speed delta =
    let speed2 = speed + delta * accel_func pos speed
    pos + delta * speed2, speed2
```

Not much new here, just notice how the "accel" parameter was renamed to "accel\_func". "euler\_implicit" has been modified in this installment to have the same signature as "runge\_kutta\_4".

```
let (pos_fin, speed_fin) = simulate (euler_implicit accel) t0 t_fin delta pos speed
```

Here is the beautiful bit: the main program calls simulate, passing the integration method ("euler\_implicit"). The simulation function is generic enough to work independently of the integration method.
Notice also how "(euler\_implicit accel)" is passed as an argument, which is a function built by "plugging" the "accel" function into "euler\_implicit". The "simulate" function never sees the "accel" function.

This tutorial also shows how to time the execution of the program using a function from the .NET framework. I have implemented RK4 in C++ in order to compare the execution times. The C++ implementation outperforms the F# version by a factor of 10, but [Tutorial6](Tutorial6.md) shows how to optimize the F# version without sacrificing functional purity.