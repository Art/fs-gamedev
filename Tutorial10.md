## Introduction ##

This tutorial shows how to parse .X files, which are used for 3d models and scenes. Note that DirectX offers convenient functions to manipulate these files, meaning that the practical usefulness of the code in this tutorial is limited, unless you don't have access to DirectX.

Be aware that the code in this tutorial is not fully functional. For instance, error reporting is kept to a minimum (error **detecting** is handled, though)

This tutorial demonstrates the following concepts:
  * Discriminated unions and simple pattern matching
  * Stateless lexing
  * Hiding side effects in IO operations
  * Error handling using the `option` type
  * Defining mutually and self dependent types
  * Building functions at run-time by combining basic functions

The content of this tutorial is somewhat more dense than usual.

### The .X format ###

A fairly comprehensive description of the file format is available on MSDN or on http://ozviz.wasp.uwa.edu.au/~pbourke/dataformats/directx/.

An .X file can basically divided into two parts. The first declares data structures, the second defines data, which structured according to the declarations in the first part.

### Parser Architecture ###

There are several options for the design of the parser:
  1. Ignore the data structures declared in the first part, and assume they match the standard definitions by Microsoft; then parse the second part using a "hard-wired" data parser.
  1. Parse the data structure declarations, producing data which is used when parsing the second part.
  1. Parse the data structure declarations, producing parsers which are used to parse the second part.

I find the third option to be the most attractive, mostly for aesthetic reasons. It is therefore the one used for the code of this tutorial.

## Code ##

The full code is available at http://code.google.com/p/fs-gamedev/source/browse/#svn/trunk/Tutorial013.

It is organized into the following modules:
  * **XLexers.fs** The interface of lexers, i.e. functions turning bytes read from a file into data sent to the parser.
  * **TextLexer.fs** An implementation of a lexer which extracts data from text files. .X files may also be binary files, but this case is not handled in this tutorial. **TextLexer.fs** is automatically generated by fslex from **TextLexer.fsl**.
  * **XParsers.fs** The parser.
  * **Program.fs** Entry point for the application.

The rest of this section illustrates each concept.

### Discriminated unions ###

From XLexers.FS:

```
type Token =
    | NAME of string
    | STRING of string 
    | INTEGER of int
    | FLOATVAL of float
    | UUID of string
    | INTEGER_LIST of int list   // Not used ?!
    | FLOAT_LIST of float list   // Not used ?!
    | OBRACE | CBRACE | OPAREN | CPAREN
    | OBRACKET | CBRACKET | OANGLE | CANGLE
    | DOT | COMMA | SEMICOLON
    | TEMPLATE
    | WORD | DWORD
    | FLOAT | DOUBLE
    | CHAR | UCHAR
    | SWORD | SDWORD
    | VOID
    | LPSTR
    | UNICODE
    | CSTRING
    | NSTRING
    | ARRAY
    | EOF

```

This defines a _discriminated union_. A variable of type `Token` can be a `NAME`, in which case its value is a `string`, or a `INTEGER`, in which case its value is an `int`, and so on...

`NAME` and `INTEGER` are called _discriminators_, and are used to identify the actual type of a variable of type `Token`.

### Stateless lexing ###

From XLexers.FS:

```
type Lexer<'Src> = 'Src -> (Token * 'Src) option
```

Another type definition: This is the signature of functions turning file contents into tokens. The type `Lexer` is parametrized by the type of the source from which tokens are extracted. A `Lexer` takes a source, and returns:
  * a pair of a token and a new source, or
  * nothing, if there was an error while reading the source, or if the source is ill-formatted.

Typically, the source contains information about the file being read, and where the 'head' points inside the file.
The lexer is expected to return the source it was passed, with the difference that the new position points after the token that was just read.



```
let expect nextToken (src : 'Src) (tokens : Token list) =
    let rec work src tokens =
        match tokens with
        | [] -> Some src
        | tok :: toks ->
            match nextToken src with
            | Some(head, rest) when head = tok -> work rest toks
            | _ -> None
    work src tokens
```

The function `expect` takes a lexer (somewhat misleadingly called `nextToken`), a source and a list of expected tokens. It checks that the sequence of tokens extracted from the source matches the expected list, then returns the source, pointing after the sequence of tokens. If the lexer returned a token which was not expected, or reached the end of the source too early, `None` is returned.

You may wonder why `nextToken` returns a modified copy of the source it received. The traditional way of doing things modifies the source being passed as an argument, and returns the parsed token or an error code. The method used here avoids the side effect, which is both elegant and practical if you want to be able to do something like:

```
   (* Get next token, but ignore SomeTokenToIgnore *)
   let token, source = nextToken in_file
   let really_token, source2 =
      if token == SomeTokenToIgnore then nextToken source
      else token, source
```

Another example:

```
   (* For backward compatibility reasons, try to parse using the old parser.
      If we fail, use the new parser, which uses a different syntax *)
   let parse_attempt1 = oldParser in_file
   let final_attempt =
      match parse_attempt1 with
      | None -> newParser in_file
      | _ -> parse_attempt1
```

You can do that because `in_file` does not change after calling `nextToken`.

There is an open problem, though. Designing the interface of a lexer that encapsulates the state of the file from which it reads is easy, but may well be unusable. After all, most IO APIs are based on side-effects. The next section shows a solution to this problem.

### Hiding side effects in IO operations ###

From Program.fs:

```
open TextLexer
open System.IO
open Microsoft.FSharp.Text.Lexing

let tokenStream filename =
    seq {
        use stream = File.OpenRead(filename)
        use reader = new StreamReader(stream)
        reader.ReadLine() |> ignore
        let lexbuf = Lexing.from_text_reader (new System.Text.ASCIIEncoding()) reader
        while not lexbuf.IsPastEndOfStream do
            match TextLexer.token lexbuf with
            | XLexers.EOF as value -> yield! [value]
            | token -> yield token
        }
    |> LazyList.of_seq
```

This function takes the name of file to read.

It creates a _sequence expression_, that is a piece of code that generates items in a sequence. Sequences are abstract types which can be converted to concrete lists, arrays. They can also be evaluated lazily, which means that the content of the sequence is not instantiated at once when the sequence is created. Instead, items are created on demand.
In the example above, the sequence is turned into a lazy list. Lazy lists are similar to normal lists, with the difference that items are created when needed.

This sequence expression does the following:
  1. Opens the file to read
  1. Creates a stream reader to read from this file
  1. The first line is retrieved, and ignored
  1. A lexer, generated using fslex, is instantiated; it will read from the stream reader
  1. The lexer reads tokens one by one, building a sequence of tokens, until the end of the file is reached

Individual items are produced using `yield`, chunks of items can be produced at once using `yield!`.

Note that the last step does not actually read the entire file at once. Instead, each iteration will be executed when a new token is needed, at which point execution continues in the main program.

The type `seq` (which is instantiated by the sequence expression) is similar to the type `list`. It has operations allowing to retrieve items, but unlike `list`, it does not allow pattern-matching. This minor issue is easily solved by converting the sequence to a `LazyList`. The following is now possible:

```
// A lexer working on a lazy list of Tokens. The list is constructed on the fly as the file is read
let nextTokenLazyList (debug : bool) (src : LazyList<XLexers.Token>) =
    match src with
    | LazyList.Nil -> None
    | LazyList.Cons(head, rest) ->
        if debug then printfn "%A" head;
        Some(head, rest)
```

This function takes an argument controlling whether debug info is printed, and a lazy list of tokens from which tokens are read and returned, one by one.

Note that calling `nextTokenLazyList` twice with the same argument will produce the same result, as one would expect from a pure function.

### Writing a lexer using fslex ###

See TextLexer.fsl for an example. This tutorial will not go into further details. If you are familiar with lex you should not have difficulties understanding the content of this file. Otherwise, you may want to look into one of the many tutorials available on lex on the net, for instance http://epaperpress.com/lexandyacc/. This tool was originally written for the C programming language, and derivatives were made for almost all new programming languages that came after C.

### Error handling using the `option` type ###

A common way to handle errors is to use the special value `null`. For instance, functions allocating memory return a reference to a newly allocated chunk of memory, or `null` if the allocation failed, maybe due to a lack of available memory. Although convenient, this trick has introduced a family of bugs, so-called null-pointer exceptions, which occur when trying to use a reference which is actually `null`.

The `option` type in F# is used where the special value `null` is typically used in other languages. It can be seen as a parametrized discriminated union:

```
type Option<'T> =
  | None
  | Some of 'T
```

For efficiency reasons, the F# compiler replaces instances of `option` by `null` and valid references.

The point of `option` is that it makes it possible to write code that is safe with respect to null-pointer exceptions.

Here is an example illustrating the use of `option`:
```
// Parse the entire file.
let parse (lexer : LexerFuncs<'Src>) src : (string option * Record) list option =
    // Function parsing all templates
    let rec loop src parsers =
        match parseTemplate lexer src parsers with
        | Some(parsers, src) -> loop src parsers
        | None -> parsers, src
    
    // First parse all templates
    let env, src = loop src Map.empty
    
    // Then parse the data
    let r = parseData lexer src env
    match r |> lexer.MaybeExpect [EOF] with
    | Some(value, _) -> Some value
    | None -> None
```

This function is the top-level parsing function. It parses the first part of an .X file, which contains the declarations of data structures. The .X file format does not have an explicit separator between the two parts, which means that one basically has to try and read the first part until it fails, at which point an attempt to read the second part can be made.

The local function `loop` repeatedly reads data from the file, parsing the data it reads as data structure declarations. When `parseTemplate`, the function parsing data structures, returns `None`, `loop` returns the result of the successful part of parsing.

The rest of `parse` continues the parsing process, _where successful parsing of data structures stopped_. Note that this would have been trickier to do using a stateful lexer and file reader: By the time `parseTemplate` notices that it's not parsing a data structure declaration, several tokens may already been read and discarded. To solve this problem, one must leave a 'bookmark' in the file reader after each successful parsing of a data structure declaration, which is easily and elegantly done when using stateless lexing: just keep around the `src` variable used when parsing failed.

The common mistake when using `null` is to use a pointer or reference without checking it first. Performing exactly those checks which are necessary is non-trivial to implement once, and even harder to maintain. Using a discriminated union has the advantage that the 'nullness' of references becomes a static property made explicit in the syntax. A program with missing checks for `None` cannot be successfully compiled.

### Mutual and self dependent types ###

From XParser.fs:

```
type Record = {
    type_name  : string
    components : Map<string, obj>
    children   : SubRecord list
}
and SubRecord =
    | RefByName of string
    | RefByUUID of string           // Not implemented
    | FullRef   of string * string  // Not implemented
    | Nested    of Record
```

This defines two types, `Record` and `SubRecord`, which are mutually dependent. This type is used when parsing the second part of the file to store the data.

```
type DataParser<'Src> = DataParser of (Env<'Src> -> LexerFuncs<'Src> -> 'Src -> Record -> (Record * 'Src) option)
and Env<'Src> = Map<string, DataParser<'Src>>
```

This defines a type `DataParser` which is parametrized by the type of source from which tokens are extracted. Concretely, this type is used with a lazy list. A `DataParser` is basically a function which takes a lexer, a source, a record, and returns `None` or a new record and a new source.
The new record is the record passed as argument, extended with the data that was just passed.
I have omitted `Env<'Src>`, which a type used to hold all DataParsers produced so far. Data in .X files is organized as objects which can be nested, which means that DataParsers may need to use other DataParsers to parse nested objects.

These two pieces of code illustrate how to declare mutually dependent types: simply use keyword `and` instead of `type` in all but the first declaration.

The second example shows a trick allowing to define types which depend on themselves. The trick consists of wrapping the type in a discriminated union. The F# compiler rejects the following code, which lacks the discriminated union:

```
type DataParser<'Src> = Env<'Src> -> LexerFuncs<'Src> -> 'Src -> Record -> (Record * 'Src) option
and Env<'Src> = Map<string, DataParser<'Src>>
```

### Building functions at run-time by combining basic functions ###

From XParser.fs

```
type ComponentParser<'Src> = Env<'Src> -> LexerFuncs<'Src> -> 'Src -> (obj * 'Src) option
```

This type, similar to `DataParser`, represents functions which parse text from the second part of an .X file, to extract data. `Record`, which is used by `DataParser`, represents composite data. `obj`, which is the standard basic type of all objects in F#, is used by `ComponentParser` to represent data which is itself part of other data.

```
// Does no parsing, return the record untouched
let empty_data_parser : DataParser<'Src> =
    let f _ lexer src record = Some(record, src)
    DataParser f


// Parses a ";", returns the record untouched
let semi_colon_parser : DataParser<'Src> =
    let f  _ (lexer : LexerFuncs<'Src>) src record =
        match lexer.Expect src [SEMICOLON] with
        | Some(src) -> Some(record, src)
        | None -> None
    DataParser f
```

These two functions are examples of simple `DataParser`s which don't do much.

```
// Parses an int, returns it, boxed.            
let int_parser : ComponentParser<'Src> =
    fun _ lexer src ->
        match lexer.NextToken src with
        | Some(INTEGER(value), src) -> Some (box value, src)
        | _ -> None

        
// Parses a float, returns it, boxed.             
let float_parser : ComponentParser<'Src> =
    fun _ lexer src ->
        match lexer.NextToken src with
        | Some(FLOATVAL(value), src) -> Some (box value, src)
        | _ -> None

        
// Parses a string, returns it, boxed.             
let string_parser : ComponentParser<'Src> =
    fun _ lexer src ->
        match lexer.NextToken src with
        | Some(STRING(value), src) -> Some(box value, src)
        | _ -> None
```

The functions above are `ComponentParser`s. They could have equivalently been defined using the more common syntax:

```
// Parses an int, returns it, boxed.            
let parseInt _ lexer src : ComponentParser<'Src> =
    match lexer.NextToken src with
    | Some(INTEGER(value), src) -> Some (box value, src)
    | _ -> None
```

The first form can be seen as a variable holding a function, whereas the second form directly defines a function. The first form was chosen to reflect the fact that these functions are treated as data.

```
// Turn a ComponentParser to a DataParser. The boxed value is added in the Record with name 'name'        
let convertComponentToDataParser name (comp_parser_func : ComponentParser<'Src>) : DataParser<'Src> =
    fun env lexer src (record : Record) ->
        match comp_parser_func env lexer src with
        | Some(value, src) -> Some(record.Add(name, value), src)
        | None             -> None
    |> DataParser
```

As the comment says, this function converts a `ComponentParser` into a `DataParser`.

`DataParser`s can then be composed to build more complex parsers using the functions below.

```
// Makes a parser that executes two parsers one after another.                
let composeDataParsers (parser1 : DataParser<'Src>) (parser2 : DataParser<'Src>) : DataParser<'Src> =
    let f env lexer src record =
        match parser1, parser2 with
        | DataParser(parser1), DataParser(parser2) ->
            match parser1 env lexer src record with
            | Some(record, src) -> parser2 env lexer src record
            | None              -> None
    DataParser f


// Infix operator for composeDataParsers   
let (>>>>) parser1 parser2 =
    composeDataParsers parser1 parser2
```

In order to parse an int, followed by a semi-colon, an `int_parser` can be combined with a `semi_colon_parser`:

```
    let my_parser = int_parser >>>> semi_colon_parser
```

Parsing text using `my_parser` is done as shown here:

```
    let env : Env<'Src> = ...
    let lexer : LexerFuncs<'Src> = ...
    let src : 'Src = ...

    my_parser env lexer src (Record.empty.SetName(name))
```

## Conclusion ##

This tutorial exposes a number of features and techniques that were new in these series. It may therefore seem a bit excessively dense. In my opinion, the key feature resides in the choice of architecture. It was clear parsing would have to be split in two stages: first parse the data structure definitions, then the structured data itself.
I had initially designed the first parser to produce information describing the data structure. The second parser would basically be composed of a large selection on the type of expected data:

```
    let expected_data = ...

    let new_data =
        match expected_data with
        | INT -> parseInt lexer src data
        | STRING -> parseString lexer src data
        | FLOAT -> parseFloat lexer src data
        ...
```

I think my second design, presented here, leads to a much more elegant second-stage parser:

```
    let parser = ...

    let new_data =
        parser lexer src data
```

In my implementation, the first-stage parser is located in function `parseTemplateContent` in [XParser.fs](http://code.google.com/p/fs-gamedev/source/browse/trunk/Tutorial013/XParser.fs?r=40#228). I reckon it is somewhat hard to read, mostly because of its length. Although I have not implemented my initial idea, I expect both the first and second-stage parsers would be equally large. The second-stage parser shown in the source code of this tutorial is considerably shorter.