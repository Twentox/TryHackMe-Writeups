- in the description off this room their mentioning that we have to use an `IDOR`-vulnerability to get the flag 
- also they talk about `URL` endpoints, that look like hashes 
- so lets look at the website:
![](assets/Corridor_1.png)
- we see a picture with different doors in it
- when we hover over these doors we can see that our cursor changes to a pointer, so that probably means that we can click on these doors 
- at first i clicked on the first door on the left site and we see this: 
![](assets/Corridor_2.png)
- we get greeted by an picture of an empty room
- when we look at the `URL` we see the endpoints that were mentioned: 
```
http://10.64.140.192/c4ca4238a0b923820dcc509a6f75849b
```
- lets see if the endpoint is really a hash
- to check this we can go to https://crackstation.net/ and paste it in: 
![](assets/Corridor_3.png)
- we can see that this was indeed a `MD5` hash and i represented the value `1` 
- if we do this with the other endpoints we see that we have the id's `1-13`  
- what we could try is to hash an `id` that is not yet present like `0` or `14` and request it 
- the `MD5` hash representation of the String `0` is `cfcd208495d565ef66e7dff9f98764da` (https://emn178.github.io/online-tools/md5.html)
- so lets do a request with this `hash`:
```
http://10.64.140.192/cfcd208495d565ef66e7dff9f98764da
```
- and we get the flag: 
![](assets/Corridor_4.png)