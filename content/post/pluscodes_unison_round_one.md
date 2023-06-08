---
title: "Generating PlusCodes with Unison - Round 1"
date: "2023-06-07"
toc: true
#author: "Kevin Hoffman"
type: post
description: "Generating PlusCodes with Unison - Round 1"
subtitle: "A brutal struggle with porting existing code to Unison"
categories: 
    - "unison"
    - "pluscode"
---
_Attempting to port my old Elixir Open Location Code (PlusCode) library to Unison, and failing over and over and over._

<!--more-->

## What are PlusCodes?
Most of us are familiar with the Global Positioning System (GPS). Our phones have it, our cars have it, even some of our luggage has it. GPS divides the earth into slices. The combination of an east-west slice and a north-south slice can give you any point on the planet.

{{< figure
  src="/images/gps_globe.webp"
  class="class param"
  type="margin"
  label="mn-tesla"
  title="GPS Globe"  
  caption="Enyclopedia entry for GPS"    
  alt="Globe divided into degrees"  
 >}}
This system of using two floating point values to represent a point on the planet works extremely well for all kinds of purposes. However, sometimes we want to represent _regions_ of space on the planet - a 9'x9' space or a 300km x 300km space, etc.

We also want a system that is simple and easy to use. We want to be able to know immediately whether one region is within another. Even more, we'd like to be able to represent these regions in a human-readable fashion.

A [plus code](https://maps.google.com/pluscodes/) is like a street address, but for locations that aren't or can't be represented as an address. As an example, the code `87G8Q2PQ+94` represents a small rectangle in the middle of the reservoir in Central Park, New York.

Some time ago, I was planning on using open location codes (OLC) in an augmented reality application for off-road vehicles. People would be able to use topographical maps (with no roads, obviously) and drop region markers highlighting obstacles, vehicle damage, fun bits, gathering fields, etc. 

It would also be super easy to tell if a given location was within the designated trail boundaries or not. If one code starts with the same digits as another, that region is _within_ the other. The matching prefix digits form a code that defines a broader region. Rather than having to do super complicated GPS math, you can just use functions like `startsWith` to check for collisions and containment. The longer the code, the smaller and more precise the region.
 {{< section "end" >}}

There are countless potential applications for a simple mechanism like this. Another use that I want to explore is for gaming. If you adjust the math to account for a different distance between degrees (e.g. a fictional moon), then you could use these codes in a game and also leverage the fast collision and contains checks.

## Reviewing my Elixir Library
Google has a [GitHub repository](https://github.com/google/open-location-code/tree/main) that contains a vast collection of implementations of the PlusCode algorithms. It doesn't contain an Elixir version, and so that's why I wrote my own Elixir port of the library some time ago. You can find my implementation in my [GitHub repo](https://github.com/autodidaddict/openlocationcode/tree/master).

Here's the `encode_pairs` function (you can see all the helper functions in the GitHub repo). This function is called by `encode` and performs recursive computations to generate a code from GPS coordinates.

```elixir
 defp encode_pairs(adj_latitude, adj_longitude, code_length, code, digit_count) when digit_count < code_length do
    place_value = (digit_count / 2)
                  |> floor                  
                  |> resolution_for_pos
      
    {ncode, adj_latitude} = append_code(code, adj_latitude, place_value)
    digit_count = digit_count + 1
    
    {ncode, adj_longitude} = append_code(ncode, adj_longitude, place_value)
    digit_count = digit_count + 1

    # Should we add a separator here?
    ncode = if digit_count == @separator_position and digit_count < code_length do
      ncode <> @separator
    else 
      ncode        
    end
    
    encode_pairs(adj_latitude, adj_longitude, code_length, ncode, digit_count)    
  end

  defp encode_pairs(_, _, code_length, code, digit_count) when digit_count == code_length do   
    code 
      |> pad_trailing 
      |> ensure_separator
  end 
```

I make no claims about the cleanliness or even reliability of this code.

## Porting to Unison
Porting code from one language to another isn't usually a painful process for me. The big time consumer and source of frustration is often the "language brain" mistakes where I use a `}` in a language that doesn't use brackets and I'll mix and match syntax all over the place. It usually takes a few hours for me to stop bleeding context across languages.

When I set out to port this to Unison, I thought it would be straightfoward. I couldn't have been more wrong. There were moments working on this where I would burn through _4 hours_ just trying to get simple syntax working. I was plagued by everything from compilation failures due to bad indentation, me forgetting that I can't use infix operations everywhere, and probably the most annoying of all: float and whole number conversions.

Let's take a look at the Unison version of the `encode_pairs` function. This function recurses as it builds up the code until it reaches the desired length.

```haskell
encodePairs : Float -> Float -> Nat -> Text -> Nat -> {Exception} Text 
encodePairs = cases
    _, _, clen, code, dcount | dcount == clen -> 
        code |> padTrailing |> ensureSeparator
    lat, long, clen, code, dcount | dcount < clen ->       
        pv = placeValue dcount        
        (ncode, adjLatitude) = appendCode code lat pv
        (ncode2, adjLongitude) = appendCode ncode long pv
        digitCount = dcount + 2
        newCode = if (digitCount == separatorPosition) && (digitCount < clen) then 
            ncode2 ++ separator
        else    
            ncode2
            
        encodePairs adjLatitude adjLongitude clen newCode digitCount
    _, _, _, _, _ ->
        Exception.raise (Generic.failure "failure!" ())
```
The first change is that I have to use `cases` here to pattern match on the arguments, since Unison doesn't support the multiple function head syntax like Elixir.

The next source of trouble was the `Exception` ability. There are a number of functions that I needed to use that return data of type `Optional`. It took many hours of smashing my head against the desk and the _remarkably_ (possibly even saint-like) patient efforts of the folks in the Unison slack channel to get me past this. I couldn't just call `unwrap()` or put a `?` at the end of every function call like I can in Rust.

The `encodePairs` function calls two other important functions: `placeValue` and `appendCode`. The first obtains a precision value that can in turn be used to locate a character within the base-20 alphabet. The latter appends that code to the current string.

```haskell
placeValue : Nat -> {Exception} Float
placeValue digitCount = 
    Abort.toGenericException "placeValue failed!" "error" do
        index = floor (Nat.toFloat digitCount / 2.0) |>  Float.toNat |> Optional.toAbort               
        List.at index pairResolutions |> Optional.toAbort

appendCode : Text -> Float -> Float ->{Exception} (Text, Float)
appendCode code adjCoord placeValue =
  Abort.toGenericException "appendCode failed!" "error" do
    digitValue = floor (adjCoord / placeValue) |> Float.toNat |> Optional.toAbort
    newAdjCoord = adjCoord - (toFloat digitValue * placeValue)
    char = charAt digitValue codeAlphabet |> Optional.toAbort
    (code ++ Char.toText char, newAdjCoord)
```
Here `Abort.toGenericException` is an ability handler that will raise a generic exception whenever an `abort` is encountered in the code. It's tempting to think of this as a try/catch block, but it's better to try and think of it in Unison terms.

So let's see how this works:
```haskell
myProgram : '{IO, Exception} ()
myProgram = do
    printLine (encode 20.375 2.775 6)
    printLine (encode 20.3700625 2.7821875 10)
```
This should give us two successively smaller and more precise regions of space somewhere on the planet. The first call to `encode` wants 6 characters in the code and the second wants 10. If I've done this properly, the more precise code should have the same prefix as the less precise, and the number of digits that differ represents the difference in precision/size of the grid box.

```shell
.> run myProgram
7FG49Q00+
7FG49QCJ+2V
```
Both of these codes are within the region defined by `7FG49Q`, a small spot in southern Algeria. You can look up any plus code by putting it on the `plus.codes` URL, e.g. [https://plus.codes/7FG49Q00+](https://plus.codes/7FG49Q00+).

## Next Steps
My next steps are to write the _decode_ portion of the library. Decoding an open location code involves working backwards over the base-20 alphabet and producing a "code area", which is defined by the GPS coordinate of the southwest corner and latitude and longitude resolution.

I also need to write tests on my code, because I know it's super flaky. Right now if you ask for any code over 10 digits it breaks. It's going to be another long, arduous journey to get to the point where I'm ready to publish my PlusCode library on Unison share.

I will, of course, continue blogging about this journey as I force myself to write more Unison code by porting the PlusCode library.