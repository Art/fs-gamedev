# Optimized RK4 #

## Introduction ##

This tutorial shows how to improve the performance of the implementation of RK4 shown in Tutorial6, using the following features of F#:
  * the "inline" keyword,
  * using structs instead of tuples, and
  * the "OptimizedClosures" type.


## Code ##

The full code is also available [there](http://code.google.com/p/fs-gamedev/source/browse/#svn/trunk/Tutorial007).

```
#light

open System

let run_time_func (func: unit -> 'a) =
    let time0 = System.DateTime.Now
    let res = func ()
    let diff0 = System.DateTime.Now - time0
    (diff0, res)

let map_parallel func items =
    let tasks =
        seq {
            for i in items -> async {
                return (func i)
            }
        }
    Async.Run (Async.Parallel tasks)

type ps<'num> = struct
    val pos:'num
    val speed:'num
    new(pos, speed) = { pos = pos; speed = speed }
end

type sa<'num> = struct
    val speed:'num
    val accel:'num
    new(speed, accel) = { speed = speed; accel = accel }
end
        
let gravity = 
    -9.81

let inline spring pos =
    let k = 100.0 in
        -k * pos

let inline drag k pos speed = -k * speed
    
let inline euler_implicit accel_func pos speed delta =
    let speed2 = speed + delta * accel_func pos speed in
        ps<_>(pos + delta * speed2, speed2)

let inline adapt (f: 'num -> 'num -> 'num) = 
    fun pos speed -> 
        let accel = f pos speed in
            sa<_>(speed, accel)

let inline runge_kutta_4 deriv_func pos speed delta =
    let half_delta = 0.5 * delta
    let sixth_delta = delta / 6.0
    let sa1: sa<_> = deriv_func pos speed
    let sa2 = deriv_func (pos + half_delta * sa1.speed) (speed + half_delta * sa1.accel)
    let sa3 = deriv_func (pos + half_delta * sa2.speed) (speed + half_delta * sa2.accel)
    let sa4 = deriv_func (pos + delta * sa3.speed) (speed + delta * sa3.accel)
    let pos1 = pos + sixth_delta*(sa1.speed + 2.0 * sa2.speed + 2.0 * sa3.speed + sa4.speed)
    let speed1 = speed + sixth_delta*(sa1.accel + 2.0 * sa2.accel + 2.0 * sa3.accel + sa4.accel) in
        ps<_>(pos1, speed1)
    
let inline simulate intg_func t0 t_fin delta pos0 speed0 =
    let oc = OptimizedClosures.FastFunc3<_,_,_,_>.Adapt(intg_func)
    let rec repeat t0 pos0 speed0 =
        if t0 > t_fin then (pos0, speed0)
        else
            let t1 = t0 + delta
            let ps: ps<_> = oc.Invoke(pos0, speed0, delta)
            repeat t1 ps.pos ps.speed
    repeat t0 pos0 speed0


let initial_states = [ for s in 1..100 -> (float s, 0.0) ]    
let t0 = 0.0
let t_fin = 1000.0
let delta = 0.005

let accel = fun pos speed -> (drag 1.0) pos speed + spring pos + gravity

let ef = (euler_implicit accel)
let rk4f = (runge_kutta_4 (adapt accel))

for i in 0..3 do
    let (run_time, res) = 
        run_time_func (fun () -> 
            initial_states |> List.map(fun (x,y) ->
                simulate ef t0 t_fin delta x y))
    printfn "Euler single: %f" run_time.TotalSeconds;

    let (run_time, res) =
        run_time_func (fun () ->
            initial_states |> map_parallel(fun (x,y) ->
                simulate ef t0 t_fin delta x y))
    printfn "Euler multi: %f" run_time.TotalSeconds;

    let (run_time, res) =
        run_time_func (fun () ->
            initial_states |> List.map(fun (x,y) ->
                simulate rk4f t0 t_fin delta x y))
    printfn "RK4 single: %f" run_time.TotalSeconds;

    let (run_time, res) =
        run_time_func (fun () ->
            initial_states |>
                map_parallel(fun (x,y) ->
                    simulate rk4f t0 t_fin delta x y))
    printfn "RK4 multi: %f" run_time.TotalSeconds;
done;

ignore(Console.ReadKey(true))
```

## Details ##

### struct ###
```
type ps<'num> = struct
    val pos:'num
    val speed:'num
    new(pos, speed) = { pos = pos; speed = speed }
end
```

This defines a struct with two data members named pos and speed, and a constructor taking (oh surprise!) a position and a speed. The parts between curly braces means "create a value of type ps where the pos member is set to the parameter pos, and the speed member is set to the speed parameter".
This struct is parametrized by a type 'num', which is used to represent the position and speed. In the single-dimensional case 'num' would typically be float, in higher dimensions vector types would be used.

This struct is used later in the program in place of tuples. In F#, tuples are non-primitive types which are dynamically allocated and whose deallocation is handled by the garbage collector. This can be slow in performance-critical parts of the code. The "struct" type was introduced to alleviate this problem. Storage for "struct" values is allocated locally on the stack, which keeps the garbage collector out of the picture.

There are however some downsides:
  * "struct" values are passed by value in function calls, which may be slower than using passing by reference for anything but very small objects (128 bytes or less);
  * structs have a default constructor which initializes all its members to 0 or null, which may cause bugs and crashes such as NullPointer exceptions.

### inline ###

```
let inline spring pos =
    let k = 100.0 in
        -k * pos
```

This tells the F# compiler to emit the body of function 'spring' whenever it is called. In the .NET platform, inlining is normally a run-time optimization reserved to the CLR, and AFAIK, other .NET languages lack the "inline" keyword. F# does however have the "inline" keyword, possibly because F# supports advanced features related to generics which are not supported by the current implementation of generics in the CLR (e.g. type constraints on class methods). These features require the possibility to perform inlining at compile-time.

Inlining can improve performance for two reasons:
  * For very short functions, the overhead associated to calling the function may be large enough that inserting the body of the function at the call site is preferable.
  * The F# compiler performs complex optimizations which are made possible when the code of the function that is called in inserted into the code of the caller. For instance, parameters may be replaced by constants, and unused code (in the call considered) can be removed.

The CLR is supposed to do a good job for the first case, but it will never be able to generate code as optimized as a compiler would for instances of the second case.

All this may be a bit vague, but later tutorials will show concrete examples of interesting optimizations the F# compiler can perform on inlined code.

### OptimizedClosures ###

```
let inline simulate intg_func t0 t_fin delta pos0 speed0 =
    let oc = OptimizedClosures.FastFunc3<_,_,_,_>.Adapt(intg_func)
    let rec repeat t0 pos0 speed0 =
        if t0 > t_fin then (pos0, speed0)
        else
            let t1 = t0 + delta
            let ps: ps<_> = oc.Invoke(pos0, speed0, delta)
            repeat t1 ps.pos ps.speed
    repeat t0 pos0 speed0
```

I have no idea what "OptimizedClosures" does, so I won't write much about that here. Citing Jomo Fisher (see http://blogs.msdn.com/jomo_fisher/archive/2008/09/16/f-performance-tweaking.aspx ):

> Closure optimization can be used when a function will be called many times. It allows F# to do a bit of expensive work up front that will later be amortized across the many calls to the function. Unfortunately, this is not something that you can stick everywhere and just get a benefit (otherwise, the compiler could just inject it for you). You should definitely profile before and after.

### Moving creation of closures outside of loops ###

```
let ef = (euler_implicit accel)
let rk4f = (runge_kutta_4 (adapt accel))
```

'ef' and 'rk4f' are functions which are partial applications of functions 'euler\_implicit' and 'runge\_kutta\_4'. They are also closures, as they refer to 'accel', which "lives" in the top level.
Closures are turned by the F# compiler into function objects, i.e. objects which have an "Invoke" method called when the closure is called. As any other object, they must be instantiated at some point. Such instantiations can be done once, and should therefore be moved out of loops to avoid repeatedly creating multiple identical instances.

This is how the old code looked:
```
    let final_states = initial_states |> Array.map(fun (x,y) -> simulate (euler_implicit accel) t0 t_fin (0.25*delta) x y)
```

## Conclusion ##

Together, these optimizations allow for a ~3X speed up.
In the next tutorials, we will move this integrator to the multi-dimensional world. We will show how generic programming can be used to decouple the implementation of the integrator from the implementation of multi-dimensional vectors.