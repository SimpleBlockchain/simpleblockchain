# simpleblockchain

## Linked Lists

Bitcoin and other cryptocurrencies are a huge deal and part of what
makes them possible is the concept of a blockchain. The blockchain
seems magical, but really it's just a data structure with a few
special features. Let's look at how it works.

The first basic concept is just a linked list where each child points
to the parent it came from. (Note that a parent might have more than
one child, but a child can have no more than one parent.)

    +--------+     +----------+     +----------+
    | link 0 | <---+-* link 1 | <---+-* link 2 | <---...
    +--------+     +----------+     +----------+

A blockchain is a stronger form of a linked list. In a regular linked
list, you can't tell if someone moved the pointers. With a blockchain,
you can.

Let's build up to that feature. First, let's start by changing our
representation of a linked list to be a little more concrete. We want
a data payload in there. Also, instead of arrows, let's use addresses:

    +---------------+     +---------------+     +---------------+
    | parent | addr |     | parent | addr |     | parent | addr |
    |   NA   | 0xa3 |     |  0xa3  | 0x24 |     |  0x24  | 0x1b |
    +---------------+     +---------------+     +---------------+  ...
    |     DATA      |     |     DATA      +     +     DATA      +
    +---------------+     +---------------+     +---------------+

The addresses are written in hexadecimal to be reminiscent of
addresses in RAM. In an instantiated linked list, the addresses are
usually physical--they point to a particular memory location. In our
case they are merely logical. Look at a link and read the parent
address. Then find the node that has that address.

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
```

(All of the above code is in the file blockchain1.)

## Security Measure #1

We're going to take two security measures that will work together to
ensure no one can edit the block chain. The first security measure is
changing how the addresses work.

Right now, the address is basically a random number, which represents
some location in RAM or meatspace. This address is something *about*
the link, but is not a fundamental propery *of* the link. We're going
to change that. Instead of a location, we're going to make the address
a hash of the link itself.

(If you don't know what a hash is, it's basically a one-way
function. You hand the hash function a bunch of data and it hands you
back a unique fingerprint. There's no way to recover the data from the
fingerprint. Also, minor tweaks to the data result in huge leaps in
the fingerprint.)

It's easy to alter our existing linked list code to work like this:

```python
import hashlib

class Link():
    def __init__(self, parent, data):
        self.parent = parent
        self.data = data

    def addr(self):
        return hashlib.md5(str(self.parent) + str(self.data)).hexdigest()

    def show(self):
        print('+---------------------------------------------------------------------+')
        print('| parent                           | addr                             |')
        if self.parent:
            p = self.parent
        else:
            p = 'None'
        print('| %-32s | %s |' % (p, self.addr()))
        print('+---------------------------------------------------------------------+')
        print('|%-69s|' % (self.data))
        print('+---------------------------------------------------------------------+')
        
l1 = Link(None, 'Amy pays Joe $5')
l2 = Link(l1.addr(), 'Joe pays Amy $7')
l3 = Link(l2.addr(), 'Joe pays Lou $1')

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
address is now some fundamental to what that first link *is*. When I
rerun the above code, I no longer get random addresses, I get **the
exact same** addresses. That's because it's computed from the link
itself, which hasn't changed.

