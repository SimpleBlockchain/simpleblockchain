# simpleblockchain

Bitcoin and other cryptocurrencies are a huge deal and part of what
makes them possible is the concept of a blockchain. The blockchain
seems magical, but really it's just a data structure with a few
special features. Let's look at how it works.

The first basic concept is that it's just a linked list where each
child points to the parent it came from. (Note that a parent might
have more than one child, but a child can have no more than one
parent.)

    +--------+     +----------+     +----------+
    | link 0 | <---+-* link 1 | <---+-* link 2 | <---...
    +--------+     +----------+     +----------+

Except a blockchain is stronger than this. In a regular linked list,
you can't tell if someone moved the pointers. With a blockchain, you
can.

Let's build up to that feature. First, let's start by changing our
representation of a linked list to be a little more concrete. We want
a data payload in there. Also, instead of arrows, let's use addresses:

    +---------------+     +---------------+     +---------------+
    | parent | addr |     | parent | addr |     | parent | addr |
    |   NA   | 0xa3 |     |  0xa3  | 0x24 |     |  0x24  | 0x1b |
    +---------------+     +---------------+     +---------------+  ...
    |     DATA      |     |     DATA      +     +     DATA      +
    +---------------+     +---------------+     +---------------+

(The addresses are written in hexadecimal to be reminiscent of
addresses in RAM. In an instantiated linked list, the addresses are
usually physical. In our case they are merely logical.)

And here is some Python code to simulate this simple linked list
situation:

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
    l1.show()
    
    l2 = Link(l1.addr, 'Joe pays Amy $7')
    l2.show()
    
    l3 = Link(l2.addr, 'Joe pays Lou $1')
    l3.show()
    ```

(This code is in the file blockchain1.)