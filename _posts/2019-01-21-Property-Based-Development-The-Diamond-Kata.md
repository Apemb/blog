# Property Based coding The diamond kata
## Let's print diamonds
The diamond kata is a well known exercise in TDD. Here's a description copied from http://cyber-dojo.org:

Given a letter print a diamond starting with 'A' with the supplied letter at the widest point.
For example: print-diamond 'E' prints

        A
       B B
      C   C
     D     D
    E       E
     D     D
      C   C
       B B
        A

For example: print-diamond 'C' prints

      A
     B B
    C   C
     B B
      A

For example: print-diamond 'A' prints

    A

We want to write such a program following the Test Driven Development way, but using Property Based tests instead of usual test cases. 

## Properties Todo List
Looking at the examples above, we can find interesting properties about the program results:

- The number of rows of the resulting pattern is obviously a function of the parameter: with E, the length in rows is 9, with C it's 5, with A it's 1. Thus the number of rows in the pattern is _2n-1_ where _n_ is the position of the letter in the alphabet.

- The maximum number of columns in the pattern is also _2n-1_.

- The upper left corner of the pattern of size _n x n_, is filled with a diagonal formed by the letters A,B,C etc. The letter A is in position _n-1_ (starting from 0), B in postion _n-2_, C, in position _n-3_, and so on.

- The pattern has horizontal symmetry, which means that flipping the diamond horizontally yields the initial pattern.

- The pattern also has vertical symmetry, which means that flipping the diamond vertically yields the initial pattern.

## QuickCheck
Using _QuickCheck_ is very simple. As an example, let's create a test harness and check different ways to obtain the index position of a letter in the alphabet. We use the `choose :: (a,a) -> Gen a` function, a generator that will allow for testing only chars from 'A' to 'Z' in our case.

```Haskell
-- Specs.hs -- 
import Test.QuickCheck
import Data.Char

letter = choose ('A','Z')

propLetterIndex = 
    forAll letter $ \maxLetter -> 
        let position = length ['A'..maxLetter]
            order    = ord maxLetter - ord (pred 'A')
        in order == position

main = do
    quickCheck propLetterIndex
```
Running our test program will show that this equality is valid for 100 random choices of a letter:
```
runhaskell Specs.hs ⏎
+++ OK, passed 100 tests.
```

## First property
Now we can write a test for the first property : a diamond to the letter with position _n_ in the alphabet should be _2n-1_ rows long:
```Haskell

position maxLetter = length ['A'..maxLetter]

propSizeInRows =
    forAll letter $ \maxLetter ->
        let n = position maxLetter
        in length (diamond maxLetter) == 2 * n - 1

main = do
    quickCheck propLetterIndex
    quickCheck propSizeInRows
```
And now we have to write a function `diamond` to make the test fail:
```Haskell
diamond :: Char -> [String]
diamond _ = []
```
```
*** Failed! Falsifiable (after 1 test):
'M'
```
It's easy to make it pass: just fill the result with _2n-1_ empty lines.
```Haskell
diamond :: Char -> [String]
diamond maxLetter = 
    let n = length ['A'..maxLetter]
     in replicate (2 * n - 1) ""
```
## Second property
The second property states that a diamond has a maximum of _2n-1_ cols.
```Haskell
propMaxSizeInCols =
    forAll letter $ \maxLetter ->
        let n = position maxLetter
        in maximum (map length (diamond maxLetter)) == 2 * n - 1
```
And to make it pass, we just need fill each row with _2n-1_ spaces.
```Haskell
diamond :: Char -> [String]
diamond maxLetter = 
    let n = length ['A'..maxLetter]
        t = 2 * n - 1
        spaces x = replicate x ' '
     in replicate t (spaces t)
```
## Third property
So far we have a function that yields a simple block of spaces, this block being of the right size to contain a diamond. This is not a lot, but it allowed to discover how to create lists with `replicate :: Int -> a -> [a]`. Let's now write a check for the third property: there should be a diagonal in the upper left corner of the diamond. This check is a bit more complicated, as it involves evaluting a predicate for all the elements of a list, and inspecting the diamond pattern at a given row and column.

```Haskell
propDiagonal = 
    forAll letter $ \maxLetter ->
        let d = diamond maxLetter
            n = position maxLetter
         in all (\l -> let pos = position l
                           row = pos - 1
                           col = n - pos
                        in d !! row !! col == l)
              ['A'..maxLetter]
```
Making this test pass involves creating a diagonal with letters A,B,C.. at columns _n_-1, _n_-2,_n_-3, where _n_ is the index of the last letter. This can be done with `zipWith :: (a -> b -> c) -> [a] -> [b] -> [c]`. Then this diagonal (which is only a quarter of the final result) is to be padded with empty lines so that our first and second properties still hold.
```Haskell
diamond :: Char -> [String]
diamond maxLetter = 
    let n = length ['A'..maxLetter]
        t = 2 * n - 1
        spaces x = replicate x ' '
        format c p = spaces p ++ [c] ++ spaces (t - p - 1)
        topHalf = zipWith format ['A'..maxLetter] [n-1,n-2..]
        bottomHalf = replicate (n-1) (spaces t)
     in topHalf ++ bottomHalf
```

When trying the `diamond` function with _ghci_ we get this (partial, yet promising) result:
```
> putStr (unlines (diamond 'E'))
    A
   B
  C
 D
E




>
```
## Fourth property
The fourth property states that flipping the diamond pattern horizontally should yield the same pattern. This check is easy to write.
```Haskell
propHorizontalSymmetry =
    forAll letter $ \maxLetter ->
        let d = diamond maxLetter
         in map reverse d == d
```
For this test to pass, the diagonal should be mirrored in the upper right corner of the resulting pattern.  To make this change, create the diagonal in a space limited to half the total length and then map a function that will _mirror_ every line, where `mirror s = s ++ drop 1 (reverse s)` 
```Haskell
diamond :: Char -> [String]
diamond maxLetter = 
    let n = length ['A'..maxLetter]
        t = 2 * n - 1
        spaces x = replicate x ' '
        format c p = spaces p ++ [c] ++ spaces (n - p - 1)
        upperLeft = zipWith format ['A'..maxLetter] [n-1,n-2..]
        mirror s = s ++ drop 1 (reverse s)
        topHalf = map mirror upperLeft
        bottomHalf = replicate (n-1) (spaces t)
     in topHalf ++ bottomHalf
```
And now a trial run gives the following result:
```
> putStr (unlines (diamond 'E'))
    A
   B B
  C   C
 D     D
E       E




>
```
## Fifth property
The fith property states that flipping the diamond pattern vertically should yield the same pattern. 
```Haskell
propVerticalSymmetry =
    forAll letter $ \maxLetter ->
        let d = diamond maxLetter
         in reverse d == d
```
This change is really easy to make, as we already have the `mirror` function to help us:
```Haskell
diamond :: Char -> [String]
diamond maxLetter =
   let n = length ['A'..maxLetter]
       spaces x = replicate x ' '
       format c p = spaces p ++ [c] ++ spaces (n - p - 1)
       upperLeft = zipWith format ['A'..maxLetter] [n-1,n-2..]
       mirror s = s ++ drop 1 (reverse s)
       topHalf = map mirror upperLeft
    in mirror topHalf 
```
And now trying our function gives the following result:
```Haskell
> putStr (unlines (diamond 'E'))
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
import Diamond
import System.Environment

main = do
    letter <- fmap (head . head) getArgs
    putStr $ unlines $ diamond letterHaskell
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


