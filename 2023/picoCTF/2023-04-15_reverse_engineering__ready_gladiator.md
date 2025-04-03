# Reverse Engineering / Ready Gladiator

This is a write-up of my solution to these challenges from the [2023 picoCTF](/posts/2023_picoctf):

- Reverse Engineering / [Ready Gladiator 0](https://play.picoctf.org/practice/challenge/368) (100 Points)
- Reverse Engineering / [Ready Gladiator 1](https://play.picoctf.org/practice/challenge/369) (200 Points)
- Reverse Engineering / [Ready Gladiator 2](https://play.picoctf.org/practice/challenge/370) (400 Points)

## CoreWars
Going into this challenge, I knew nothing about [CoreWars](https://en.wikipedia.org/wiki/Core_War).
It is a game from the 1980s, where each player prepares a "warrior"-program in an assembly-like language called `Redcode`.
For a game, each player's program is loaded to a random address in a cyclic memory space (*the core*).
The execution of the programs is alternated between the players after each performed instruction (i.e. each line of code).
The goal is to be the last program that is still executing valid instructions (e.g. "kill" other warriors by overwriting their code).

For further reading into this, `corewars.org` has a [list of tutorials](http://corewars.org/information.html).
I found [The beginners' guide to Redcode](https://vyznev.net/corewar/guide.html) by Ilmari Karonen to be particularly useful during this challenge.
For experimenting offline with a simple visualization,
Ben Lynn has a [simple JavaScript implementation](https://crypto.stanford.edu/~blynn/play/redcode.html) that runs in the browser
(this does not implement the full command set available in the picoCTF challenge though).

For the particular servers used in these challenges, I could determine some base rules using `assert`-statements:
```
;assert CORESIZE == 8000
;assert MAXCYCLES == 80000
;assert MAXPROCESSES == 8000
;assert MAXLENGTH == 100
;assert PSPACESIZE == 500
```

## Ready Gladiator 0 (100)
> Can you make a CoreWars warrior that always loses, no ties?
> Your opponent is the Imp. The source is available here. If you wanted to pit the Imp against himself, you could download the Imp and connect to the CoreWars server like this: `nc saturn.picoctf.net [port] < imp.red`

Downloading the `imp.red` reveals he has only one instruction that is copying itself in the next cell:
```
;redcode
;name Imp Ex
;assert 1
mov 0, 1
end
```

Given `MAXCYCLES == 80000` and `CORESIZE == 8000`, it will loop through the entire core 10 times during a game, overwriting every single memory cell.

The task is to loose all of 100 games, so as the most trivial warrior, I just removed the `mov` line and save it as `looser.red`:
```
$ nc saturn.picoctf.net [port] < looser.red
[...]
Warning:
        No instructions
Number of warnings: 1

Rounds: 100
Warrior 1 wins: 0
Warrior 2 wins: 100
Ties: 0
You did it!
picoCTF{h3r0_t0_z3r0_4m1r1gh7_XXXXXXXX}
```


## Ready Gladiator 1 (200)
> Can you make a CoreWars warrior that wins?
> Your opponent is the Imp. The source is available here. If you wanted to pit the Imp against himself, you could download the Imp and connect to the CoreWars server like this: `nc saturn.picoctf.net [port] < imp.red` 
> To get the flag, you must beat the Imp at least once out of the many rounds.

The opponent is the same `imp.red` as before, but the task is now to win at least once out of 100 rounds.

This is where I started to actually read through some of the references I listed above.
For an easy win, I just copy the `dwarf.red` from [Ilmari Karonen's tutorial](https://vyznev.net/corewar/guide.html#start_dwarf):
```
;redcode
;name dwarf
;assert 1
ADD #4, 3
MOV 2, @2
JMP -2
DAT #0, #0
end
```

Since this is a more sophisticated warrior, I expected it to win at least once against the imp.
It actually won 22 times (and never lost):
```
$ nc saturn.picoctf.net [port] < dwarf.red 
[...]
Rounds: 100
Warrior 1 wins: 22
Warrior 2 wins: 0
Ties: 78
You did it!
picoCTF{1mp_1n_7h3_cr055h41r5_XXXXXXXX}
```

## Ready Gladiator 2 (400)
> Can you make a CoreWars warrior that wins every single round?
> Your opponent is the Imp. The source is available here. If you wanted to pit the Imp against himself, you could download the Imp and connect to the CoreWars server like this: `nc saturn.picoctf.net [port] < imp.red`
> To get the flag, you must beat the Imp all 100 rounds.

The opponent is still the same `imp.red` as before, but the task is now to win every single round (out of 100).

Here the dwarf was not enough anymore and I started writing my own warriors.

Since I know that the enemy is always a single imp, I can base my strategy on that.
To stop the imp, I want to overwrite its current position with a `DAT` instruction.

### The Mole
I skimmed the tutorial section about [the process queue](https://vyznev.net/corewar/guide.html#start_queue)
and thought I could use the `SPL` instruction to start an endless loop that overwrites the core
*backwards*, thus meeting the imp, before it reaches me:
```redcode
SPL 0
ADD #-1, 2
MOV 1, @1
DAT #0, #-3
```

This wins 21:0 (with 79 ties), so it has a similar performance to the dwarf.

As a second attempt, I created the `multi_mole.red`, which spawns multiple copies of the mole at different offsets.
I thought with multiple processes, I would have a better chance of catching the imp.
```
SPL mole1
[...]
SPL mole7
mole0:  SPL 0
        MOV 2, @2
        SUB #1, 1
        DAT #0, -40
mole1:  SPL 0
        MOV 2, @2
        SUB #1, 1
        DAT #0, -1040
[...]
```

However, this only won 8:0 (with 92 ties), less than the normal mole.
After carefully reading through the tutorial section again, it was clear why:
Each player has its own process queue.
So each time that I split my process with `SPL`, my processes alternate taking turns.
The imp however can execute every turn because it is only one process.
Effectively, I was slowing down my processes the more I used `SPL`.

### The Blocker
At this point, I confirmed the meta-parameters of the sever.
As stated above, `MAXCYCLES == 80000` and `CORESIZE == 8000` mean that the imp should circle the entire core 10 times during each game.
Might it be enough to just block a single field constantly?

The simplest way to do this is the `blocker.red`:
```
MOV 2, -1
JMP -1
DAT #0, #0
```

It just copies the `DAT` in front of itself over and over again.
When the imp reaches that memory cell, it is overwritten and looses.
This warrior won 49:0 (51 tiew).

The problem is, that it takes two instructions to overwrite the field whereas the the imp only takes one to move away again.
Thus the blocker wins depending on the relative start position of the imp.
But I can actually detect, if the imp will loose or not and make the jump backwards conditional
If now the imp survived my first blocker, I jump forward to a second one which catches the imp with 100% certainty:
```
MOV 3, -1
JMZ -1, @-2
JMP 2
DAT 0, 0
MOV 2, -1
JMP -1
DAT 0, 0
```
This wins 100:0, yielding the flag `picoCTF{d3m0n_3xpung3r_XXXXXXXX}`.
