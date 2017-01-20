---
title: Using Portier with Python's asyncio
excerpt_separator: <!--more-->
issue_id: 3
---

Let's say you're creating an application that has user accounts. That means you
have some serious decisions to make.

Do you store passwords? Everyone knows [you shouldn't roll your own
crypto][rol], so you should instead turn to a well-regarded
[password-hashing][hsh] library for your programming language or framework. But
even if you do everything right... your database could still be compromised!
How many times can we hear the same story over and over?

Storing passwords is nasty business, but that's why there's [OAuth][oau], right?

<!--more-->

http://django-social-auth.readthedocs.io/en/latest/backends/index.html

[rol]: http://security.stackexchange.com/questions/18197/why-shouldnt-we-roll-our-own
[hsh]: https://en.wikipedia.org/wiki/Cryptographic_hash_function#Password_verification
[oau]: https://en.wikipedia.org/wiki/OAuth
