---
title: Warm Up Your MacBook
layout: post
date: 2019-11-18T15:36:13.718Z
published: false
imageurl: /images/frozen-mac-610x371.jpg
description: >-
  Who else hates putting their hands on an icy metal laptop after it's been
  riding outside in your cold backpack? Here's a one-line solution.
---
You've been there - after putting your backpack in a frigid car, walking against the Wisconsin wind, or biking across the frozen lake, you arrive at work. You rest your palms on the keyboard to begin typing your password, and recoil in pain from the sudden cold of the metal sucking the heat from your skin.

How do you quickly warm up a laptop? Make it do a lot of work.

Here's a one-line, built-in command that will peg your CPU to 100%:

```
yes > /dev/null
```
Run that from a Terminal, and don't forget about it heh. What the command does is repeatedly send the word `yes` over and over to the null device, using 100% CPU.

If you want to stress your Mac more quickly and get your CPU hotter, try the `stress` utility:

```
brew install stress
stress -c 6 -m 2 -t 300
```
This will start 6 threads that each peg your CPU to 100% and 2 thread that do memory malloc/free. It has a 300-second timeout (5 mins) in case you walk away from your computer so it doesn't overheat.

Bonus points, you can add an alias to your `~/.bash-profile` to automate these options:
```
alias warm='stress -c 6 -m 2 -t 300'
```

```
fred: bash$ warm
stress: info: [65121] dispatching hogs: 6 cpu, 0 io, 2 vm, 0 hdd
```

Happy winter!


