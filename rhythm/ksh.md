# ksh

Category: Rhythm

Points: 147

Solves: 65

Difficulty: 1/5

>This map is pretty cool... but I hear the latest version is better.

## Solution

We are given some `harumachialigned.ksh` file with this at the top of the file:

```
title=Harumachi Clover
artist=Hanasaka Yui(CV: M.A.O)
effect=Quintec
jacket=TwoRooms.png
illustrator=
difficulty=challenge
level=10
t=142
m=harumachialigned.wav
mvol=75
o=0
bg=desert
layer=arrow
po=0
plength=30000
pfiltergain=50
filtertype=peak
chokkakuautovol=0
chokkakuvol=50
ver=171
```

After a bit of ~~chatgpt~~ research, we come to know that ksh is the chart file format for the game `k-shoot mania`

After this we then find a website for sharing k-shoot mania charts https://ksm.dev/.

We then search for `Harumachi Clover` and get:

![found chart](/images/KSM_chart.png)


Now if we look into the details for this upload we get the first part of the flag:

![part 1](/images/KSM_part1.png)

Then if we downlaod the chart and open up the KSH file we get the 2nd part:

![part 2](/images/KSM_part2.png)

Together we get the flag:

`osu{f4iry_d4nc1ng_in_l4k3_eeeeee}`