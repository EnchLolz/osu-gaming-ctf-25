# beyond-wood

Category: Crypto

Points: 106

Solves: 392

Difficulty: 1/5

>spinning white floating glowing man in forest

## Solution

We are given a script to encode the flag:

```py
from PIL import Image
import random

FLAG = Image.open("flag.png")
width, height = FLAG.size

key = [random.randrange(0, 256) for _ in range(width+height+3)]

out = FLAG.copy()
for i in range(width):
    for j in range(height):
        pixel = FLAG.getpixel((i, j))
        pixel = tuple(x ^ k for x, k in zip(pixel, key))
        newi, newj = (2134266 + i * 727) % width, (4501511 + j * 727) % height 
        out.putpixel((newi, newj), pixel)

out.save("output.png")
```

and the output image:

![output](/images/beyondwoodoutput.png)

Notice that when we `xor` the pixel with a key we will always be xoring with the same 3 random bytes:

```py
pixel = tuple(x ^ k for x, k in zip(pixel, key))
```

This means, that we don't actually need to care about the key, as we know the colors will be randomized, but the same input colors will map to the same output colors. Thus we just need to reverse the shuffling of the pixels. Since we are given the magic numbers for the suffle. We can just precompute the final position and undo them.

```py
from PIL import Image

FLAG = Image.open("output.png")
width, height = FLAG.size

# Precompute end pixel destination
mp = dict()
for i in range(width):
    for j in range(height):
        newi, newj = (2134266 + i * 727) % width, (4501511 + j * 727) % height 
        mp[(newi, newj)] = (i, j)

# Reverse the pixel shuffling
out = FLAG.copy()
for i in range(width):
    for j in range(height):
        pixel = FLAG.getpixel((i, j))
        out.putpixel(mp[(i, j)], pixel)

out.save("flag.png")
```

Running the script, we recover:

![flag](/images/beyondwoodflag.png)

We then just transcribe the flag:

`osu{h1_05u_d351gn_t34m}`