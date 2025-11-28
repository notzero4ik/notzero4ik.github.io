---
layout: post
title:  "Digging into Annelids multiplayer"
date:   2025-11-27 22:27:08 +0300
categories: annelids
---
Hello! When I started writing this article it should have been github repository, but after i created this blog I can finally post here everything I have in my mind.

When I just got my first phone (using it was a pain) Micromax Q415, my first game was the Annelids. It's hugely expanded remake of old game Liero, where you play as a worm and kill other worms with many different weapons. I loved this game so much that I still playing it sometimes.

My uncounted Annelids phase started exactly when I started learning reverse-engineering, which was pretty unlucky for the game. We assembled a team with a few friends and decided to replicate game behaviour. 

To explain, there's two different servers in game: European (api.annelids.io) and American (ms.annelids.io), which both work in the same way. Servers have rooms for players to play with different gamemodes and maps up to 6 players. To connect to the room you first need to know what rooms do you have, so game connects to the server on 12360 port via TCP, and then sends two packets to get response. I made python code to do the same:
{% highlight python %}
# import the library
import socket

# create TCP socket
with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as sock:
    # connect to host/port
	sock.connect(("api.annelids.io", 12360))
	# send bytes
	sock.send(bytes.fromhex("12 00 00 00 01 00"))
	sock.send(bytes.fromhex("08 3c 10 f7 80 04 18 ea e1 04 28 00"))
	# get response
	data = sock.recv(65536)
	d = repr(data)
	# print and end the program
	print(d)
{% endhighlight %}
So the code just sends same bytes that sended by the game to replicate its behaviour.
Now lets look into output:
{% highlight ruby %}
b'#\x02\x00\x00\x03\x00\n!\x18\xf7\x80\x04M<\x04\x00\x00"\x01-*\rrandom_ground0\x028\x01@\x06\n&\x18\xf7\x80\x04M<D\x00\x00"\x01-*\x12maps/corridors.map0\x028\x04@\x06\n&\x18\xf7\x80\x04M<\x0c\x00\x00"\x01-*\x12maps/corridors.map0\x068\x00@\x06\n&\x18\xf7\x80\x04M<,\x00\x00"\x04100k*\x0fmaps/cliffs.map0\x018\x00@\x06\n!\x18\xf7\x80\x04M<$\x00\x00"\x01-*\rmaps/xmas.map0\t8\x01@\x06\n"\x18\xf7\x80\x04M<0\x00\x00"\x01-*\x0erandom_electro0\x018\x06@\x06\n\x1e\x18\xf7\x80\x04M<X\x00\x00"\x01-*\nrandom_sea0\x018\x04@\x06\n\x1f\x18\xf7\x80\x04M<\x10\x00\x00"\x01-*\x0brandom_city0\x018\x02@\x06\n \x18\xf7\x80\x04M<4\x00\x00"\x01-*\x0crandom_space0\x018\x04@\x06\n#\x18\xf7\x80\x04M<@\x00\x00"\x01-*\x0fmaps/cliffs.map0\x018\x01@\x06\n#\x18\xf7\x80\x04M<(\x00\x00"\x01-*\x0fmaps/cliffs.map0\x048\x00@\x06\n \x18\xf7\x80\x04M<\x14\x00\x00"\x01-*\x0crandom_pipes0\x088\x03@\x06\n\x1e\x18\xf7\x80\x04M<\x1c\x00\x00"\x01-*\nrandom_sea0\x038\x00@\x06\n\'\x18\xf7\x80\x04M<\x00\x00\x00"\x01-*\x13maps/fortresses.map0\x058\x00@\x06\n!\x18\xf7\x80\x04M<\x08\x00\x00"\x01-*\rmaps/xmas.map0\x078\x00@\x06'

{% endhighlight %}
Bruh. Looks unreadable. The only thing that you can understand is name of the maps, and even although it might look useless in understading this response, its actually what I needed. We can divide different rooms by map names, so it will be easier to understand. Let's take random room string:
```
\n'\x18\xf7\x80\x04M<\x00\x00\x00"\x01-*\x13maps/fortresses.map0\x058\x00@\x06\n!
```
We can see that there's \n in the beginning and in the end, which defines that the previous room string ended. That's really useful because if you look at three next bytes after map's name, you can see that `\x00@\x06` is current player count. There's also `\x058` byte, which stands for the gamemode.
With that being said, we can make our own rooms viewer! I expanded my script to make map enumerator, so `maps/fortresses.map` will be `Fortresses` and etc.:
{% highlight python %}
k = map_enum.keys()
	for j in k:
		if j in i:
			s = f"{s}    {map_enum[j]}"
{% endhighlight %}
It takes all the keys from dictionary, checks if it is there in room string, and then adds it into another string. This will look like this:
```
maps/xmas.map
>>> Christmas
```
Now lets find the gamemode. There's 9 different gamemodes in the game, so we can just look into every next byte after the map, then go to the game, and find room with that map to know what gamemode it is:
{% highlight python %}
d=d.split('"')
for i in d:
	if "x018" in i:
		if "100k" in i:
			s = "Deathmatch (100 kills)"
		else:
			s = "Deathmatch"
	elif "x028" in i:
		s = "team deathmatch"
	elif "x038" in i:
		s = "Capture the flag"
	elif "x048" in i:
		s = "Conquest"
	elif "x058" in i:
		s = "King of the hill"
	elif "x068" in i:
		s = "Egg hunt"
	elif "x078" in i:
		s = "Team egg hunt"
	elif "x088" in i:
		s = "crown"
	elif "t8" in i:
		s = "zombie"
{% endhighlight %}
here I splitted the whole response into the strings by quote, even although it would be better to make it with `\n`, I just forgot about it.
And the last thing to make script ready was to get player count:
{% highlight python %}
ind = i.find("@")
cnt = i[ind-2:ind+5].replace("x", "").replace("@", "")
print(s + "    " + cnt)
{% endhighlight %}
So we find index of the `@` byte, then we remove x and @ and add it to the final string. Now we can look into smooth output:
```
team egg        City    
team deathmatch        Jungle    02\06
Egg hunt        Igloos    00\06
Deathmatch (100 kills)        Sea    00\06
zombie        Underground    00\06
Deathmatch        Flat world    06\06
Deathmatch        Christmas    06\06
Deathmatch        Jungle    04\06
Deathmatch        Fortresses    06\06
Deathmatch        Flat world    05\06
Deathmatch        Igloos    06\06
Deathmatch        Igloos    
Deathmatch        Circuit    05\06
Deathmatch        Space    01\06
Deathmatch        Inferno    02\06
Deathmatch        Ice cave    05\06
Conquest        Jungle    00\06
crown        Underground    00\06
Capture the flag        Inferno    01\06
King of the hill        City    01\06
Team egg hunt        City    00\06

```
:)
