# Concurrency #
## Introduction ##

It seems that every tutorial on concurrency has to cite [The Free Lunch is Over: A Fundamental Turn Toward Concurrency in Software](http://www.gotw.ca/publications/concurrency-ddj.htm) by Herb Sutter, and I won't make an exception. Basically, we are not going to get faster CPUs in the future. What we are going to get is more CPUs. The challenge for programmers is then to write software which can take advantage of the extra CPUs.

The code of this Tutorial shows an easy way to achieve that in F#.

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

```
let map_parallel func items =
    let tasks =
        seq {
            for i in items -> async {
                return (func i)
            }
        }
    Async.Run (Async.Parallel tasks)
```

This defines a function 'map\_parallel' taking a function and a list of items as parameters. It is similar to 'Array.map', which was used in Tutorial5: it applies the function 'func' on each item of the list, and produces a list (an array, actually) of the results. What is new here is it does so not in a sequential manner, but in an asynchronous way.

Looking into the function body:

```
    let tasks =
        seq {
            for i in items -> async {
                return (func i)
            }
        }
```

'tasks' is an expression which is used to evaluate items in a collection. Each item is an _asynchronous_ workflow. I won't go into what there are, it is enough for now to see these as independent computations which can be executed concurrently. This is just a definition, these computations are not executing yet.
Each computation will execute function 'func' with the i-th item in 'items' passed as a parameter.

At this point it may seem that actually executing all these computations in parallel and waiting for their completion is a complicated thing to do. Happily, there are functions in the standard library of F# that handle all the tricky business of creating threads, starting them, distributing computations on these threads and waiting for their completion.

```
    Async.Run (Async.Parallel tasks)
```

I don't fully understand what these two functions exactly do, I'm happy enough that they do what I want when composed.

All there is to do in the main program is to replace the call to Array.map by parallel\_map:

```
map_parallel(fun (x,y) -> simulate ef t0 t_fin delta x y))
```

## Conclusion ##

When I run this code on my dual-core, I observe a speed-up by little less than 2, which I find impressive, considering how little change was necessary to execute this code concurrently.