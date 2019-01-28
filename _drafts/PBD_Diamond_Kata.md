---
layout: post
title:  "Property Based coding The diamond kata"
date:   2019-01-26 21:29:25 +0100
categories: kata haskell coding
toc: false
---

In this post I show an example of solving a kata the TDD way, while using a Property Based Testing tool. 

_Test Driven Development_ is a programming pattern based on the short feedback cycle :

- _Red_: write a test that fails.
- _Green_: write the simplest code that makes the test pass.
- _Refactor_: remove duplication.

_Property Based Testing_ is a testing pattern consisting in making statements about the function of a function based on the input, and checking that these statements are true for many different possible inputs. It is generally used to find problem in a function that is already designed. 

In the following exercise, I am using  _QuickCheck_ to describe properties of a function, while designing the function one property at a time. My interest in doing so is around the question: can we use a design approach based on _building_ incrementally some code with the help of a testing tool made for _criticizing_ existing code ?

## Let's print diamonds

The diamond kata is a well known exercise. Given a letter, print a diamond starting with 'A' with the supplied letter at the widest point.

Here are three examples: 
```
input      A       C               E
           
output     A       A               A    
                  B B             B B   
                 C   C           C   C  
                  B B           D     D 
                   A           E       E
                                D     D 
                                 C   C  
                                  B B   
                                   A    
```
We want to write a program that prints a diamond in this fashion, while following the Test Driven Development way, using Property Based tests instead of usual test cases. 

### Properties Todo List
Looking at the third example above (`diamond 'E'`), we can find interesting properties about the output:

- The top left corner of the pattern consist in a diagonal formed with the letters A,B,C,D,E, which is replicated (and possibly reversed) in the other corners of the pattern.
- There is vertical symmetry in the pattern, meaning that reversing the pattern will produce the same pattern.
- There is horizontal symmetry in the pattern, meaning that reversing all the lines will produce the same pattern.
- The central line of the pattern is not duplicated, meaning that the top "half" of the pattern is always one line longer than the bottom "half".
- The central column of the pattern is not duplicated, meaning that the left "half" of the pattern is 1 column longer than the right "half".

## Discovering QuickCheck
### Checking for a single boolean value
Creating a test harness using _QuickCheck_ is very simple. Import the library, and start using the `quickCheck :: Testable prop => prop -> IO ()` function. The simplest value of type `Testable prop => prop` is probably a single value of type `Bool` (because `Bool` is an instance of `Testable`). Let's try:
```
> import Test.QuickCheck ⏎
> quickCheck True ⏎
+++ OK, passed 1 tests.
> quickCheck $ 2+2 == 5 ⏎
*** Failed! Falsifiable (after 1 test):
```
### Checking for a property
We can use quickCheck on a function rather than a simple `Bool` value. This is because `quickCheck` feeds on anything that is an instance of `Testable`, and `instance (Arbitrary a, Show a, Testable prop) => Testable (a -> prop)`, thus a function such a `even :: Integral a => a -> Bool` can be made a `Testable` :
```
> quickCheck $ even ⏎
*** Failed! Falsifiable (after 2 tests and 1 shrink):
1
```
Of course, this test fails because quickCheck can find numbers that are not even. Here is a property that should always pass the test:
```
> quickCheck $ \x -> even x == odd (succ x) ⏎
+++ OK, passed 100 tests.
```
### Implicit (by default) generators
The `quickCheck` function, when given a function of type `a -> prop`, runs the tests on a random values of type `a` (This is why the constraint `Arbitrary a` is part of the type signature of `Testable (a -> prop)`). Here's an example with different types:
```
quickCheck $ \b -> b == False || b == True ⏎
+++ OK, passed 100 tests.
> quickCheck $ \x -> sin x > -1 &&  sin x < 1 ⏎
+++ OK, passed 100 tests.
> quickCheck $ \xs -> reverse (reverse xs) == xs
+++ OK, passed 100 tests.
```
### Explicit generators
QuickCheck comes with useful random generators. The most common is the function `arbitrary :: Arbitrary a => Gen a`. To observe the functionning of a generator, use `sample :: Show a => Gen a -> IO ()` which prints a sample of random values:
```
> sample (arbitrary :: Gen Int) ⏎
0
0
-3
-2
1
-2
-3
-2
16
-5
5
> sample (arbitrary :: Gen Char) ⏎
'u'
'p'
'\240034'
'\707530'
'\383523'
'b'
'\1041008'
'\SYN'
'7'
'H'
'n'
> sample (arbitrary :: Gen [Int]) ⏎
[]
[2]
[-4,-1]
[-6,2,5]
[5,5,-6,5]
[-6,-7,9,1,-2,5,6,5]
[-2,0,-4,-11,-5,-11]
[4,9,-2,7,-4,9,6,-3,6,-1]
[-4,-1,-15,8,12]
[14,5,8,-4,-9,-5,-4,-15,-10,-1,13,-5,6,-11,-15,-16,-2]
[-16,-11]
```
The `choose :: Random a => (a,a) -> Gen a` function allows for generating values within an interval:
```
> sample (choose (-10,10)) ⏎
-6
-3
6
-7
-2
3
-7
6
1
-6
-7
```
### Using generators in a property
The way we make use of these generators explicitly is by combining them with the `forAll :: (Show a, Testable prop) => Gen a -> (a -> prop) -> Property` function. Here's an example:
```
> quickCheck $ forAll arbitrary $ \b -> not (not b) == b ⏎
+++ OK, passed 100 tests.
```
We can create tests that check a property requiring two (or more) values: 
```
> quickCheck $ forAll arbitrary $ \x -> forAll arbitrary $ \y -> x + y == y + x ⏎
+++ OK, passed 100 tests.
```
As an example, let's suppose we want to check an obvious property: for any letter _l_ and _m_ ∈ {A,…,Z}, if _l_>_m_ the sequence from A to _l_ is longer than the sequence from A to _m_, and vice versa. Let's write this property into a _Specs.hs_ file:
```Haskell
-- Specs.hs -- 
import Test.QuickCheck

letter = choose ('A','Z')

propLetterSequence = 
    forAll letter $ \l ->
        forAll letter $ \m ->
            (l > m) == (length ['A'..l] > length ['A'..m])
main = do
    quickCheck propLetterSequence
```
Running our test program will show that this equality is valid for 100 random choices of letters:
```
runhaskell Specs.hs ⏎
+++ OK, passed 100 tests.
```
## First property of the `diamond` function
Do you remember the first property we noticed about the diamond function ?
- The letters A,B,C,D,E form a diagonal in the upper left corner of the pattern.

Here's how to describe this property: any pattern produced by a call to `diamond max` function will contain a diagonal sequence from A to _max_ going from the top center of the pattern to the bottom left. Here's an example with letters from A to E:
```
   j
  i \012345678
    0    A.... 
    1   B ....
    2  C  ....
    3 D   ....
    4E    ....
    5.........
    6.........
    7.........
    8.........
```
The property should state that for any given row and column _i_,_j_ ∈ {0,…,4}:
- if _j = 4-i_ : the pattern at  the pattern at _(i,j)_ is the letter with position _i_ in the sequence _ABCDE_.
- if _j ≠ 4-i_ : the pattern at _(i,j)_ is a space.

Here's the property:
```Haskell
propDiagonal =
    forAll letter $ \l ->
        let letters = ['A'..l]
            max = length letters - 1
            pattern = diamond l
            coord = choose (0,max)
         in forAll coord $ \i -> 
             forAll coord $ \j ->
                case j == max - i of
                    True  -> pattern !! i !! j == letters !! i
                    False -> pattern !! i !! j == ' '
```
### Making our first test compile and fail
Let's make sure that our test executes (and fail) by creating a `diamond` function which yields an empty list.

```Haskell
-- Diamond.hs
module Diamond where

diamond :: Char -> [String]
diamond _ = [[]]
```
Sure enough, our test now fails:
```
*** Failed! Exception: 'Prelude.!!: index too large' (after 1 test):
'D'
1
0
```
since any access with `!!` will result in an 'index too large' error.
### Making our first test pass
To make this test pass, we have to implement `diamond` in such a way that, given for instance the letter E, it produces a list of strings with the following characteristics:

```
    spaces | letter | spaces
        4  |    A   |   0
        3  |    B   |   1
        2  |    C   |   2
        1  |    D   |   3
        0  |    E   |   4
```
Thus we can create a helper function `format :: Char -> Integer -> String` which given a letter and a position, will create the corresponding line of the corner, provided it knows about the maximum position in the letter sequence (4 in our example).  Then we repetitively call this `format` function with the letters {A,B,C…} and positions {0,1,2…}.
```Haskell
diamond :: Char -> [String]
diamond l = 
    let letters = ['A'..l]
        max = length letters - 1
        spaces n = replicate n ' '
        format c i = spaces (max-i) ++ [c] ++ spaces i 
     in zipWith format letters [0..max]
```
## The second property : vertical symmetry
The second property of the diamond pattern is stated thusly:
- There is vertical symmetry in the pattern, meaning that reversing the pattern will produce the same pattern.
It is very easy to implement in Haskell:

```Haskell
propVerticalSymmetry = 
    forAll letter $ \l ->
        reverse (diamond l) == diamond l
```

And making it pass is easy too:

```Haskell
diamond :: Char -> [String]
diamond l = 
    let letters = ['A'..l]
        max = length letters - 1
        spaces n = replicate n ' '
        format c i = spaces (max-i) ++ [c] ++ spaces i 
        half = zipWith format letters [0..max]
     in half ++ reverse half
```

Of course this is not yet satisfactorily, as a quick manual test in _ghci_ will confirm:

```
> putStrLn $ unlines $ diamond 'C' ⏎
  A 
 B 
C  
C  
 B 
  A
```

But we will adjust this when we'll take care of the last properties.
## The third property : horizontal symmetry
Here's the third property
- There is horizontal symmetry in the pattern, meaning that reversing all the lines will produce the same pattern.
```Haskell
propHorizontalSymmetry =
    forAll letter $ \l ->
        map reverse (diamond l) == diamond l
```
Here again, make the test pass is easy:
```Haskell
diamond :: Char -> [String]
diamond l = 
    let letters = ['A'..l]
        max = length letters - 1
        spaces n = replicate n ' '
        format c i = spaces (max-i) ++ [c] ++ spaces i 
        diagonal = zipWith format letters [0..max]
        half = map (\s -> s ++ reverse s) diagonal
     in half ++ reverse half
```
Again, this does not produce the final pattern yet:
```
> putStrLn $ unlines $ diamond 'C' ⏎
  AA
 B  B
C    C
C    C
 B  B
  AA
```
### Refactoring
The way we reverse and concat being similar for vertical and horizontal symmetry, we can easily factor it into a helper `mirror` function:
```Haskell
diamond :: Char -> [String]
diamond l = 
    let letters = ['A'..l]
        max = length letters - 1
        spaces n = replicate n ' '
        format c i = spaces (max-i) ++ [c] ++ spaces i 
        diagonal = zipWith format letters [0..max]
        mirror s = s ++ (reverse s)
     in mirror (map mirror diagonal)
```
## The last property : no duplication on the central column (or line)
Since things are simple enough, we can aggregate the fourth and fifth properties into one:
- The central line and column of the pattern are not duplicated, meaning that the top half of the pattern is 1 line shorter than the bottom half, and the left half of the pattern is 1 column shorter than the right half.

An even better way very to state that this property is the following:
- Given that all the previous properties hold, the number of lines and the number of columns in the pattern should be odd, not even. Precisely, it should be equal to _2l-1_ where _l_ is the length of the sequence {A,B,C,…,_m_} and _m_ is the maximum letter of the pattern.

```Haskell
propNoDuplicationOfCenter = 
    forAll letter $ \l ->
        let d = diamond l
            m = length ['A'..l] 
            h = length d
            w = length (head d)
         in h == 2*m-1 && w == 2*m-1
```
To make this test pass without breaking the previous properties, we need to only change our mirror function so that it drops 1 element from the reversed copy. This is done with the `tail` function:
```Haskell
diamond :: Char -> [String]
diamond l = 
    let letters = ['A'..l]
        max = length letters - 1
        spaces n = replicate n ' '
        format c i = spaces (max-i) ++ [c] ++ spaces i 
        diagonal = zipWith format letters [0..max]
        mirror s = s ++ (tail (reverse s))
     in mirror (map mirror diagonal)
```
And now trying our function gives the following result:
```
> putStr (unlines (diamond 'E')) ⏎
    A
   B B
  C   C
 D     D
E       E
 D     D
  C   C
   B B
    A
>
```
## Printing Diamonds
All we need now is a program calling the function with a parameter taken on the command line:

```Haskell
-- PrintDiamond.hs
import Diamond
import System.Environment

main = fmap (head . head) getArgs >>= putStr . unlines . diamond
```

And we are done :-)

```
runHaskell PrintDiamond Z ⏎
                         A
                        B B
                       C   C
                      D     D
                     E       E
                    F         F
                   G           G
                  H             H
                 I               I
                J                 J
               K                   K
              L                     L
             M                       M
            N                         N
           O                           O
          P                             P
         Q                               Q
        R                                 R
       S                                   S
      T                                     T
     U                                       U
    V                                         V
   W                                           W
  X                                             X
 Y                                               Y
Z                                                 Z
 Y                                               Y
  X                                             X
   W                                           W
    V                                         V
     U                                       U
      T                                     T
       S                                   S
        R                                 R
         Q                               Q
          P                             P
           O                           O
            N                         N
             M                       M
              L                     L
               K                   K
                J                 J
                 I               I
                  H             H
                   G           G
                    F         F
                     E       E
                      D     D
                       C   C
                        B B
                         A
```


