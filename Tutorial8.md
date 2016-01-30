# Generic programming #

## Introduction ##

I have decided to continue this series by extending the integrator to handle 2d and 3d. One of the annoyances an amateur game developer encounters is the multiplication of 3d vector classes. They all use the same data layout, all offer the same methods, all offer the same performance, yet there is at least one such class per library. For instance, in my [go-kart racing game](http://www.top10-racing.org), I have to deal with 3d vectors in OpenAL, 3d vectors in CGAL (which I used for DeLaunay triangulation), and should I start using ODE (a physics library), I'll have to deal with yet another type of 3d vectors. To make things worse, I am myself guilty of having implemented my own 3d vector math library.

The type inference feature in F# makes it easy to avoid introducing yet another 3d vector class. When the obligation to explicitly annotate all your variables with types disappears, writing polymorphic code becomes easy.

Normally I would have started by showing the code, but in this case I thought I needed to justify why I should change the code at all. Especially if you come from C++, you might think that running the integration in 2d or 3d would just be a matter of changing the kind of vectors passed to the 'simulate' and 'accel' functions. As long as the new vector types support `operator+` and `operator*` we should be fine.

You would be wrong. Consider:

```
let x = 0.5 * a
```

I am not sure of the reasons behind this design decision, but in F#, this means that "a" and "x" are inferred to be the same type as 0.5, which is float. Moreover, the multiplication operator is now constrained to be applicable only on floats. That's right, you are no longer allowed to write `2 * 3`. Like it or not, that's the way it is.

In practice, this means that I am probably best off avoiding operators `+` and `*`. Instead, the caller of my functions will have to tell them what functions to use to perform arithmetic operations.

## Code ##

The full code is available [there](http://code.google.com/p/fs-gamedev/source/browse/trunk/Tutorial008/Program008.fs).

It's getting a bit long, therefore I won't put the whole copy here. Let us go to the interesting bits instead.

## Details ##

```
type Vec2d = struct
    val x: float
    val y: float
    
    new (x, y) = { x = x; y = y }
    
    static member plus ((a: Vec2d), (b: Vec2d)) = Vec2d(a.x + b.x, a.y + b.y)
    static member scale ((k: float), (a: Vec2d)) = Vec2d(k * a.x, k * a.y)
end
```

This defines a two-dimensional vector type supporting addition through the static method 'plus', and scalar product by a float through the static method 'scale'.

These methods do not use the curried form, i.e. they cannot be partially applied. When called, both arguments must be provided. That's the way it should be for all object types. Indeed, object types must be compatible with .NET's idea of object and methods, and curried forms do not normally exist in the world of .NET.

Note that it's not forbidden to use curried forms for methods, it's just that it will result in slow code because there are optimizations the F# compiler does not perform when dealing with methods (even static ones) instead of functions.

```
type RVec3d = {
    x: float
    y: float
    z: float
    }
    with    
    static member plus ((a: RVec3d), (b: RVec3d)) = { x = a.x + b.x; y = a.y + b.y; z = a.z + b.z }
    static member scale ((k: float), (a: RVec3d)) = { x = k * a.x; y = k * a.y; z = k * a.z }
```

Just for the fun of it I'm implementing a 3d vector using vectors instead of a struct or a class. Not much to say there, however note that the same comment as above goes for curried forms and methods of records.

```
type VecOps<'vector, 'scalar> = {
    up: 'vector
    add: ('vector * 'vector) -> 'vector
    scale: ('scalar * 'vector) -> 'vector
}
```

This defines a so-called _dictionary of operations_.
The first line defines a type called VecOps which is parametrized by two types, denoting respectively the vector type and the scalar type.
The second line is used to specify what is the upward direction. It's used only by function 'gravity'. The third and second lines define the type of the addition and scalar product operations.

The functions which perform operations on vectors have been modified as shown below:

```
let inline runge_kutta_4 (vec_ops: VecOps<'a, 'b>) deriv_func pos speed delta =    
    let extrapolate x delta dx = 
        vec_ops.add (x, (vec_ops.scale (delta, dx)))

    let half_delta = 0.5 * delta
         
    let s1, a1 = deriv_func pos speed
    let s2, a2 = deriv_func (extrapolate pos half_delta s1) (extrapolate speed half_delta a1)
    let s3, a3 = deriv_func (extrapolate pos half_delta s2) (extrapolate speed half_delta a2)
    let s4, a4 = deriv_func (extrapolate pos delta s3)      (extrapolate speed delta a3)

    let sixth_delta = delta / 6.0    

    let update old v1 v2 v3 v4 =
        let (*) k v = vec_ops.scale(k, v)
        let (+) u v = vec_ops.add(u, v)
        old + sixth_delta * (v1 + 2.0 * (v2 + v3) + v4)
                             
    let pos1 = update pos s1 s2 s3 s4
    let speed1 = update speed a1 a2 a3 a4
    pos1, speed1
```

A new parameter was added to hold the functions to apply on vectors to perform additions and products.

Some may find the code harder to read, as `+` and `*` were replaced by 'vec\_ops.add' and 'vec\_ops.scale'. It's still possible to use `+` and `*`, as shown in the local function 'update'. It seems the restriction mentioned in the introduction about operators being constrained to specific types after a first application is limited to the current scope, which means that this restriction is not very limiting in practice, provided you don't mind the extra "let" bindings.

To avoid repeatedly passing VecOps objects to functions in the physics library, users can introduce short-cuts using partial application and function closures:

```
let ef_Vec1d = euler_implicit Vec1D_ops (accel1D)
let rk4_Vec1d = runge_kutta_4 Vec1D_ops (accel1D')
```

## Conclusion ##

In [Tutorial9](Tutorial9.md), we'll see a technique similar to specialization to allow for efficient instantiations of the code when dealing with large structures.