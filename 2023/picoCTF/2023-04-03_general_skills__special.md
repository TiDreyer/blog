# General Skills / Special(er)

This is a write-up of my solution to these challenges from the [2023 picoCTF](/posts/2023_picoctf):

- General Skills / [Special](https://play.picoctf.org/practice/challenge/377) (300 Points)
- General Skills / [Specialer](https://play.picoctf.org/practice/challenge/378) (400 Points)

## The Problem
In both cases an `ssh`-login to a system is provided and the task is "simply" to read the flag from a simple text file on the system.
The difficulty comes from the shells used;
instead of landing in a regular `bash` after login, each system has its own twist...

## Special (300)
> Don't power users get tired of making spelling mistakes in the shell? Not anymore!
> Enter Special, the Spell Checked Interface for Affecting Linux.
> Now, every word is properly spelled and capitalized... automatically and behind-the-scenes!
> Be the first to test Special in beta, and feel free to tell us all about how Special streamlines every development process that you face.
> When your co-workers see your amazing shell interface, just tell them: That's Special (TM)

The 300 point version of this task uses the shell `special` ("Spell Checked Interface for Affecting Linux").
This does exactly what the name suggests; each command is run through a spell checker and auto-corrected before it is executed:
```
Special$ ls
Is 
sh: 1: Is: not found

Special$ whoami
Whom 
sh: 1: Whom: not found

Special$ bash
Why go back to an inferior shell?
```

So we need to find a way to circumvent this "assistance".

Just using commands that are already english words does not work, since they get capitalized before execution:

```
Special$ cat
Cat 
sh: 1: Cat: not found
```

But if it is a correct spell checker, only the first word in a sentence should start with a capital letter.
This can be used by [chaining commands](https://www.gnu.org/software/bash/manual/bash.html#Lists), for example with the `&` operator.
While the first command is spell checked and fails, the second command can stay as it was and is executed correctly:

```
Special$ echo & kill
Echo & kill 
sh: 1: kill: Usage: kill [-s sigspec | -signum | -sigspec] [pid | job]... or
kill -l [exitstatus]
sh: 1: Echo: not found

Special$ echo & cat flag.txt
Echo & cat flag.txt 
sh: 1: Echo: not found
cat: flag.txt: No such file or directory
```

(Notice that `&` will run the commands asynchronously, so the output is not always in the same order as the commands)

This allows us to run most commands now; let's look around the file system with `find`:

```
Special$ a & find
A & find 
sh: 1: A: not found
.
./blargh
./blargh/flag.txt
./.cache
./.cache/motd.legal-displayed
```

Ok, all that is left to do is to print the content of `blargh/flag.txt`.
I tried to open an editor, but none of `less`, `vim`, `nano` and `pico` were installed.
`more` is present however:

```
Special$ a & more blargh/flag.txt
A & more blargh/flag.txt 
sh: 1: A: not found
picoCTF{5p311ch3ck_15_7h3_w0r57_XXXXXXXX}
```

## Specialer (400)
> Reception of Special has been cool to say the least.
> That's why we made an exclusive version of Special, called Secure Comprehensive Interface for Affecting Linux Empirically Rad, or just 'Specialer'.
> With Specialer, we really tried to remove the distractions from using a shell.
> Yes, we took out spell checker because of everybody's complaining.
> But we think you will be excited about our new, reduced feature set for keeping you focused on what needs it the most.
> Please start an instance to test your very own copy of Specialer.

The set-up for the 400 point version is almost the same, except this time the shell used is called `specialer` ("Secure Comprehensive Interface for Affecting Linux Empirically Rad").

There is no spell checking this time, but the command set installed on the system is very limited.
Tab completion without prior input yields the complete list:

```
Specialer$ 
!          bind       compopt    elif       fc         if         printf     shift      true       while
./         break      continue   else       fg         in         pushd      shopt      type       {
:          builtin    coproc     enable     fi         jobs       pwd        source     typeset    }
[          caller     declare    esac       for        kill       read       suspend    ulimit     
[[         case       dirs       eval       function   let        readarray  test       umask      
]]         cd         disown     exec       getopts    local      readonly   then       unalias    
alias      command    do         exit       hash       logout     return     time       unset      
bash       compgen    done       export     help       mapfile    select     times      until      
bg         complete   echo       false      history    popd       set        trap       wait
```

Tab completion can reveal file names too, but there is no way to output them directly with these commands.

Since `echo` is available, I thought of piping the files as `stdin` to echo.
I found an [answer on StackOverflow](https://stackoverflow.com/a/7045517) that shows how to do this. 
I can define my own bash function that mimics `cat` with that method:

```bash
function cat {
  while read line
  do
    echo "$line"
  done < "${1:-/dev/stdin}"
} 
```

Or in one line for easier copying:

````bash
Specialer$ function cat { while read line; do echo "$line"; done < "${1:-/dev/stdin}"; } 
Specialer$ echo "asdf" > testfile
Specialer$ cat testfile 
asdf
````

With tab completion I could find six files, but they all seemed to be empty...

```bash
Specialer$ cat abra/cadabra.txt
Specialer$ cat abra/cadaniel.txt
Specialer$ cat ala/mode.txt
Specialer$ cat ala/kazam.txt
Specialer$ cat sim/city.txt
Specialer$ cat sim/salabim.txt
```

I thought this might be an issue with file permissions, so I checked for that.
I am not allowed to write on the files, but they are regular readable non-zero files:

```bash
Specialer$ echo "asdf" >> ala/kazam.txt 
-bash: ala/kazam.txt: Permission denied
Specialer$ function is_file { test -f $1 -a -r $1 -a -s $1; echo $?; }
Specialer$ is_file abra/cadaniel.txt 
0
```

After some further testing, I found that the `cat` function above did not work as expected.
I am not sure what exactly the problem was, but this version worked better:

```
Specialer$ function cat { readarray FILECONTENT < $1; echo "${FILECONTENT[@]}"; }
Specialer$ echo "1" > testfile
Specialer$ echo "2" >> testfile
Specialer$ echo "3" >> testfile
Specialer$ cat testfile
1
 2
 3
```

Applying this to the six files finally yields the flag:

```
Specialer$ cat abra/cadabra.txt 
Nothing up my sleeve!
Specialer$ cat abra/cadaniel.txt 
Yes, I did it! I really did it! I'm a true wizard!
Specialer$ cat ala/kazam.txt 
return 0 picoCTF{y0u_d0n7_4ppr3c1473_wh47_w3r3_d01ng_h3r3_XXXXXXXX}
Specialer$ cat ala/mode.txt 
Yummy! Ice cream!
Specialer$ cat sim/city.txt 
05ed181c-4aa0-4d4a-8505-2fe6ca9097d3
Specialer$ cat sim/salabim.txt 
#He was so kind, such a gentleman tied to the oceanside#
```
