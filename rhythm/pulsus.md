# pulsus

Category: Rhythm

Points: 156

Solves: 55

Difficulty: 1/5

>Check out Pulsus, a rhythm game for PC in which you hit incoming beats on a 3x3 tile board! There's a flag in my account, can you find it?

## Solution

After going to the game https://pulsus.cc/play/. We navigate to leader board to find Quintec's account

![search](/images/QuintecSearch.png)

![user](/images/RealQuintec.png)

Ok after a bit of having no idea what we are doing, and fiddling around with the page. We can come to see that the page dynamically loads content with websockets. So if we monitor the websocket logs while searching for `Quintec`, we can get back the user data for `Quintec`.

![websocket](/images/QuintecFlag.png)

We then see a flag entry set in his user data:

`osu{pl4y_pul5u5_e6234bd516f}`