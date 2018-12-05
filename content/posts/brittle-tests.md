---
title: "On Brittle Tests"
date: "2018-12-05"
draft: false
description: "Let's take some time to talk about what makes a test brittle, and go through a real example."
---
If you think about modifying your code as a set of chain reactions, can you make a change without changing other things?  The longer the length of the chain due to a change lets you know that your code is brittle.  Tests are no different in this regard.
<!--more-->

Simply enough, a brittle test is one that breaks easily with even the smallest of code changes.  You may have also heard them called "[Fragile Tests](http://xunitpatterns.com/Fragile Test.html)".  If a slight change in your code always requires you to modify your tests then you have created a brittle test.  Brittle tests lead to test fatigue making developers not want to write tests or even bother fixing them because they'll just break again at the next small change.

Let's take a look at a real example of a brittle test that I just wrote the other day, and then figure out how we can make that same test *not* brittle.

For a bit of backstory I maintain an application that allows our support personnel to modify ACLs for client applications.  These need to be added to our system in valid [CIDR notation](https://wikipedia.org/wiki/Classless_Inter-Domain_Routing).  Sometimes we get invalid CIDRs from our customers.  

So I created a small utility to validate these, but before I created that they would get 3/4 of the way through the process before AWS rejected the invalid CIDR and would end up leaving the creator in a weird state of limbo, where the infrastructure was half-created and had to be manually cleaned up, or destroyed completely and tried to be created again.

```Python
"""
IP address CIDR Validation
"""
from typing import Tuple
import ipaddress
import re


def validate_cidrs(acl_list: str) -> Tuple[list, list]:
    """
    Validates a comma delimited list of CIDRs passed in
    """
    valid_cidrs = []  # type: list
    invalid_cidrs = []  # type: list
    regex = r"^([0-9]{1,3}\.){3}[0-9]{1,3}(\/([0-9]|[1-2][0-9]|3[0-2]))+?$"

    for cidr in acl_list.split(','):
        try:
            in_cidr_notation = re.match(regex, cidr)

            if not in_cidr_notation:
                raise ValueError('Not Valid CIDR Notation')

            ipaddress.ip_network(cidr)
            valid_cidrs.append(cidr)
        except ValueError:
            invalid_cidrs.append(cidr)

    return (valid_cidrs, invalid_cidrs)


def strify_invalid_cidr(cidr: str) -> str:
    """
    Takes an invalid CIDR and returns a string to return to end-user
    """
    ipduh = 'https://ipduh.com/ip/cidr/?{}'.format(cidr)
    return '_{}_ is not a valid CIDR block. {}'.format(
        cidr,
        ipduh
    )
```

See, nothing crazy.  It's pretty simple. Using python's built in `ipaddress` library, we can try to create an `ip_network` object, which will raise a `ValueError` if the network is not a valid IP address.  To ensure that it's in CIDR notation (because we found out that if you leave off the /subnet part at the end it still validates through `ip_network` but not through AWS) we do a relatively simple regular expression check.

For User Experience, I also created a simple function `strify_invalid_cidr` that is meant to just spit out a string that lets a user know one of the subnets that they passed in is not valid, and gives a handy link to a tool that gives a bit more information about what the subnet *might* be.

Testing the first function is pretty straightforward.  We can create a list of known good subnets in CIDR notation, and then a list of known bad ones, and then pass them all into the function and make sure they come out the other end the same:

```Python
"""
IP Address CIDR Validation
"""
import pytest
from utilities import cidr_utils


def test_list_validates():
    with pytest.raises(TypeError):
        cidr_utils.validate_list()

    valid_cidrs = [
        "192.168.0.0/24",
        "3.0.0.0/8",
        "3.5.0.0/16",
    ]
    invalid_cidrs = [
        "192.168.0.4/5",
        "192.168.0.1",
        "192.168.0.10/28",
    ]

    valid, invalid = cidr_utils.validate_list(
        ",".join(valid_cidrs + invalid_cidrs)
    )

    assert valid == valid_cidrs
    assert invalid == invalid_cidrs
```

Again, nothing crazy.  Now let's talk about the second function, because this is the good part.  Testing this function *properly* is hard.  Yes, we can write a unit test that ensures that the text that comes out is the same, but that test will be brittle:

```Python
def test_invalid_cidr_strify():
        cidr = "192.168.0.10/28"
        assert cidr_utils.strify_invalid_cidr(cidr) == "192.168.0.10/20 is not a valid CIDR block. https://ipduh.com/ip/cidr/?192.168.0.10/20"
```

How is that brittle?  What happens when we change any bit of that string?  What if we want to change "CIDR block" to "subnet in CIDR Notation", or product decides that it's better to show something like `192.168.0.10/20 is not valid.  See also: ...`  Brittle in this case means that any small change to the text will make this test fail, which will make you hate this test.  So what can we do to combat this?

Let's find out what's important in this function!  Taking another look, we're looking to see the invalid IP that's being passed in be sent back.  We're not really concerned with what the actual message is, just that some kind of message is being returned the IP address that we send in is contained within that string.

So what does our new test look like?

```Python
def test_invalid_cidr_strify():
        cidr = "192.168.0.10/28"
        assert cidr in cidr_utils.strify_invalid_cidr(cidr)
```

That's much simpler, isn't it?  We've now obtained the same coverage while making our test less brittle.
