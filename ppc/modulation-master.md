# modulation-master

Category: PPC

Points: 202

Solves: 30

Difficulty: ?/5

>Have you heard of digital modulation?


## Solution

After visting the website and starting the challenge, it looks like we are given some sort of modulated wave and we need to decode the character it represents within 2 seconds:

![the challenge](/images/PPCexplore.png)

Ok, so the first step is how do we get this image. Well, we can see that this challenge will require us to communicate with there server through websockets:

![websocket](/images/PPCwebsocket.png)


So we can first implement a way to communicate with the server:


```py
from PIL import Image
from io import BytesIO
import websockets
import asyncio

async def main():
    uri = "wss://CHALLENGE-URL/echo"
    async with websockets.connect(
        uri,
    ) as ws:
        # Start the game
        await ws.send("start")
        reply = await ws.recv()
        print("Received:", reply)


        # While we haven't won yet
        while "100/100" not in reply:
            # Recieve Image
            img_response = await ws.recv()
            img = Image.open(BytesIO(img_response))
            
            # TODO implement decode
            char = decode(img)
            print("Identified character:", char)
            
            # Send character back to server
            await ws.send(char)
            reply = await ws.recv()
            print("Received:", reply)

        # Keep recieving for the flag
        while True:
            reply = await ws.recv()
            print("Received:", reply)


asyncio.run(main())
```

Now, how are we going to decode these images, especially since there are different modulation types. EG:

![O](/images/O_1.png)
![S](/images/S_0.png)
![U](/images/U_1.png)

We'll, after a bit of researching and a bit too much of collecting images from there servers, we can see that there are only 3 modulation types that they have implemented as shown above. These digital modulation types are, Frequecy Shift Keying (FSK), Aplitude Shift Keying (ASK), and Phase Shift Keying (PSK). As we can see by the above examples. The first one has places a higher frequency and places of lower frequency. The second one has places with max amplitude and 0 amplitude. The third one while is may seem strange, the inverses at the tickmarks are just a 180 degree phase shift in the sin wave.

Next, we can realize that each bit is encoded by the region between every 1000 units on the x-axis. ie, 0-1000 encodes the first bit, 1000-2000 encodes the second bit, and so on. While, we can no visually decode this, giving this to a computer is still rather hard.

Our goal then is to first, find a way to categorize the images into FSK, ASK, or PSK, then find a way to decode the image in each of the cases. Let's start out with ASK. Visually, it's the only one where the signal line is flat at 0 at times. In fact, since our answer must be an ascii character, we know that the first bit must always be 0. Thus we can check if an image is in ASK if we can detect the flat line between 0 to 1000.

Then between, FSK and PSK, we can imagine picking some height, then shooting a line horizontally from the height to determine the frequency. For a FSK encoded signal, the amount of intersections between this line and the signal will be more in the lower frequency regions while there will be more intersection in the higher frequency regions. (Compare 0-1000 to 1000-2000 in the first image). On the other hand, PSK has the same frequency, so as long as we don't pick some edge case height, the amount of intersections in each region should be the same.

Now let's implement this idea into code:

first we should crop the images so that only the signal fills the screen. To do this we pick some width and height and then find the first intersection with the black border from the front and back, then top and bottom:

```py
img = img.convert("RGB")
    h = 50
    w = 0
    lw = 0
    for x in range(20):
        img.putpixel((x,h), (255,0,0))

    rw = img.width - 1
    while w < img.width:
        pixel = img.getpixel((w, h))
        if pixel == (0,0,0):
            lw = w
            break
        w+=1
    
    w = img.width - 1
    while w >= 0:
        pixel = img.getpixel((w, h))
        if pixel == (0,0,0):
            rw = w
            break
        w -= 1
    
    crop_box = (lw+1, 0, rw, img.height)
    img = img.crop(crop_box)

    lu = 0
    ru = img.height - 1
    h = 0
    while h < img.height:
        pixel = img.getpixel((0, h))
        if pixel == (0,0,0):
            lu = h
            break
        h+=1
    h = img.height - 1
    while h >= 0:
        pixel = img.getpixel((0, h))
        if pixel == (0,0,0):
            ru = h
            break
        h -= 1
    crop_box = (0, lu+1, img.width, ru)
    img = img.crop(crop_box)
```

Omg hella inefficient code but it works so now our images become:

![O modified](/images/O_1_modified.png)
![S modified](/images/S_0_modified.png)
![U modified](/images/U_1_modified.png)

We can also now split the image into chunks of 8 by taking the whole width of the image and dividing by 8. Now to check for ASK, we can just check that at a height of `height/2`, the first chunk is all blue, then we check each chunk to see if the amplitude is 0 or not:


```py
width = img.width
height = img.height
step = width // 8

#check if its ASK
ASK = True
for w in range(100):
    color = img.getpixel((w,height//2))
    if color == (255,255,255):
        ASK = False

if ASK:
    print("ASK detected")
    bits = []
    for i in range(8):
        one = False
        for x in range(step-20):
            color = img.getpixel((i*step + x+10, height//2))
            img.putpixel((i*step + x+10, height//2), (100+i*20,0,0))
            if color == (255,255,255):
                one = True
                break
        bits.append(1 if one else 0)
    byte = 0
    for i, bit in enumerate(bits):
        byte += bit << (7 - i)
    return chr(byte)
```

Next for FSK, we just loop through each chunk, and count the frequency by counting intersections of a horizontal line with the blue curve, then see if there are 2 frequencies:

```py
count = []
for i in range(8):
    cnt = 0
    for x in range(step-20):
        color = img.getpixel((i*step + x+10, height//2))
        # check if its hitting the blue curve (not white pixel)
        if color != (255,255,255) and img.getpixel((i*step + x+10-1, height//2)) == (255,255,255):
            cnt+=1
    count.append(cnt)

if len(set(count)) != 1:
    print("FSK detected")
    zero = count[0]
    bits = []
    for c in count:
        bits.append(0 if c == zero else 1)
    byte = 0
    for i, bit in enumerate(bits):
        byte += bit << (7 - i)
    return chr(byte)
```

Finally, for PSK, while this is technically a bit tricky, it's made easier by the fact that we know its a 180 degree shift, and also we are allowed 10 wrong answer. Thus we can do something very hacky, where if we pick some very specific offsets, we can see the difference between a non-phase shifted curve and a phase shifted curve:

![The trick](/images/U_1_modified%20labeld.png)

Thus, if we fiddle around with some $\Delta y$, and $\Delta x$, we should be able to detect when the signal is shifted or not shifted with enough accuracy to pass the challenge.

```py
count = []
# tweak after testing running script on challenge
offset = 15
for i in range(8):
    cnt = 0
    # -2 is also an offset
    for x in range(5,step-2):
        color = img.getpixel((i*step + x, height//2+offset))
        if color != (255,255,255) and img.getpixel((i*step + x-1, height//2+offset)) == (255,255,255):
            cnt+=1
        img.putpixel((i*step + x,height//2+offset+1), (100+i*20,0,0))
    count.append(cnt)

zero = count[0]
bits = []
for c in count:
    bits.append(0 if c == zero else 1)
byte = 0
for i, bit in enumerate(bits):
    byte += bit << (7 - i)
```

Putting it all together we get:

```py
from PIL import Image
from io import BytesIO
import websockets
import asyncio

def decode(img):
    #crop the image
    img = img.convert("RGB")
    h = 50
    w = 0
    lw = 0
    for x in range(20):
        img.putpixel((x,h), (255,0,0))

    rw = img.width - 1
    while w < img.width:
        pixel = img.getpixel((w, h))
        if pixel == (0,0,0):
            lw = w
            break
        w+=1
    
    w = img.width - 1
    while w >= 0:
        pixel = img.getpixel((w, h))
        if pixel == (0,0,0):
            rw = w
            break
        w -= 1
    
    crop_box = (lw+1, 0, rw, img.height)
    img = img.crop(crop_box)

    lu = 0
    ru = img.height - 1
    h = 0
    while h < img.height:
        pixel = img.getpixel((0, h))
        if pixel == (0,0,0):
            lu = h
            break
        h+=1
    h = img.height - 1
    while h >= 0:
        pixel = img.getpixel((0, h))
        if pixel == (0,0,0):
            ru = h
            break
        h -= 1
    crop_box = (0, lu+1, img.width, ru)
    img = img.crop(crop_box)

    width = img.width
    height = img.height
    step = width // 8

    #check if its ASK
    ASK = True
    for w in range(100):
        color = img.getpixel((w,height//2))
        if color == (255,255,255):
            ASK = False

    if ASK:
        print("ASK detected")
        bits = []
        for i in range(8):
            one = False
            for x in range(step-20):
                color = img.getpixel((i*step + x+10, height//2))
                img.putpixel((i*step + x+10, height//2), (100+i*20,0,0))
                if color == (255,255,255):
                    one = True
                    break
            bits.append(1 if one else 0)
        byte = 0
        for i, bit in enumerate(bits):
            byte += bit << (7 - i)
        return chr(byte)

    #check if its FSK
    count = []
    for i in range(8):
        cnt = 0
        for x in range(step-20):
            color = img.getpixel((i*step + x+10, height//2))
            if color != (255,255,255) and img.getpixel((i*step + x+10-1, height//2)) == (255,255,255):
                cnt+=1
        count.append(cnt)

    if len(set(count)) != 1:
        print("FSK detected")
        zero = count[0]
        bits = []
        for c in count:
            bits.append(0 if c == zero else 1)
        byte = 0
        for i, bit in enumerate(bits):
            byte += bit << (7 - i)
        return chr(byte)

    print("PSK detected")
    count = []
    offset = 15
    for i in range(8):
        cnt = 0
        for x in range(5,step-2):
            color = img.getpixel((i*step + x, height//2+offset))
            if color != (255,255,255) and img.getpixel((i*step + x-1, height//2+offset)) == (255,255,255):
                cnt+=1
            img.putpixel((i*step + x,height//2+offset+1), (100+i*20,0,0))
        count.append(cnt)

    zero = count[0]
    bits = []
    for c in count:
        bits.append(0 if c == zero else 1)
    byte = 0
    for i, bit in enumerate(bits):
        byte += bit << (7 - i)
    return chr(byte)

async def main():
    uri = "wss://modulation-master-c826343e1069.instancer.sekai.team/echo"
    async with websockets.connect(
        uri,
    ) as ws:
        await ws.send("start")
        reply = await ws.recv()
        print("Received:", reply)

        while "100/100" not in reply:
            img_response = await ws.recv()

            img = Image.open(BytesIO(img_response))

            char = decode(img)
            print("Identified character:", char)
            
            await ws.send(char)
            reply = await ws.recv()
            print("Received:", reply)

        while True:
            reply = await ws.recv()
            print("Received:", reply)


asyncio.run(main())
```

Output:

```
Received: You have got 96/100 correct (+1), 1/10 wrong.
ASK detected
Identified character: B
Received: You have got 97/100 correct (+1), 1/10 wrong.
PSK detected
Identified character: L
Received: You have got 98/100 correct (+1), 1/10 wrong.
FSK detected
Identified character: o
Received: You have got 99/100 correct (+1), 1/10 wrong.
PSK detected
Identified character: r
Received: You have got 99/100 correct, 2/10 wrong (+1).
ASK detected
Identified character: P
Received: You have got 100/100 correct (+1), 2/10 wrong.
Received: Congratulations! Here's your flag: osu{I_h0p3_y0u_d1dn't_u53_LLM_2_s0Lv3_7H1S}
```

## Funny Fails

Let's just say that someone might have tried to use an LLM:
![llm](/images/LLMattempt.png)

And then someone might have tried to bruteforce all the images and store them to create a lookup table:

![images](/images/storedimages.png)