#TI-BASIC Bejeweled

TI-BASIC is the unofficial name for the programming language included on the stock operating systems of the TI-83 and TI-84 Plus series of calculators. This includes a large set of calculators, the most popular these days being the 96x64 monochrome TI-84+SE, the 320x240 16 bit color TI-84+ Color SE (referred to as the "CSE"), and the newly released TI-84+CE which features a LCD which is significantly faster to access than the CSE and a processor upgrade from a z80 to an ez80.

The 84+ and 84+CSE use z80 processors clocked at roughly 15mhz. This is an old CPU architecture, with no modern features like caching, out of order execution, floating-point units, or even pipelining. As a result of the CPU limitations, the slow interpreter, and the fact that all numbers in TI-BASIC are 9 byte floats, programmers may find that even the simplest of games they make run too slowly to be playable. Many turn to projects like the [Axe Parser](https://www.omnimaga.org/the-axe-parser-project/) to provide a faster compiled language. Some bite the bullet and learn how to program games in assembly. However, those that continue with TI-BASIC learn the ins and outs of the language, and how to make it work for them.

###Quirks of TI-BASIC
TI-BASIC has a few things that should be known before reading the following code. All numeric variables are one letter. Valid variables include A-Z and θ (theta). There are also one dimensional List variables such as L₁ and two dimensional Matrix variables such as \[A\]. List and matrix indices are in the range of \[1,Length\]. Matrix values are accessesed with \[Matrix\](Row,Column).


##Creating a Match-3 Game
One of the first problems anyone who creates a match-3 game (like Bejeweled) runs across is detecting matches. Matches need to be detected wherever they may lie on the board, so that chain reactions can occur from tiles falling in. One of the simplest solutions works just fine for anything written to run on modern hardware.
```
for every row
    counter = 0
    last_seen = 0      //Last type of tile scanned
    for every column
        if last_seen == tile_at(column, row)
            counter++
        else
            counter = 1
        if counter >= 3
            set_tile(column - 2, row, 0)
            set_tile(column - 1, row, 0)
            set_tile(column    , row, 0)
        last_seen = tile_at(column, row)
```
This works for removing tiles for matches greater than or equal to 3 in length. It removes all horizontal matches; row and column can be switched to find vertical matches. The problem is that this code is somewhat complex, and as a result the TI-BASIC version of it runs fairly slowly.
```
//R = row
//C = column
//I = counter
//L = last_seen
//[A] = matrix storing tiles
//Clears vertical matches
For(C,0,7)
  0→L
  0→I
  For(R,0,7)
    [A](R,C)→X
    If L=X:Then
      I+1→I
      If I≥3:Then
        For(A,R-2,R)
          0→[A](A,C)
        End
      End
    Else
      1→I
      X→L
    End
  End
End
```
<img class="img-right" src="/img/ti-basic-bejeweled/demo-1.gif" width="320" height="240"></img>

Experienced TI-BASIC developers will notice that this version is written for clarity over speed or size. Even with all the micro-optimizations that can be added to the code, it runs too slowly to be useful, as shown in the included recording. From the time that the cursor disappears to the time it reappears, the player can't do anything but stare at the screen. The displayed version also has the added optimization that it won't check columns or rows which could not have changed since the previous check, since no changes implies no matches, but it's still not enough to be playable.

<!--<span style="clear: both"></span>-->

So what can be done to make this program run better? Up until this point I had thought of the problem solely as counting the number of matching tiles, but it can be thought of another way, one which the calculator has built in functions to deal with. They key lies with the DeltaList() function.

DeltaList(), or ΔList() as it's displayed on the calculator, takes a list and returns a new list with the difference between the elements of the source list. The result will be one element shorter than the source list, since there is no element preceding the first to subtract from. Therefore `DeltaList({1,3,3,3,2,4})` results in `{-2,0,0,-1,2}`. If the difference between two elements is zero, they are equivelant! Applying DeltaList twice to a list should then place zeros only at the end of matches of three, indicating exactly where matches exist with no counting being done in TI-BASIC code.
```
DeltaList({1,3,3,3,2,4}) = {-2,0,0,-1,2}
DeltaList({-2,0,0,-1,2}) = {-2,0,-1,3}   //0 = Match
```

##Dealing With False Positives
Unfortunately, there's a flaw with simply using DeltaList(DeltaList()) on our input. What if we provide the list `{6,1,2,3,5}`?
```
DeltaList({6,1,2,3,5}) = {-5,1,1,2}
DeltaList({-5,1,1,2})  = {6,0,1}    //0 = False Positive
```
We get a false positive. That simply won't work for the game, but the solution isn't difficult: Raise the source list to a power. Raising to the power of two results in two possible sequences that give false positives, `{7,5,1}` and `{1,5,7}`, but raising to the power of three results in none. Even with this extra calculation, the speed still beats out calculating matches in TI-BASIC. The new matching code (minus optimizations) looks something like this.
```
For(C,1,8)
  //Store column C of matrix [A] into list L₁
  Matr►list([A],C,L₁)
  //not() normalizes the list so that matches are 1, non-matches are 0
  not(DeltaList(DeltaList(L₁^3)))→L₁
  For(Y,1,6max(L₁))
    If L₁(Y):Then
      //Y is equal to the start of the match because the length
      //of L₁ is two less than the size of the board
      For(A,Y,Y+2)
        0→[A](A,C)
      End
    End
  End
End
```
<img class="img-left" src="/img/ti-basic-bejeweled/demo-2.gif" width="320" height="240"></img>

Not only is the code faster, it's also much simpler. In TI-BASIC, the less code running the better, and here all the code does in the inner loop is a simple `If` and a mass set to zero. The result is a much more playable match-3 game. The recording shown is a more complete version with a scoring and level system, but it displays the speed advantages when making matches quite well. Not everyone is lucky enough to be able to reframe their problem in a way that fits the language so well, but these sort of tricks are what really allow people to continue to create impressive games in this incredibly restrictive language.

<span style="clear: both"></span>

##Hidden Complexity
To the untrained eye, it may seem that I've removed the loop necessary to count the number of matching elements in a row or column. Not only have I _not_ done that, I've increased the amount of work that needs to be done. DeltaList(DeltaList()) has to take the difference of the input list, and then further take the difference of the differences. Additionally, the input list has to be cubed first! In reality, I've simply hidden the loop by moving the work out of my code and into the interpreter's code. DeltaList() is implemented in assembly, while my naïve solution is not. All the operations being performed on a particular row or column are still O(n), but the assembly is able to work so much more efficiently that it results in a noticeable speed up.

##Game-Over Detection
Every good match-3 game needs to be able to tell the player when the game is over. Otherwise, they may spend a long time staring at an unsolvable board, before finally getting frustrated and quitting. Unfortunately, checking if a game is still playable is a rather long process. My initial plan was to perform a bunch of pattern matching using more DeltaList(). The code builds a list by reading different patterns of 3 tiles from the board that, if they all contain the same tile, indicate a potential future match. This strategy works, but it's extremely slow. To solve this problem, the matching can be staggered. On every repetition of the main game loop, the game processes key input and then performs one step in the game-over detection. There are 48 total steps. If the 48th step is reached, and returns no solutions, the game is over.

##Saving the Game
Saving the game is a fairly simple process. A few variables are saved to a TI-BASIC List, and loaded at next start-up. This presents two problems: How can the program handle an archived (unaccessable) list, and how can it verify that the data is valid? To handle archived lists, it tries to access the list very early on in the program, ensuring that if the list is archived the program will fail quickly, without wasting time. To verify data correctness, an error detection code is calculated from the other 4 values in the list. If the error detection code stored in the list does not match the code calculated from the saved values, the game resets to its default state. As an added bonus, this means that the game can simply create the list if it doesn't exist and read in the zeroed out list to load the game. It will fail the error detection, forcing the game to the proper initial state.

##A Note On Graphics
Anyone who has even lightly touched on using TI-BASIC knows that the level of graphics displayed in the two recordings above simply isn't possible in pure TI-BASIC at that speed. To work around this problem, people have developed shells for launching these TI-BASIC programs that provide routines the programs can call for graphics, advanced key input, and other various utility operations. Unfortunately they can't fully solve the speed problems of the language, but they're certainly a leg up on pure TI-BASIC. The library I'm using for the demos above is called xLIBC, written by tr1p1ea and included in the [DCSE](http://dcs.cemetech.net/index.php/Main_Page) library created by [Christopher Mitchell](http://z80.me).

The final game is downloadable [Here](http://www.cemetech.net/programs/index.php?mode=file&id=1327).
