# Reverse Engineering / strings & grep

This is a write-up of my solution to these challenges from the [2023 picoCTF](index.md):

- Reverse Engineering / [Reverse](https://play.picoctf.org/practice/challenge/372) (100 Points)
- Reverse Engineering / [Safe Opener 2](https://play.picoctf.org/practice/challenge/375) (100 Points)
- Reverse Engineering / [timer](https://play.picoctf.org/practice/challenge/381) (100 Points)
- Reverse Engineering / [No way out](https://play.picoctf.org/practice/challenge/361) (200 Points)

**Note:** These are most likely not the intended solutions to these challenges.
See the following section for context.

## Context: strings and grep
`strings` and `grep` are two tools that come pre-installed on almost all linux systems.
`strings` will simply go through a binary file and list out all sequences of printable characters
(with a given minimal length).
`grep` is one of the most useful CLI-tools in general.
It allows to filter text (files, `stdin` or entire directories) using [regular expressions](https://en.wikipedia.org/wiki/Regular_expression).

In combination, these tools make it very easy to find the flag, if it written in plain text somewhere.
For example, say the executable `challenge` is based on some code like this:
```cpp
#include <iostream>

int main() {
    int guess;
    std::cout << "Guess my lucky number: ";
    std::cin >> guess;
    if (guess == 7) {
        std::cout << "Correct! Here is the flag: flag{XXX}" << std::endl;
    } else {
        std::cout << "Wrong!" << std::endl;
    }
    return 0;
}
```

Then the binary file `challenge` will at some position contain the bytes for the string `Correct! Here is the flag: flag{XXX}`.
Strings makes it easy to get all the printable strings in the file and grep allows to filter for the known pattern `flag{.*}`:
```
$ strings challenge | grep "flag{.*}"
Here is the flag: flag{XXX}
```

In this case, there is no need to guess the number or even execute the binary at all to get the flag

This is the reason many challenges are run on servers accessible only via `netcat` or `ssh`;
this makes it possible that the flag is stored in a separate file that the player can not access directly.
Another option would be to "hide" the flag in the source code,
e.g. storing an encoded/encrypted version as the string and only decoding/decrypting it when the challenge is solved.

With this being said, let's grep some easy points!

## Reverse (100)
> Try reversing this file? Can ya? I forgot the password to this file. Please find it for me?

We are given the binary `ret`:
```
$ strings ret | grep "picoCTF{.*}"
Password correct, please see flag: picoCTF{3lf_r3v3r5ing_succe55ful_XXXXXXXX}
```

## Safe Opener 2 (100)
> What can you do with this file?
> I forgot the key to my safe but this file is supposed to help me with retrieving the lost key.
> Can you help me unlock my safe?

We are given the compiled Java class data file `SafeOpener.class`:
```
$ strings SafeOpener.class | grep "picoCTF{.*}"
,picoCTF{SAf3_0p3n3rr_y0u_solv3d_it_XXXXXXXX}
```

## timer (100)
> You will find the flag after analysing this apk
> Download here.

We are given an Android app in the form of the file `timer.apk`.
This needs to be unzipped before further analysis:

```
$ unzip timer.apk
[...]
$ grep -rn timer -e "picoCTF{.*}"
grep: timer/classes3.dex: binary file matches
$ strings timer/classes3.dex -n 7 | grep "picoCTF{.*}"
*picoCTF{t1m3r_r3v3rs3d_succ355fully_XXXXX}
```

## No way out (200)
> Put this flag in standard picoCTF format before submitting.
> If the flag was h1_1m_7h3_f14g submit picoCTF{h1_1m_7h3_f14g} to the platform.
> Windows game, Mac game

We are given `win.zip` or `mac.zip`, containing a full Unity game each.
This one is a bit trickier, because it doesn't directly contain `picoCTF` as a string.
The files `win/pico.exe` and `win/pico_Data/level0` look like good candidates,
since most other files appear to be standard `.dll`s.

I then used the fact, that all flags contained underscores so far:
```
$ unzip win.zip
$ strings win/pico_Data/level0 | grep "_.*_"
WELCOME_TO_XXXXXXX
```

`picoCTF{WELCOME_TO_XXXXXXX}` solves the challenge.
