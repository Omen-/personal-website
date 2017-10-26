---
title: "Breaking dofus 1.29.1 maps encryption"
date: 2017-10-26T11:43:01+02:00
draft: false
---

**TL;DR**

A few years ago I found a way to retrieve unknown dofus maps keys by exploiting a badly designed XOR encryption. Recently other people achieved the same thing by using statistical based attacks. It got me motivated to share my research with the community.

You can find an implementation of the attack described here on my [github](https://github.com/Omen-/dofus-key-finder).

# Dofus

> Dofus is a Flash-based tactical turn-oriented massively multiplayer online role-playing game (MMORPG) developed and published by Ankama Games, a French computer game manufacturer.

> The game has attracted over 25 million players worldwide and is especially well known in France. Today, there are more than 1.5 million subscribers every month on this game.

[Wikipedia - Dofus](https://en.wikipedia.org/wiki/Dofus)

In 2010, Ankama Games reveal a major update of dofus, Dofus 2.0, with updated graphics and new content. This new version was not well received by the community and it resulted significant loss in the playerbase. 

![dofus](/images/dofus-maps/dofus2-4.png#center)
<center>*Dofus 1.29 / 2.0* - Copyright © 2017 Ankama Games</center>

Similarly to some other games like RuneScape or World of Warcaft, this led to the creation of communities aiming to build [server emulators](https://en.wikipedia.org/wiki/Server_emulator) to recreate the original experience. 

# Asset protection

To protect some of his assets against data mining and private servers, Ankama Games implemented an encryption algorithm using [XOR cipher](https://en.wikipedia.org/wiki/XOR_cipher). The idea is simple : static assets like maps are stored on the client's computers, encrypted. When the asset is needed the server sends the decryption key to the client.

When the player enters a map, the server sends him the decryption key. The game client uses this key to decrypt the map and then displays it.

In order to obtain the key for a certain map, you need to be able to access this map. There is no way to cheat this system. If there was one it would also be a game-breaking bug that lets you teleport on any map.

If you send the wrong key to the client, the map will not be displayed properly.

![bad key](/images/dofus-maps/bad-key.png#center)
<center>*Not quite correct key* - Copyright © 2017 Ankama Games</center>

![good key](/images/dofus-maps/good-key.png#center)
<center>*Correct key* - Copyright © 2017 Ankama Games</center>

# Checking the XOR cipher implementation

I will use `^` to symbolize the [XOR opperator](https://en.wikipedia.org/wiki/Exclusive_or).

Here is the client decryption function (edited to make it more readable) :

```as3
function (data, key, c)
{
  var decryptedData = new String();
  var i = 0;
        
  while (i < data.length)
  {
    decryptedData = decryptedData + (key.charCodeAt(i) ^ 
      key.charCodeAt((i + c) % key.length));
    i = i + 1;
  }
  return decryptedData;
}
```

As you may be able to see, this is almost a standard [XOR cipher](https://en.wikipedia.org/wiki/XOR_cipher) algorithm. The only difference is the parameter called `c` wich adds an offset to the key. This is probably an attempt to make the algorithm safer but nothing happens in practice. You do not need to understand why to continue with the rest of this post and can just ignore this but it only means that we will need to apply a [circular shift](https://en.wikipedia.org/wiki/Circular_shift) on the key before sending it to the client.

[XOR cipher](https://en.wikipedia.org/wiki/XOR_cipher) can be a strong encryption algorithm if the following conditions are met:

* The data and the key are about the same length.
* The key cannot be guessed
* The data cannot be guessed

To make you understand why this is important, let's check if our case meets all this requirements and what we can do with the results.

Note: We have access to all the encrypted data from the client and to about 65% of the keys from the server emulators community. This is what allows us to make statistics about keys and decrypted data.

### 1. The data and the key are about the same length.

![encryptedData length distribution](/images/dofus-maps/edl-distribution.png#center)
<center>*Encrypted data length distribution*</center>

![key length distribution](/images/dofus-maps/kl-distribution.png#center)
<center>*Key length distribution*</center>

As you can see, most of the times the data and the key do not have the same length by one order of magnitude. 

#### Why is it important ?

[XOR](https://en.wikipedia.org/wiki/Exclusive_or) has an intresting property that make [XOR cipher](https://en.wikipedia.org/wiki/XOR_cipher) possible and XOR so usefull in computer science :
```
a == a ^ b ^ b
> true
```
This property can be used to prove another property : 
```
xor1 = a ^ c
xor2 = b ^ c
xor1 ^ xor2 == a ^ b
> true
```
---
Since the key length is lower than the data length, it means that during encryption each part (byte) of the key is going to be applied multiple time on the data.  
```
encryptedData[0] = data[0] ^ key[0]
encryptedData[keyLength] = data[keyLength] ^ key[0]
```
As you can see this correspond to the two first lines of the previous property :

* `encryptedData[0]` is `xor1`
* `encryptedData[keyLength]` is `xor2`
* `data[0] ` is `a`
* `data[keyLength]` is `b`
* `key[0]` is `c`

Wich means :

```
encryptedData[0] ^ encryptedData[keyLength] == data[0] ^ data[keyLength]
> true
```

We can extend this reasoning to a more general case :
```
encryptedData[x*keyLength + z] ^ encryptedData[y*keyLength + z] == 
data[x*keyLength + z] ^ data[y*keyLength + z]
> true
```

This tool is not usefull in itself, but if we can somehow estimate the possible values for `data[x*keyLength + z]` and `data[y*keyLength + z]`, we may be able to deduce their real values.

**Example :** 

`data[0]` is equal to one of this values : `[25,2,18]`

`data[keyLength]` is equal to one of this values : `[5,13,48]`

`encryptedData[0] ^ encryptedData[keyLength]` is equal to `15`

We compute all possible values for `data[0] ^ data[keyLength]` : 

```
25 ^ 5 = 28   25 ^ 13 = 20    25 ^ 48 = 41
 2 ^ 5 = 7     2 ^ 13 = 15     2 ^ 48 = 50
18 ^ 5 = 23   18 ^ 13 = 31    18 ^ 48 = 34
```

Since the only values for `data[0]` and `data[keyLength]` able to produce `15` are `2` and `13` we can deduce that it is their real value.

Getting the part of the key used to encrypt `encryptedData[0]` and `encryptedData[keyLength]` back is trivial :

```
key[0] = encryptedData[x*keyLength] ^ 2
```
Or
```
key[0] = encryptedData[y*keyLength] ^ 13
```

To find `key[1]` we can then use the same method again on `data[1]` and `data[keyLength+1]` and so on.

### 2. The key cannot be guessed.

To check if the key can be guessed we do not have many options :

* We do not have the software used to generate the keys and thus can not know wich pseudo-random implementation has been used.
* We have a set of known keys that we can use to find patterns.

By building statistics about the keys I discovered that they only contains values between 32 and 127, wich correspond to displayable ASCII characters. This choice was probably made to allow the key to be copy-pasted.

#### Why is it important ?

This discovery significantly reduces the amount of possible keys. Like for the previous tool, if we can somehow have a set of possible values for `data[x]`, we may be able to eliminate some of them.

**Example :** 

`data[0]` is equal to one of this values : `[25, 40, 140, 166]`

`encryptedData[0]` is equal to `10`

We compute all possible values for `key[0]` ( equal to `data[0] ^ encryptedData[0]`) : 

```
25 ^ 10 = 19
40 ^ 10 = 34
140 ^ 10 = 134
166 ^ 10 = 172
```

To have a `data[0]` equal to `140` or to `166` we would need an invalid key. This proves that `data[0]` is not equal to any of thoses two values.

⇒

`data[0]` is equal to one of this values : `[25, 40]`

⇒

`key[0]` is equal to one of this values : `[19, 34]`

### 3. The data cannot be guessed.

To answer this hypothesis, we need to understand the format of dofus maps data.

The data is divided in chunks of 10 bytes that each represents one cell. Thoses ten bytes all have a specific role in describing the cell : they inform the game if the cell is walkable, what is it's texture, etc..

Here is an example of the begining of a decrypted map data containing 7 cells :

<pre>
<code><span style="color:	#a93226">HhfDaaaaaa</span><span style="color:	#9b59b6">HhfDeaaaaa</span><span style="color:#2980b9">HhfDeaaaaa</span><span style="color:#229954">HhfDeaazaa</span><span style="color:#f1c40f">HhfDeaazaa</span><span style="color:  #ba4a00">HhfDeaaaaa<span style="color: #34495e">HhfDeaaaaa</span></code>
</pre>

It is easy to see that some bytes of the cells tend to have the same values. Here, the first byte is always `H`.

Thoses observations led me to make statistics for each of theses 10 bytes.

![first byte of cells distribution](/images/dofus-maps/1bc-distribution.png#center)
<center>*Distribution of values for the first byte in cells*</center>

As you can see, for the first byte, there are only 6 possibilites, with 3 of them appearing in less than 1% of the cases.

I did the same for the remaining 9 bytes and the number of possibilities ranges from 10 to 63. If you want to take a look at the full statistics, you can do so [here](/etc/cell_stats.json).

#### Why is it important ?

We now have [10 arrays](/etc/cell_stats.json) containing the possibles values for each byte of any cell. We will name the array containing these 10 arrays `possibleValuesForCellAtPosition[10][]`

We could use this to build an array of possible values for each byte of the data of a map.

**Example in pseudocode :** 

```go
func initDecryptedDataPossibilitiesUsingMethod3(int dataLength)
  byte decryptedDataPossibilities[dataLength][]
  for i = 0;i < len(decryptedDataPossibilities);i++
    decryptedDataPossibilities[i] = possibleValuesForCellAtPosition[i % 10]
  return decryptedDataPossibilities
```


## Building the attack

During our analysis of this XOR cipher implementation, we discovered 3 ways to narrow down the possible values for the decrypted data and therefore the key. The objective is now to merge all of this in an algorithm that will hopefully be able to reduces the possibles values for each part of the key to only one.

Here is my implementation at the highest level in pseudocode :

Note : `decryptedDataPossibilities[][]` is a 2 dimensional array that contains all the possible decrypted values for the decrypted data. For example `decryptedDataPossibilities[0]` contains all the possible values for `data[0]`

```go
int dataLength = length(encryptedData)
byte decryptedDataPossibilities[dataLength][]
byte key[keyLength]
decryptedDataPossibilities = initDecryptedDataUsingMethod3(dataLength)
eliminateValuesMethod2(encryptedData, keyLength, decryptedDataPossibilities)
eliminateValuesMethod1(encryptedData, keyLength, decryptedDataPossibilities)

for i = 0;i < length(decryptedDataPossibilities);i++
  if length(decryptedDataPossibilities[i]) == 1
    key[i % keyLength] = decryptedDataPossibilities[i][0] ^ encryptedData[i]

if noValueIsNullInArray(key)
  print "Found key :" + key
else
  print "Multiple keys possible try statistical analysis on remaining possibilities"
```

However, as you may have noticed, we do not have the key length. To find its value we apply the previous algorithm to each known key value (128 - 277) and after we narrowed down the values using the 3 methods we check if `decryptedDataPossibilities` still contains at least one values at all positions. If it does not, there is no solution for a valid key with this length and we can proceed to the next length.

```go
int dataLength = length(encryptedData)
for keyLength = 128;keyLength <= 277;keyLength++
  byte decryptedDataPossibilities[dataLength][]
  byte key[keyLength]
  decryptedDataPossibilities = initDecryptedDataUsingMethod3(dataLength)
  eliminateValuesMethod2(encryptedData, keyLength, decryptedDataPossibilities)
  eliminateValuesMethod1(encryptedData, keyLength, decryptedDataPossibilities)

  for i = 0;i < length(decryptedDataPossibilities);i++
    if length(decryptedDataPossibilities[i]) == 0
      continue
    else if length(decryptedDataPossibilities[i]) == 1
      key[i % keyLength] = decryptedDataPossibilities[i][0] ^ encryptedData[i]

  if noValueIsNullInArray(key)
    print "Found key :" + key
    return key
  else
    print "Multiple keys possible try statistical analysis on remaining possibilities"
```

With this algorithm, it is possible to find the full key of about 60% all maps. 

To explain why we can't find all the keys here is an interesting benchmark : 

```
Percentage of the key found for keys with keyLength % 10 equal to
0: 0.192308 % on average
1: 99.861222 % on average
2: 35.013849 % on average
3: 99.215823 % on average
4: 33.853928 % on average
5: 5.542589 % on average
6: 39.950291 % on average
7: 96.070707 % on average
8: 36.983387 % on average
9: 99.421586 % on average
```

The results depends on the length of the key. This is due to the way method 1. works : the more diversity there is between the two arrays of possibilites we are trying to narrow, the better the result will be. If the keyLength % 10 = 0 we will always use method 1. with two bytes that correspond to the same cell. There will then be very little diversity in the two array of possibilities.

From there the best way to find the missing parts of the is to use a statistical approach.

You can find an implementation of this attack on my [github](https://github.com/Omen-/dofus-key-finder).

If you want more details on any part of this attack you can contact me, I will be more than happy to aswer !