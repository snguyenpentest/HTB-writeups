# HTB Jeeves Overview

[The Jeeves pwnbox](https://app.hackthebox.com/challenges/167) is an easy-rated retired Challenge. It features concepts of Binary Exploitation, Fuzzing, Buffer Overflow, and a basic understanding of python.

I used [@PinkDraconian](https://github.com/PinkDraconian) and [@dannytzoc](https://github.com/dannytzoc) YouTube videos for inspiration to better understand binary exploitation and how to execute buffer overflow on this challenge. I highly suggest you give them a follow for more OffSec and HTB CTF content!

https://www.youtube.com/watch?v=W5dVsa3__N4

https://www.youtube.com/watch?v=03Qm8h1L3nw

## Download the file
As with all pwn challenges, we first must download the file from HTB's website.
![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/ab61c0dd-c312-4ed3-a87d-c115d7fb7b15)

To open this file, we have to use the `chmod +x file` command to give us execute privileges:

`chmod +x jeeves`  <br> ![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/84626b55-20f6-4575-ad8c-413884bc6033) 

To find out what kind of file we're working with, use the `file filename` command:

`file jeeves `  <br>
![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/b97ede57-a159-4ecf-be78-7c79c94443d8)

Checking out the binary:  <br> • it's 64-bits  <br> • it's not stripped which means that all the original names are still in there. The function names, etc.

## Ghidra

Now that we have the file information, we will use Ghidra: https://github.com/NationalSecurityAgency/ghidra

Ghidra is a very nice tool that allows us to generate some pseudo c code from the non-user-friendly assembly code.

`ghidra jeeves`  <br> ![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/be62eef1-1d39-484b-82c4-4e20f1f43211)

From opening jeeves into Ghidra, we can see that there's only really a relevant `main` function here from the `/symbol` tree, so let's break down the `main` function in a simple way.

> 1. First Output:  <br> _"Hello, good sir! May I have your name?_
> 
> 2. You give it an Input:  <br> _{name}_
> 
> 3. It Outputs the received Input:   <br> _Hello {name}, hope you have a good day!_
> 
> 4. However, if this {name} variable is equal to {0x1337bab3} it's going to open flag.txt and print it out to us.  <br> _"Pleased to make your acquaintance. Here's a small gift: {name}, {flag.txt}"_

Okay, so let's see if this is also what happens when we run the program. 

When we execute the file using the `./jeeves` command, we see the following (as expected):

`./jeeves`  <br>
![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/8184b27b-bc18-4ba7-b353-2251c81fff08)  <br>
Now that we know the output behavior when the input is expected, let's try to fuzz the program to see if we can generate an unexpected result.

## Fuzzing
Why Fuzz? 

[According to OWASP](https://owasp.org/www-community/Fuzzing), _"The purpose of fuzzing relies on the assumption that there are bugs within every program, which are waiting to be discovered. Therefore, a systematic approach should find them sooner or later."_  <br> 

To find our approach, we must understand the stack requires information from the top and then calls those functions/inputs to be used in later lines.  <br> ![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/a7dbb9a1-e529-4dec-bb70-ced8c18bc5fd)  <br> ![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/445713e5-50d9-4e3c-a394-f4541883f81c)

This leads us to the approach to the theme of the box to use **Buffer Overflow** on this stack. 

### Buffer Overflow
Since our buffer input is on top of the stack, if we overflow there, everything below it could also be overflowed... including `local_c` in Line 8, so we need to find a way to overflow lines 5-7, but not 8.

Our first hint is here in Line 5, in which `local_48` is defined by an array of 44 characters.  <br>
![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/1ab2e40f-7046-47ac-bf9d-cd4082faf6fa)  <br> Knowing this, what if we enter more than 44 characters..?

Using this function below, we'll output 48 “a" characters to use when the jeeves program prompts our input. We can later update the number of “a” characters needed.

`python -c "print('a' * 48)" `

Now run `./jeeves` again and copy + paste the 48 “a” characters when it prompts us for our input + press enter.  <br> ![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/50207639-a851-4d47-92bb-2e0d8a90d079)  <br> On the front-end, the behavior still looks similar to the intended output.

Now, let's try 100 “a" characters  <br> ![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/4e01b82d-fc9f-46be-9512-7e83062a6b9f)  <br> We received a segmentation fault error using 100 “a” characters.

The segmentation fault means we were successful in our overflow by overwriting the return address on the stack.
 
However, we overflowed too far! We do not want this as we wanted the overflow to **STOP** on line 8, so it can output us the flag.

![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/bdc04ffd-996b-416d-a334-416a3dcdc416)  <br> ![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/8b4d82a0-e9a9-4da7-9c3c-7a040d5f75a1)

_Note: Although we overflowed the stack using 48-characters, we did not receive a segmentation fault._

Unfortunately, the program did not output the flag as Assembly could not process the answer, so let's continue with another route knowing this. 

But how do we know exactly how many characters it will take before we overflow `local_c`? To achieve this, we'll have to find the exact number of padding bytes!

#### Calculating Padding Bytes
We will need to find out how many padding bytes are needed to reach the correct position to input `{0x1337bab3}` for our flag. The positions are already recorded as hexadecimal, so this requires us to:  

> 1. Know our Starting position and Ending Positions   
> 2. Convert the hexadecimal value to a decimal (bytes)  
> 3. Calculate the difference in bytes between them. 

Since we want to find the byte difference between our desired stop position `local_c` and our buffer overflown input `local_48`, We convert the first and last lines of the stack from Hexadecimal to Decimal, and subtract the difference.

![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/452646da-12ef-4d17-804d-5652f329c5df)  <br> _Note: `local_c` is located at the Stack [-0xc] hexadecimal_

From our results:
> local_c [-0xc] -> 12 bytes
>
> local_48 [-0x48] -> 72 bytes
> 
> 72-12 = 60 bytes needed to reach Line 8 local_c, but not overflow it!

or (if we do not have the internet)

> We hover over our stack to find each length from the stack and add them up:

![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/4460b37a-c85d-4943-a1d0-b47cbc1dcefd)


| Stack | Bytes |
| ------------- | ------------- |
| char (buffer - name)  | 44  |
| int   | 4 |
| void (pointer) | 8 |
| int | 4 |
| Total  | 60 padding bytes |

44 + 4 + 8 + 4 = 60 bytes needed to reach Line 8 local_c, but not overflow it

To reach our flag, the solution will look like this:  <br> 
> [ # of Padding bytes + {0x1337bab3} ]

Our payload will have 60 “a” characters, THEN include the desired `local_c` hexadecimal value `0x1337bab3`. 

### Capture the Flag
From @dannytzoc's video, Let's use pwn-tools and create a script to do this.

![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/3df5d894-e1c0-48d8-8933-6355d974801b)

```
from pwn import *
r = remote ("target machine ip", port)
r.sendline(b'A'*60+p64(0x1337bab3)) 
print(r.recvline())
print(r.recvline())
print(r.recvline())
r.close()
```

The script uses pwn-tools to:

> 1. First, create a Remote SSH tunnel from our localhost machine to target machine using port 30381.
>    
> 2. Send our custom payload of the 60 “a” characters + desired local_c hexadecimal value.
> _Note that we need to include b to notate “bytes” in front of ‘A’ as the script would be trying to concenate dissimilar variables (a string with bytes)._
>          
> 3. Print out the messages once the payload has been received
>    
> 4. Last, close the connection

Note: Replace `target machine ip` and `port` with the information provided by your machine, then Save this script as a python file such as `payload.py`

Lastly, run the script using: `python3 payload.py` and capture your flag!  <br> HTB{*****}
