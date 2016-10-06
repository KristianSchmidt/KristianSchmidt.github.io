---
layout: post
title: F# Interactive Pretty Printing with Deedle  
---
In my department at [PFA Pension](http://www.pfa.dk) we use F# Interactive (FSI) a lot in our development. My current project is liability modeling, where I need to keep track of a lot of insurance policies. As such, a lot of collections are being displayed in FSI.

The standard printer in FSI is decent, but I find it gets unwieldy as soon as the types in your collection has more than, say, 3-4 properties. If your type is composed of other types with their own properties, then it's pretty much impossible to glean any useful information from the output.

You can find a couple of different F# pretty-printing solutions by googling, but since we were already using [Deedle](http://bluemountaincapital.github.io/Deedle/) for ad-hoc analyses, and its table based printing is very pretty, I decided to wire it up to print our collections.

## How it works

If we take a look in the `Deedle.fsx` initialization script that ships with Deedle, we'll see a line like this:

```fsharp
do fsi.AddPrinter(fun (printer:Deedle.Internal.IFsiFormattable) -> "\n" + (printer.Format()))
```

This instructs FSI to run a special ToString-method every time it wants to print an object of type `IFsiFormattable`. There's also the method `fsi.AddPrintTransformer : ('T -> obj) -> unit`, where you can return an object to be printed instead of a string.

What we're going to do is take our own collection, add an `AddPrintTransformer` method for it that turns it into a Deedle data frame, which will then be pretty-printed just like the regular Deedle frames.

## An example

First, open Deedle:

```fsharp
#load @"..\packages\Deedle\Deedle.fsx"

open Deedle
```

We'll then start with a (very) simplified policy:

```fsharp
type PolicyState = | Working
                   | Retired

type Policy =
    { PolicyNumber : int
      Age : decimal
      State : PolicyState }

let policyList =
    [ { PolicyNumber = 1; Age = 50m; State = Working }
      { PolicyNumber = 2; Age = 40m; State = Retired } ]
```

Typing `policyList` in FSI will yield something like:

```fsharp
val policyList : Policy list = [{PolicyNumber = 1;
                                 Age = 50M;
                                 State = Working;}; {PolicyNumber = 2;
                                                     Age = 40M;
                                                     State = Retired;}]
```

Not pretty! Try typing `Frame.ofRecords policyList` instead. Then we get

```fsharp
     PolicyNumber Age State                
0 -> 1            50  FSI_0006+PolicyState 
1 -> 2            40  FSI_0006+PolicyState 
```

That's better, but we had to type a different command, and the `PolicyState` type isn't displaying correctly.

To get automatic printing, we'll create a transformer method and add it to FSI.

```fsharp
let printPolicyList (policies : Policy list) =
    Frame.ofRecords policies
    |> box // Boxing is needed because AddPrintTransformer takes a ('T -> obj)

fsi.AddPrintTransformer(printPolicyList)
```

Now the table will be displayed without needing to call `Frame.ofRecords`. There's an important distinction to be made here, though. Just because the Deedle frame is being displayed in FSI it does not mean that `policyList` is suddenly of type `Frame<_,_>`. It's still `Policy list`. The only thing that's changed is the way it's displayed in FSI.

## Even prettier
So far so good, but we still need the finishing touches. To get the `PolicyState` to display correctly, we need to implement a `ToString` method for it.

```fsharp
type PolicyState = | Working
                   | Retired
                   override this.ToString() = sprintf "%A" this
```

Now the table will look like

```fsharp
     PolicyNumber Age State   
0 -> 1            50  Working 
1 -> 2            40  Retired 
```

Much better! The last thing we'll do is to index the frame by the `PolicyNumber`. 

```fsharp
let printPolicyList (policies : Policy list) =
    Frame.ofRecords policies
    |> Frame.indexRowsInt "PolicyNumber"
    |> box // Boxing is needed because AddPrintTransformer takes a ('T -> obj)

val it : Policy list =
     Age State   
1 -> 50  Working 
2 -> 40  Retired 
```

There, now that's much easier to look at!

Note that this indexing only works if you're sure that your index is unique. If it's not, then Deedle will throw an exception when you try to print your collection.

## Tips & tricks
Since the above is just a toy example, it doesn't touch on many of the practical issues you run into with this approach. Here's a couple of tips & tricks for working with this in practice.

### Hiding the definition
When you start taking this approach for more than a couple of types, your script will quickly get cluttered with printer definitions. We've taken the Deedle approach and hidden everything in it's own .fsx file which the user can load from his own script.

### Working with large collections
We often work with collections of over 100k elements, and at that point the transformation to data frame starts to take a significant amount of time. But since Deedle will only display about 50 elements of a collection anyway, it's easy to truncate your own collection in the transformation function. Like so:

```fsharp
let printPolicyList (policies : Policy list) =
    policies
    |> List.take (min 50 policies.Length)
    |> Frame.ofRecords
    |> Frame.indexRowsInt "PolicyNumber"
    |> box // Boxing is needed because AddPrintTransformer takes a ('T -> obj)
```

### Printing a single element
When you get used to the table format for your collections, you also want to use it to display your type even if it's not in a collection. To do this, simply create a transformer function that makes it a single-element collection.

```fsharp
let printPolicy (policy : Policy) =
    printPolicyList [ policy ]

fsi.AddPrintTransformer(printPolicy)
```

### Working with option types
To get the best display of option types, you should convert them to Deedle's `OptionalValue<'T>` instead. An easy way to do this is to map the entire frame after generating it. Say we had a property of type `int option`

```fsharp
let printPolicyList (policies : Policy list) =
    policies
    |> List.take (min 50 policies.Length)
    |> Frame.ofRecords
    |> Frame.mapValues (fun (opt : int option) -> OptionalValue.ofOption opt)
    |> Frame.indexRowsInt "PolicyNumber"
    |> box // Boxing is needed because AddPrintTransformer takes a ('T -> obj)
```

Note that you can't just do `Frame.mapValues OptionalValue.ofOption`. You need to explicitly define the types in the mapping, otherwise it won't work.

### Rounding numbers
If your properties contain a lot of numbers, rounding them off can make the display easier on the eyes. As with the options, we can use the `Frame.mapValues` function to do the rounding on the entire frame.

```fsharp
let printPolicyList (policies : Policy list) =
    policies
    |> List.take (min 50 policies.Length)
    |> Frame.ofRecords
    |> Frame.mapValues (fun (d : decimal) -> Math.Round(d,2))
    |> Frame.indexRowsInt "PolicyNumber"
    |> box // Boxing is needed because AddPrintTransformer takes a ('T -> obj)
```

### Hiding properties
Sometimes you'll have some info on a type that's not needed on the screen. To get rid of these, use `Frame.dropCol "column name"` 
in the transformer function.

```fsharp
let printPolicyList (policies : Policy list) =
    policies
    |> List.take (min 50 policies.Length)
    |> Frame.ofRecords
    |> Frame.dropCol "UnwantedProperty"
    |> Frame.indexRowsInt "PolicyNumber"
    |> box // Boxing is needed because AddPrintTransformer takes a ('T -> obj)
```

### Displaying bonus information
In the transformation function, you're not limited to just providing the new object. You can print extra info in the interactive window too. For example, you can print the size of your collection before displaying it.

```fsharp
let printPolicyList (policies : Policy list) =
    printfn "POLICY COUNT: %i" policies.Length
    policies
    |> List.take (min 50 policies.Length)
    |> Frame.ofRecords
    |> Frame.indexRowsInt "PolicyNumber"
    |> box // Boxing is needed because AddPrintTransformer takes a ('T -> obj)
```

This is particularly handy when using `List.filter`, since you get both the count of the elements that satisfy the predicate along with some examples.

## Summary

Deedle data frames provides a way to get customizable pretty-printing of your own types with relatively little work. We've made the transformer methods for the types we use the most, but definitely not all of them. I recommend giving it a shot if you have a project where you use F# Interactive a lot.