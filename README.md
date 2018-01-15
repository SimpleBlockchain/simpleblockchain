# simpleblockchain

## Linked Lists

Bitcoin and other cryptocurrencies are a huge deal and part of what
makes them possible is the concept of a blockchain. The blockchain
seems magical, but really it's just a data structure with a few
special features. Let's look at how it works.

The first basic concept to understand is a linked list where each
child points to the parent it came from. (Note that a parent might
have more than one child, but a child can have no more than one
parent.)

    +--------+     +----------+     +----------+
    | link 0 | <---+-* link 1 | <---+-* link 2 | <---...
    +--------+     +----------+     +----------+

A blockchain is a stronger form of a linked list. In a regular linked
list, you can't tell if someone moved the pointers. With a blockchain,
you can.

Let's build up to that feature. We'll start by changing our
representation of a linked list to be a little more concrete. We want
a data payload in there. Also, instead of arrows, let's use addresses:

    +---------------+     +---------------+     +---------------+
    | parent | addr |     | parent | addr |     | parent | addr |
    |   NA   | 0xa3 |     |  0xa3  | 0x24 |     |  0x24  | 0x1b |
    +---------------+     +---------------+     +---------------+  ...
    |     DATA      |     |     DATA      +     +     DATA      +
    +---------------+     +---------------+     +---------------+

Note that the "parent" of each node (except the first, which has no
parent) is the address value of the parent node.

The addresses are written in hexadecimal to be reminiscent of
addresses in RAM. In an instantiated linked list in a real running
program, the addresses are usually physical--they point to a
particular memory location. In our case they are merely logical. Look
at a link and read the parent address. Then find the node that has
that address.

Here is some Python code to simulate this simple linked list
situation, with some financial transactions as the data payload:

```python
import random

class Link():
    def __init__(self, parent, data):
        self.addr = random.randint(0, 256)
        self.parent = parent
        self.data = data

    def show(self):
        print('+---------------+')
        print('| parent | addr |')
        if self.parent:
            p = hex(self.parent)
        else:
            p = 'None'
        print('| %s   | %s |' % (p, hex(self.addr)))
        print('+---------------+')
        print('|%-15s|' % (self.data))
        print('+---------------+')
        
l1 = Link(None, 'Amy pays Joe $5')
l2 = Link(l1.addr, 'Joe pays Amy $7')
l3 = Link(l2.addr, 'Joe pays Lou $1')

for l in [l1, l2, l3]:
    l.show()
```

This implementation is fine for storing data among people you
trust. But if it's editable by the public, you have a security
problem. Anyone can alter transaction data or insert/remove items at
any time by picking some link L, pointing to L and having the child of
L point to your fake transaction:


               +---------------+ 
               | parent | addr | 
               |  0xa3  | 0xfa | 
               +---------------+ 
               |    HACKER     | 
               +---------------+ 

    +---------------+     +---------------+     +---------------+
    | parent | addr |     | parent | addr |     | parent | addr |
    |   NA   | 0xa3 |     |  0xfa  | 0x24 |     |  0x24  | 0x1b |
    +---------------+     +---------------+     +---------------+  ...
    |     DATA      |     |     DATA      +     +     DATA      +
    +---------------+     +---------------+     +---------------+

This can be accomplished with some code like this:

```python
lhacker = Link(l1.addr, 'Amy -> Joe $1e6')
l2.parent = lhacker.addr

for l in [l1, lhacker, l2, l3]:
    l.show()
```

All of the above code is in the file `blockchain1`.

## Security Measure #1

We're going to take two security measures that will work together to
ensure no one can edit the blockchain. The first security measure is
changing how the addresses work.

Right now, the address is basically a random number, which represents
some location in RAM or meatspace. This address is something *about*
the link, but is not a fundamental propery *of* the link. We're going
to change that. Instead of a location, we're going to make the address
a hash of the link itself.

(If you don't know what a hash is, it's basically a one-way
function. You hand the hash function a bunch of data and it hands you
back a unique fingerprint. There's no way to recover the data from the
fingerprint. Also, minor tweaks to the data result in big changes in
the fingerprint, so you can't "fish around" by subtly modifying the
data until you get close to the right hash.)

It's easy to alter our existing linked list code to use addresses that
are hashes of the fundamental properties of the link itself:

```python
import hashlib

class Link():
    def __init__(self, parent, data):
        self.parent = parent
        self.data = data
        self.addr = hashlib.md5(str(self.parent) + str(self.data)).hexdigest()

    def show(self):
        print('+---------------------------------------------------------------------+')
        print('| parent                           | addr                             |')
        if self.parent:
            p = self.parent
        else:
            p = 'None'
        print('| %-32s | %s |' % (p, self.addr))
        print('+---------------------------------------------------------------------+')
        print('|%-69s|' % (self.data))
        print('+---------------------------------------------------------------------+')
        
l1 = Link(None, 'Amy pays Joe $5')
l2 = Link(l1.addr, 'Joe pays Amy $7')
l3 = Link(l2.addr, 'Joe pays Lou $1')

for l in [l1, l2, l3]:
    l.show()
```

The links have gotten wider, so let's stack them instead of lining
them up horizontally. When I ran this, I got these values:

    +---------------------------------------------------------------------+
    | parent                           | addr                             |
    | None                             | 967436fb856f5bd2684310f5fae41773 |
    +---------------------------------------------------------------------+
    |Amy pays Joe $5                                                      |
    +---------------------------------------------------------------------+

	+---------------------------------------------------------------------+
    | parent                           | addr                             |
    | 967436fb856f5bd2684310f5fae41773 | e9b48dab1b3aac47e943eedd67f87d13 |
    +---------------------------------------------------------------------+
    |Joe pays Amy $7                                                      |
    +---------------------------------------------------------------------+

	+---------------------------------------------------------------------+
    | parent                           | addr                             |
    | e9b48dab1b3aac47e943eedd67f87d13 | a46d8ae4f32badac6f9280328e0fb954 |
    +---------------------------------------------------------------------+
    |Joe pays Lou $1                                                      |
    +---------------------------------------------------------------------+

It still works exactly like a linked list--the "parent" of the second
link still points to the address of the first link. But that first
address is now something fundamental to what that first link
*is*. When I rerun the above code, I no longer get random addresses, I
get **the exact same** addresses. That's because the address is
computed from the link itself, which hasn't changed.

How is this any more secure? Look above at the simple linked list
hacking case. In order to insert a link, I had to change the "parent"
attribute of a link. That was a simple edit in that case, but now if
you change a link's attributes, **you also change the link's
address**.

When someone changes anything about a link, that forces the address to
change. That means that the child of that link has to change *it's*
parent address. Which means that child's address also changes, so the
grandchild must also change. Any change anywhere in the blockchain
must ripple all the way to the end.

How can we do that in code? It's still pretty simple:

```python
lhacker = Link(l1.addr, 'Amy -> Joe $1e6')
l2.parent = lhacker.addr
l3.parent = l2.addr

for l in [l1, lhacker, l2, l3]:
    l.show()
```

Only one additional link existed in this demo blockchain, but by the
argument above you can see that we might have to alter hundreds,
thousands, millions or billions of links in the chain. *All* of them,
from the point of alteration to the very end.

All of the code from this section is in `blockchain2`.

## Security Measure #2

Having to alter all subsequent history is a pain, but it's not
impossible. That's why we introduce security measure #2:
proof-of-work. In normal Bitcoin operation, this is also called
"mining", so lets call it that. Anyone who wants to generate an
address for a real blockchain link needs to do this mining step for
each link.

The mining works a lot like the lottery. With the lottery, any
particular person has a very, very low chance of winning, but the
chance of *someone* winning is very high. That's why blocks are
actually created, even though it's very unlikely a particular hacker
will be able to do it enough times to corrupt the entire chain.

The hacker has a problem. If s/he alters a single block in history,
all subsequent blocks need their addresses recomputed. But in order to
do that recomputation, the "mining" needs to happen. Succeeding at any
given mining step is phenomenally unlikely, so succeeding at
dozens/hundreds/thousands/millions is basically impossible. This is
why Bitcoin says that once your transaction is some number of blocks
deep in the chain, it's a done deal. Re-mining that number of blocks
is beyond hacker's ability.

How does the blockchain make mining difficult? By forcing the hashed
address of the link fall below a given value. Remember that the hash
function turns a pile of data into a number with a given number of
digits. There's no way to look at the data and predict what that
number is without actually computing it. And if you change the data
slightly, you get a completely different number.

Let's say the hash produces a 3 digit decimal number, 000 to 999. To
deliberately make mining difficult, we could require that the miner
gets a hash address of 000 to 099. So here's what the miner does when
trying to add a link to the blockchain:

    1. hash the link
	2. if the value of the hash is not 000 to 099
    3.   alter the link slightly so we get a different hash
    4.   go to 1
	5. success

That's the pseudocode, here's the real code:

```python
import hashlib

class Link():
    def __init__(self, parent, data, difficulty_level):
        self.parent = parent
        self.data = data

        nonce = 0
        addr = hashlib.md5(str(self.parent) + str(self.data) + str(nonce)).hexdigest()
        while int(addr,16) > difficulty_level:
            nonce += 1
            addr = hashlib.md5(str(self.parent) + str(self.data) + str(nonce)).hexdigest()

        self.nonce = nonce
        self.addr = addr

    def show(self):
        print('+---------------------------------------------------------------------+')
        print('| parent                           | addr                             |')
        if self.parent:
            p = self.parent
        else:
            p = 'None'
        print('| %-32s | %s |' % (p, self.addr))
        print('+---------------------------------------------------------------------+')
        print('| nonce                                                               |')
        print('| %-67d |' % (self.nonce))
        print('+---------------------------------------------------------------------+')
        print('|%-69s|' % (self.data))
        print('+---------------------------------------------------------------------+')

easy = 0xa0000000000000000000000000000000
hard = 0x0a000000000000000000000000000000
veryhard = 0x000000000000000a0000000000000000
difficulty_level = easy

l1 = Link(None, 'Amy pays Joe $5', difficulty_level)
l2 = Link(l1.addr, 'Joe pays Amy $7', difficulty_level)
l3 = Link(l2.addr, 'Joe pays Lou $1', difficulty_level)

for l in [l1, l2, l3]:
    l.show()
```

The "difficulty level" is exactly analogous to the "000 to 099" range
we specified in our example above. The "nonce" is how to "alter the
link slightly" to get a different hash value. We just keep altering
and hashing, altering and hashing until we happen to get a value below
the difficulty level. The lower that level is, the harder it is to
succeed.

This code is in file `blockchain3`. Try running it with different
values for the difficulty level. Watch your CPU usage at the same
time. Now you know why it's called "proof of work".

It's hard to demo the hacker portion of this. For bitcoin, you have
millions of people running the mining code. The difficulty level is
set such that in that huge group of people, there's one success once
every 10 minutes or so. The difficulty level is set lower and lower
over time to keep this rate approximate constant.

But for the demo, there's only one miner. That means the difficulty
level has to be very easy or we'll never see a success. But that makes
things too easy for the "hacker" when that portion of the code
runs. To simulate this, let's change the difficulty level between the
"real" and "hacked" sections of code.

```python
difficulty_level = veryhard
lhacker = Link(l1.addr, 'Amy -> Joe $1e6', difficulty_level)
l2.parent = lhacker.addr
l3.parent = l2.addr

for l in [l1, lhacker, l2, l3]:
    l.show()
```

On my machine, the hacker code basically just hangs. It can't
recompute this link in the chain in a reasonable time even once, let
alone twice or more. The blockchain is effectively safe.

## Omitted Stuff

There's a lot more to Bitcoin than I've presented here, which I won't
even mention.

There are also omitted issues that are more directly
blockchain-related. For instance, how does anyone know you actually
did the work to compute those hashes? Because they can check by
running the hash themselves. (Checking if the answer is right takes
only a single try, so that's fast.)

Or what happens if by chance someone does succeed in altering a chain
near the end, where there are fewer blocks to have to recompute? A
large hacker group could coordinate on this. Yes, they could, but keep
in mind that they have have a sigificant fraction of the world's
computing resources in order to keep up. There are also mechanisms
built into Bitcoin to help choose between alternate blockchains.

And what about the hash, am I cheating by using Python's `hashlib` for
that? Not really. A hash can be anything with the properties loosely
described of "one way" and "non-linear". I used `hashlib.md5` because
I used a similar library in my original code in another language that
had fewer choices.

