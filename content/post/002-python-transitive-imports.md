---
title: Fun with Imports in Python
date: 2022-06-26T23:53:17-04:00
slug: python-transitive-imports
toc: true
published: true
categories: ["wat"]
---

## Why does this work?

A few days ago one of my coworkers asked about why this innocent looking code
snippet worked at all:

```python
import some_package
import urllib

response = urllib.request.urlretrieve("https://example.com")
print(response)
```

He posited that it shouldn't, because:

> 1. `urllib.request` is a module
> 2. It hasn't been imported yet
>
> Therefore, you shouldn't be able to call `urllib.request.urlretrieve`.

This makes sense, yet our code "_works_".


Naturally the next thing to try out is to run try it inthe REPL to figure out
what might be happening. Maybe `urllib` was just special? Or this is how Python
always worked but we've been too tunnel-visioned to take note?


```python
>>> import urllib
>>> urllib.request.urlretrieve("https://example.com")
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: module 'urllib' has no attribute 'request
```

<p align="center"><img src="/images/cat_loading.jpg"></p>

But... but the application works?

Just to make sure that I am indeed sane, I run our application on the smallest
sensible input I can find & step through the debugger around the statements of
interest. It worked.

That rules out any weirdness I talked about before.


## How do Python imports even work

Link to [docs][import-system] for the curious.

After skimming through the docs above I hit the following bit:

> To begin the search, Python needs the fully qualified name of the module

We have it: `urllib.request.urlretrieve`

> This name will be used in various phases of the import search,
    and it may be the dotted path to a submodule, e.g. `foo.bar.baz`.
    In this case, Python first tries to import `foo`, then `foo.bar`,
    and finally foo.bar.baz.


So python tries to import `urllib`, and then `urllib.request`.

And finally:

> The first place checked during import search is [`sys.modules`](https://docs.python.org/3.9/library/sys.html#sys.modules).


Lets pull up our troublesome snippet again:

```python
# 1. At this point, the name `urllib` shouldn't exist in `sys.modules`
import some_package
# 2. Ditto
import urllib
# 3. Now `urllib` AND `urllib.request` are both in `sys.modules`

# 4. It works.
response = urllib.request.urlretrieve("https://example.com")
print(response)
```

We've seen before that `import urllib` only brings the `urllib` name in. So
something else was pulling `urllib.request` in. It's gotta be `some_package`
that is messing with `sys.modules`! Maybe something in there is importing
`urllib.request` and we see it transitively?

Let's try out this hypothesis (and you can try this too, reader!).

```sh
echo "import urllib.request" > thing.py
```

Now in the same directory, lets run `python3`

```python
>>> import sys
>>> len(sys.modules)
59
>>> 'urllib' in sys.modules
False
>>> import urllib
>>> len(sys.modules)
60
>>> 'urllib' in sys.modules
True
```
So far, so good. Importing `urllib` puts it in `sys.modules`. Totally reasonable.


```python
>>> # Remove it forcefully so we can see if it gets imported again.
>>> del sys.modules['urllib']
>>> 'urllib' in sys.modules
False
>>> import thing
>>> len(sys.modules)
129
>>> 'urllib' in sys.modules
True
>>> 'urllib.request' in sys.modules
True
>>> urllib.request.urlretrieve("https://example.com")
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: module 'urllib' has no attribute 'request
>>> import urllib
>>> urllib.request.urlretrieve("https://example.com")
('/var/folders/hg/5ssd2ym174j3k9tth123pxpc0000gn/T/tmpoaxj7otp', <http.client.HTTPMessage object at 0x102e97070>
```

<p align="center"><img src="/images/watman.jpg"></p>

Ah-ha! So it was a transitive import that gave us both `urllib` and
`urllib.request`!

Of course we still needed to import `urllib` to use either of them.


I'm not sure if this is a bug or not, but certainly appears to be one because
Python checked for a cache hit in `sys.modules` without verifying if the module
which imported `urllib.request` was the current module.

[import-system]: https://docs.python.org/3.9/reference/import.html#the-import-system
