Following the tradition, here is the code for a program printing "Hello world" in F#.

```
#light
open System
printf "Hello world\n"
Console.ReadKey(true)
```

Below follows an explanation of each line.

```
#light
```
This tells the compiler we are using the lightweight syntax. Apparently the syntax of F# comes in several flavors. This syntax works well for me.

```
open System
```
This includes a library containing the function called at the end to wait for a key press.

```
printf "Hello world\n"
```
Prints Hello world on the console, and goes to the next line.

An alternative would have been to use:
```
printfn "Hello world"
```

The two differ in that the first uses a UNIX-style of end-of-line, while the second uses the end-of-line of your environment. Personally I prefer to have one type end of lines regardless of the system on which the program runs, but that probably does not fit everyone's needs.


```
Console.ReadKey(true)
```
Waits for a key press.

That was it for now. We have learned the bare minimum to do some simple text I/O. In the next tutorial we will see how to use functions to implement the Euler integration method.