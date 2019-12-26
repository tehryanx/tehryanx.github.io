---
layout: post
title:  "Cracking a Christmas Story"
date:   2019-12-25
categories: puzzles
---

*There's a scene in the classic Christmas movie "A Christmas Story" where nine-year-old Ralphie uses a secret decoder pin to decode a secret message from his favorite radio program Little Orphan Annie.*

[![Ralphie's decoder pin](http://img.youtube.com/vi/zdA__2tKoIU/0.jpg)](http://www.youtube.com/watch?v=zdA__2tKoIU "Ralphie's decoder pin")

My partner and I were watching this the night before Christmas Eve and as soon as I saw that scene I started wondering whether or not I could crack the encryption scheme used in the old Little Orphan Annie series without actually having the decoder pin. 

Step one was to try and track down some old episodes of the radio series. Fortunately, archive.org has a [nice collection](https://archive.org/details/Little-Orphan-Annie/). I started going down the list and fast-forwarding the episodes to the end to see if a secret message was given. I started with three messages, found at the ends of [07 (11:00)](https://archive.org/details/Little-Orphan-Annie/1936-06-17-ep1165-2-of-4-Who-Shot-Tom-Baines-Dog.mp3), [10 (11:30)](https://archive.org/details/Little-Orphan-Annie/1936-ep1018-1-of-4-Mr-Flint-Is-Selling-Stock-In-Toll-Bridge.mp3), and [12 (11:50)](https://archive.org/details/Little-Orphan-Annie/1936-ep1020-3-of-4-Watching-The-Bridge-Being-Built.mp3). Following is the example from episode 10. 

> "Attention everybody please, for an important secret message broadcast in Annies new 1936 mystery radio code. So all you 1936 members get your pencils and papers ready to take it down. First, we always give you the special code key, and tonights secret message is coming in the o-21 code. Did you get that? o-21 is the special code key for tonights secret message so write o-21 down on your paper now so you won't forget it. And here's the secret message itself: First word: 24, 12, 12, 17, 13. Second word: 14, 17, 6, 17, 26, 6. Third word: 15, 24, 22, 13, 6. Fourth and last word: 13, 4, 1, 21, 20, 17, 19, 4. That's all, and that was another secret message in Annies new 1936 radio code!"

First, we're given a secret decoding key, which is in the form of a letter and a number. The messages then come as a list of numerical values with word boundaries retained. Here are the three examples we'll be looking at. 

ep | key | secret message
---|-----|----------------
7  | Y19 | `7 16 16 4 / 3 24 7 24 23 23 24 21 / 20 15 / 9 20 14 24 / 5 1 23 15`
10 | O21 | `24 12 12 17 13 / 14 17 6 17 26 6 / 15 24 22 13 6 / 13 4 1 21 20 17 19 4`
12 | A13 | `19 5 21 15 2 9 6 10 8 21 / 13 20 20 6 23 2 1 15 / 13 15 / 17 9 6 23 14 2`

Having seen the decoder pin in the movie, I assumed this was some kind of substitution cipher. This is where every letter in the regular alphabet is substituted with a letter from an encrypting alphabet. For example, Imagine the encrypting alphabet is just a reversed version of the normal alphabet. Anywhere you'd normally use an A, you'd instead use a Z. B would be replaced with Y, C with X, etc. ABC would therefore encrypt to ZYX. In the Little Orphan Annie cipher, numbers are used instead of letters, but they appear to be in the range 1-26 so it's likely that the numbers just pair with the letters of the alphabet. In our example, rather than ABC encrypting to ZYX it would encrypt to 26 25 24.

Having never actually played with one of these pins, I had to make some assumptions about how they work. Picture an analog clock-face with 26 numbered places rather than 12. Now imagine a smaller disk with the 26 letters of the alphabet around its circumference, each letter lining up with a number in the outer clock-face. The disk can be rotated so that any letter can align with any number. In it's base position, the A aligns with 1, B with 2, C with 3, etc. The central disk can easily be rotated so that A aligns with 2 and so on. This is how I assume the pin works based on what I could see in the film. 

When the announcer gives you a "secret key," you line up the letter with the number accordingly, and then each number in the message corresponds to a letter on the disk. So for example, In the first of the three secret messages above, you start by turning the pin so that the letter Y lines up with the number 19, and then the rest of the numbers will line up with the letters in the secret message. Assuming the pin works this way, all we have to do to reconstruct it is determine the order of the numbers in the outer circle and the order of the letters on the inner circle. 

The easiest possible solution would be a simple caesar-style shift cipher. This would mean that the numbers in the outer circle would be in numerical order going clockwise from 1 through 26. The letters on the inner circle would be in alphabetical order going clockwise from A through Z. In it's base position A would line up with 1, B with 2, etc. In that position the word CAT would encrypt to "3 1 20". By rotating the inner circle using a different key, that same message could encrypt to a completely different result. For example, by rotating the inner circle one position clockwise (which is the same as using the secret key A-2) the message would encrypt to "4 2 21". You might be familiar with a version of this cipher called "ROT-13". This is where you take the standard alphabet and shift it to the right 13 characters as below, or in the case of our decoder pin, rotate the inner circle clockwise 13 positions.  

1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10| 11| 12| 13| 14| 15| 16| 17| 18| 19| 20| 21| 22| 23| 24| 25| 26
--|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---
A | B | C | D | E | F | G | H | I | J | K | L | M | N | O | P | Q | R | S | T | U | V | W | X | Y | Z
N | O | P | Q | R | S | T | U | V | W | X | Y | Z | A | B | C | D | E | F | G | H | I | J | K | L | M

Using this cipher, the phrase "HELLO WORLD" would encrypt to "URYYB JBEYQ." If the Little Orphan Annie announcer were to use this code in a secret message he might tell the listeners to use the key A-14. Of course, because the key is just telling you how to line up the inner circle with the outer circle, he could give you any pairing in the resulting configuration. T-7, G-20, or O-2 would all work equally well. 

This kind of shift cipher is really easy to test for. We just need to shift the alphabet over according to the key that the announcer gives and see if the secret message makes sense. The first message is given with a key of Y-19. I took the standard alphabet, shifted it so that Y aligned with 19, and replaced the numbers in the message with the corresponding letters. 

The shifted alphabet looked like this: 

1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10| 11| 12| 13| 14| 15| 16| 17| 18| 19| 20| 21| 22| 23| 24| 25| 26
--|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---
G | H | I | J | K | L | M | N | O | P | Q | R | S | T | U | V | W | X | Y | Z | A | B | C | D | E | F 

And the message decrypted as follows:

7 | 16| 16| 4 | / | 3 | 24| 7 | 24| 23| 23| 24| 21| / | 20| 15| / | 9 | 20| 14| 24| / | 5 | 1 | 23| 15
--|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---
M | V | V | J | / | I | D | M | D | C | C | D | A | / | Z | U | / | O | Z | T | D | / | K | G | C | U

Gibberish! I wasn't quite at a solution yet, but I still felt that I was on the right track. If I was right about the mechanics of the device then I knew it had to be a shift cipher. Mechanically, the order of the letters couldn't change. The only transformation that could occur was that the entire alphabet could shift some number of positions either clockwise or counterclockwise. The fact that the key gave a letter and a number convinced me that the key's purpose is to tell the listener what position to shift it to. So why wasn't this working? The letters must not be in alphabetical order!

Looking at the message as letters rather than numbers clicked my brain into thinking about language patterns. 

MVVJ IDMDCCDA ZU OZTD KGCU

This isn't the correct message, but a few things do jump out at me. First, MVVJ is a four letter word with the two center letters repeated. There is a fairly limited subset of english words that fit this pattern. The third word ZU is only two letters long, and there aren't that many two-letter english words. If I could work out which two letter word that is, I'd know the Z in the next word, and the U in the last word. Like dominos, each solved word gives clues to help solve other words. The other two messages have interesting patterns of their own.

I know there are cryptanalysts and puzzlers out there who could just stare at these for a few hours and piece them together letter by letter. I decided instead to see if I could write a tool to help me. Since I didn't know the order of the letters on the inner circle, the differences between the numbers in the message didn't tell me anything useful. For example, if the letters were in alphabetical order then the numbers 10 11 could only decrypt to pairs like AB, BC, CD, DE, EF, etc. but because the letters on the inner circle weren't in alphabetical order, the numbers 10 11 could decrypt to ANY pair except for AA. There were only two things that I felt really confident I knew for sure. 

1) I knew that the position of the number in the encrypted message represented the position of the corresponding letter in the decrypted message. 
2) Each number in the message represented a letter, and that number always represented that same letter until the secret key was changed. 

I realized that I could take a word and convert it into a pattern that only cares about those two pieces of information, and then I could search the english dictionary for other words that match those patterns. For example, the first word in the first message is 7 16 16 4. This might be BEER, or BOOT, or MOON, etc. We could represent this using the pattern ABBC, which would match everything that this word could possibly decrypt to according to the two pieces of information we know about it. 

7 16 16 4 | ABBC
----------|-----
BEER | ABBC
BOOT | ABBC
MOON | ABBC

The following python function can create these patterns from a given word:

```
def make_pattern(word):
        word = word.replace('\n', '') 
        symbols = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ' 
        cursor = 0 
        pattern_key = {} 
        pattern = ''
        for i in word:
		# step through every letter in our word
                if i in pattern_key.keys():
			# If the letter already exists in the pattern, use the same symbol we've been using for it. 
                        pattern += pattern_key[i]
                else:
			# otherwise, use the next available symbol in the symbols list. 
                        pattern += symbols[cursor%len(symbols)]
                        pattern_key[i] = symbols[cursor%len(symbols)]
                        cursor += 1
        return pattern
```

![make_pattern]({{ site.url }}/assets/loa1.png)

Next, I could run this function against every line in an english dictionary wordlist and create a dataset of words with their patterns. I could then search that dataset for all english words corresponding to a given pattern. I decided to use this 4mb file I found on github: 

wget https://github.com/dwyl/english-words/blob/master/words_alpha.txt?raw=true

I created a huge list of patterns using make_pattern() 

![make_pattern]({{ site.url }}/assets/loa2.png)

And then grepped through that list for words that matched words in the secret messages. In the following screenshot I'm grepping for words matching the first word in the first message. That word was given by the announcer as: 7 16 16 4, which can be represented using our pattern system as ABBC. I quickly realized that my dictionary contained a lot of words that are unlikely to occur in a radio show designed for children. 

![make_pattern]({{ site.url }}/assets/loa3.png)

I decided that a better data source might be novels, as they'd have a much smaller list of more commonly used words which should significantly improve my signal to noise ratio. I downloaded the first five books on project gutenbergs top 100 list, concatenated them into a single file, and turned it all into a wordlist.

![make_pattern]({{ site.url }}/assets/loa4.png)

Still, I was getting a lot of results for 7 16 16 3. I realized that the best course would be to find a word in the secret message that was relatively long, but also contained a significant amount of repeating characters. The second word in the first message fit the bill. 

`3 24 7 24 23 23 24 21 -> ABCBDDBE`

![make_pattern]({{ site.url }}/assets/loa5.png)

From these results, I thought "tomorrow" made the most sense for the context and plugged that into the message, filling out any other positions that used those letters. 

7 | 16| 16| 3 | / | 3 | 24| 7 | 24| 23| 23| 24| 21| / | 20| 15| / | 9 | 20| 14| 24| / | 5 | 1 | 23| 15
--|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---
M |   |   | T | / | T | O | M | O | R | R | O | W | / |   |   | / |   |   |   | O | / |   |   | R |  

Just looking at this I immediately knew that the first word was MEET. The only words that made sense in the third position would be IN or AT, and AT was out because we know that T is 3, not 15. Using this new information I filled it out further.

7 | 16| 16| 3 | / | 3 | 24| 7 | 24| 23| 23| 24| 21| / | 20| 15| / | 9 | 20| 14| 24| / | 5 | 1 | 23| 15
--|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---
M | E | E | T | / | T | O | M | O | R | R | O | W | / | I | N | / |   | I |   | O | / |   |   | R | N 

MEET TOMORROW IN something something. The next word is "_I_O", and we know it has no repeating letters as the numbers (9 20 14 24) are all different. In order to find candidate words, I grepped the dataset using the pattern "ABCD" (for four letter words with no repeating letters), then grepped those results for words that have the I and O in the proper positions, and none of the letters we've already discovered in the remaining positions. Searching the shorter wordlist returned no results, so I went back to the much larger dictionary file. 

![make_pattern]({{ site.url }}/assets/loa6.png)

It wasn't immediately obvious to me which of the results made sense. I tried the same thing with the last word "__RN"

![make_pattern]({{ site.url }}/assets/loa7.png)

Looking over all the results I became fairly confident that the message decoded to "MEET TOMORROW IN THE SILO BARN." I now had quite a few letters that I could use to begin reconstructing the decoder pin. I knew that, when the key was Y-19, the alphabet was ordered as follows:

1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10| 11| 12| 13| 14| 15| 16| 17| 18| 19| 20| 21| 22| 23| 24| 25| 26
--|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---
  |   | T |   | B |   | M |   | S |   |   |   |   | L | N | E |   |   | Y | I | W |   | R | O |   |   

The next step was to move on to the second secret message to see if I could extract more of the alphabet. Fortunately, I didn't have to start from scratch this time. I'd already recovered a lot, but it was using the Y-19 key. The second message was encrypted using the o-21 key. Because we're using a circular shift cipher, we should be able to simply shift our recovered alphabet to the O-21 position and use it to decode the next message. We've already discovered that in the Y-19 configuration, O is in the 24th position. We just need to shift the alphabet to the left (or rather, rotate the decoder pin counter-clockwise) three positions. 

1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10| 11| 12| 13| 14| 15| 16| 17| 18| 19| 20| 21| 22| 23| 24| 25| 26
--|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---
  | B |   | M |   | S |   |   |   |   | L | N | E |   |   | Y | I | W |   | R | O |   |   |   |   | T  

Now that we're in this position we can start solving the second secret message: 

24| 12| 12| 17| 13| / | 14| 17| 6 | 17| 26| 6 | / | 15| 24| 22| 13| 6 | / | 13| 4 | 1 | 21| 20| 17| 19| 4
--|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---
  | N | N | I | E |   |   | I | S | I | T | S |   |   |   |   | E | S |   | E | M |   | O | R | I |   | M 

I immediately saw that this probably decrypted to ANNIE VISITS ___ES EMPORIUM. Using the technique developed earlier, I attempted to work out the remaining word, but there were too many results. 

24| 12| 12| 17| 13| / | 14| 17| 6 | 17| 26| 6 | / | 15| 24| 22| 13| 6 | / | 13| 4 | 1 | 21| 20| 17| 19| 4
--|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---
A | N | N | I | E |   | V | I | S | I | T | S |   |   | A |   | E | S |   | E | M | P | O | R | I | U | M 

![make_pattern]({{ site.url }}/assets/loa8.png)

A quick google search for "little orphan annie" +emporium revealed that the emporium in the series was owned by Jake, which totally fits as JAKES EMPORIUM. We can now fill out our decoder pin even further: 

1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10| 11| 12| 13| 14| 15| 16| 17| 18| 19| 20| 21| 22| 23| 24| 25| 26
--|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---
P | B |   | M |   | S |   |   |   |   | L | N | E | V | J | Y | I | W | U | R | O | K |   | A |   | T  

We can shift this again to the key for the third message, and attempt to decode it. The third message uses the key A-13: 

1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10| 11| 12| 13| 14| 15| 16| 17| 18| 19| 20| 21| 22| 23| 24| 25| 26
--|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---
N | E | V | J | Y | I | W | U | R | O | K |   | A |   | T | P | B |   | M |   | S |   |   |   |   | L  

And given what we've reconstructed of the alphabet, it decodes to the following: 

19| 5 | 21| 15| 2 | 9 | 6 | 10| 8 | 21| / | 13| 20| 20| 6 | 23| 2 | 1 | 15| / | 13| 15| / | 17| 9 | 6 | 23| 14| 2
--|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---
M | Y | S | T | E | R | I | O | U | S | / | A |   |   | I |   | E | N | T | / | A | T | / | B | R | I |   |   | E

That second word jumped right out at me as ACCIDENT because of the duplicate letters in the second and third position. That revealed the D that clued me into the final word being BRIDGE. "MYSTERIOUS ACCIDENT AT BRIDGE." This last message enabled us to extract two more letters from the encrypting alphabet, which is now as follows: 

1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10| 11| 12| 13| 14| 15| 16| 17| 18| 19| 20| 21| 22| 23| 24| 25| 26
--|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---
N | E | V | J | Y | I | W | U | R | O | K |   | A |   | T | P | B |   | M | C | S |   | D |   |   | L  

If you've read this far already I won't bore you with any more analysis. I went through a few more messages so I could extract the rest and give you a complete alphabet. I ended up finding everything except for Z, X, and Q. Here it is in the A-13 configuration: 

1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10| 11| 12| 13| 14| 15| 16| 17| 18| 19| 20| 21| 22| 23| 24| 25| 26
--|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---
N | E | V | J | Y | I | W | U | R | O | K |   | A | G | T | P | B | H | M | C | S |   | D | F |   | L  

Using this alphabet you should be able to decrypt any of the Little Orphan Annie secret messages sent in 1934. New pins were released anually so if you want to decode messages from other years you'll have to do some work, but this walkthrough should help.

This was a really fun excercise, and I think I might try this again for some of the other radio shows that released decoder pins like CAPTAIN MIDNIGHT. I wonder if any of them used more complex mechanisms / algorithms...?

# MERRY CHRISTMAS!


