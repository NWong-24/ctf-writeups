# Minecraft huh
351 pts / 11 solves

>Author: Kiinzu
>
>Say, everyone knows minecraft right? The game about mining block after block after block after block.....
>
>You only need to spawn an instance, no need to press the "Flag" button. The isSolved() function will always return false after all.

## Challenge outline
For this challenge, we were only given an instancer link and the description. My first thought was to enumerate the previous blocks, so I wrote a python script to do so. After looking through the blocks, I noticed that the input data for one of the blocks looked the flag format: `TCP3P{this_is_one_p_not_three_p}`. Unfortunately, not quite. After a few more blocks I found the real flag: `TCP1P{running_through_some_blocks_have_you?}`.
