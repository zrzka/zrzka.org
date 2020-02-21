---
title: Python IDE on the iPad Pro
date: 2017-10-20T00:00:47+01:00
categories:
  - articles
tags:
  - python
  - ipad
  - pythonista
---

My career is kind of colourful. I was working on aircraft systems, VoIP HW & SW, developing Linux
kernel drivers, Linux distributions, OpenOffice.org and many other low level (RTOS) & high level
(desktop applications) stuff. Linux nerd. I said enough one day. I wanted to _just work_ and wasn’t
willing to continue wasting my time with searching how to workaround this and that. Bye bye Linux,
did spend nice 10 years with you. Not saying it was a bad experience. Nope, I learned a lot and
enjoyed it. But …

Fast forward, I’m with Apple for more than 10 years now. Hmm, another 10 years, time to leave? Nope!
Honestly, not excited as I was when I switched, but still pretty happy. Remember the first iPhone?
iPad? Magical devices. Not mentioning first SDK. Exciting times. I did develop lot of applications
for iPhones, iPads and for the desktop. Learned a lot about Objective-C, Objective-C runtime and
all these Apple platforms.

I also bought every iPad Apple did release. But I didn’t know what to do with it. Kind of device
for content consumers. I tried, really hard, to use it as a device where I can create something.
But I always struggled, I always gave up and go back to my Mac. Apparently I was trying to use it
for something it wasn’t designed for. Or it was, but iOS itself wasn’t ready for it.

My needs changed few months ago. I no longer work as a full time developer, but rather trying to
lead people and working on new product specifications. Hard change. Lot of new things to learn.

Do I need to travel with my 15" MacBook Pro for this kind of work? Am I willing to try iPad again?
Yup, I am. Bought the iPad Pro 12.9", external keyboard and pencil few months ago. iOS 11, Files,
drag & drop, wonderful applications like [Workflow](https://workflow.is),
[OmniGraffle](https://www.omnigroup.com/omnigraffle/ios/),
[Editorial](http://omz-software.com/editorial/index.html),
[Ulysses](https://ulyssesapp.com),
[MindNode](https://mindnode.com/mindnode/ios)
, etc. I have to say that the iPad Pro makes sense as a device where I can create stuff finally.
Took quite some time, but it seems it’s already here. Suits my needs perfectly.

What all this has to do with Python IDE on the iPad Pro? I’m no longer working on our products, but
I didn’t left programming. And I never will. It’s quite refreshing to hide myself with all these
devices in some place where no one can disturb me from time to time. We do use
[Amazon Web Services](https://aws.amazon.com) a lot. I’m still kinda involved in scripting, support
tools, … and we do write all these things in Python. Amazon provides nice package named
[boto3](https://boto3.readthedocs.io/en/latest/):

> Boto is the Amazon Web Services (AWS) SDK for Python, which allows Python developers to write
> software that makes use of Amazon services like S3 and EC2. Boto provides an easy to use,
> object-oriented API as well as low-level direct service access.

I wrote lot of tools, scripts, … leveraging _boto3_ power to maintain our AWS services. Also wrote
some [django](https://www.djangoproject.com) applications for internal use. Can I do this on
my iPad Pro? Quite some effort, but yes, I can! And now, you're finally going to learn how.

## Pythonista for iOS

Hats off to [Ole](http://twitter.com/olemoritz) for writing
[Pythonista for iOS](http://omz-software.com/pythonista/index.html). Seriously, tremendous amount
of work. Quote from the Pythonista website:

> Pythonista is a complete development environment for writing Python™ scripts on your iPad or
> iPhone. Lots of examples are included — from games and animations to plotting, image manipulation,
> custom user interfaces, and automation scripts.
>
> In addition to the powerful standard library, Pythonista provides extensive support for interacting
> with native iOS features, like contacts, reminders, photos, location data, and more.

Pythonista supports Python 2.7, 3.5 (beta includes 3.6), standard library, lot of bundled packages
and neat support for native iOS features.

Release cycle of Pythonista is quite long. Version 3.0 was released an year ago, version 3.1 just
a few days ago. This release cycle leads to a situation, where bundled packages can be outdated
and native iOS features doesn’t support latest & greatest stuff — drag & drop, Files.app, etc.

What can be done about this? There’s project named
[StaSh — Shell for Pythonista](https://github.com/ywangd/stash). Do I need shell you say?
Bet you do. StaSh does contain useful commands like _pip_, which allows you to install additional
packages from [PyPI](https://pypi.python.org/pypi). Not all packages are supported. Package must be
_pure Python_, because of the way how binaries, signatures, … work on iOS. It's more complicated,
but I'm not going to explain it here. Just try it and you’ll see if your package will be
installed or not.

Remember the first iPhone OS SDK? First apps? Did you ever think this will be possible?
And we just started.

You should really [buy](https://itunes.apple.com/us/app/pythonista-3/id1085978097?ls=1&mt=8)
Pythonista if you’re into Python.

## Dark side of the moon

You’re excited, you started to love Pythonista, but you slowly do realize that you miss something.
Keyboard shortcuts, features like _Jump to definition_, _Open quickly_ and lot of other things.
Stuff which makes great IDE great.

I started
[filling issues](https://github.com/omz/Pythonista-Issues/issues/created_by/zrzka?page=1&q=is%3Aopen+is%3Aissue+author%3Azrzka)
as a first thing. There’s more than 330 open issues. Hmm, how quickly will Ole solve them? Am
I willing to wait weeks, months, …? Isn’t my use case minor one? Will they ever be fixed? Lot of
questions. I’m a problem solver guy, so, I decided to fix it by myself. That’s how the
[Black Mamba](http://blackmamba.readthedocs.io/) project started.

## Prerequisites

I assume that you’re familiar with following topics:

* [Objective-C Runtime](https://developer.apple.com/documentation/objectivec/objective_c_runtime?language=objc)
   — messaging, signatures and type encoding,
* [Event handling, responders and the responder chain](https://developer.apple.com/documentation/uikit/understanding_event_handling_responders_and_the_responder_chain?language=objc)
  — [UIResponder](https://developer.apple.com/documentation/uikit/uiresponder?language=objc),
  [UIKeyCommand](https://developer.apple.com/documentation/uikit/uikeycommand?language=objc) and
  [UIEvent](https://developer.apple.com/documentation/uikit/uievent?language=objc).

Also I highly recommend to look at [ctypes](https://docs.python.org/3.6/library/ctypes.html)
and [objc_util](http://omz-software.com/pythonista/docs/ios/objc_util.html) in advance.

## Keyboard shortcuts

Pythonista provides small amount of shortcuts and I can’t work without them. Still consider screen
tapping as a slow thing. Let’s add some of them. We have two basic requirements:

* be at the end of the responder chain, just to avoid clashes with Pythonista view controllers shortcuts,
* have them working even if the view controller hierarchy changes over time.

It seems that the `keyCommands` property of the `UIApplication` perfectly suits our needs. When
compared to the `UIViewController`, there’s no `addKeyCommand:` and we have to found another way how
to modify `keyCommands` result. Time to poke Objective-C runtime, especially
[class_addMethod](https://developer.apple.com/documentation/objectivec/1418901-class_addmethod?language=objc)
and
[method_exchangeImplementations](https://developer.apple.com/documentation/objectivec/1418901-class_addmethod?language=objc)
functions. First one allows us to add new methods and second one allows us to exchange their
implementations. It’s called swizzling. We will need to use these functions more than once, so,
here are Python wrappers:

* [add_method](https://github.com/zrzka/blackmamba/blob/v1.3.1/blackmamba/util/runtime.py#L16) — adds
   new method to the class,
* [swizzle](https://github.com/zrzka/blackmamba/blob/v1.3.1/blackmamba/util/runtime.py#L16) - adds
  new method to the class and exchange implementations with the original one.

Ready for swizzling? Let’s write custom `keyCommands` function in Python:

```python
def _blackmamba_keyCommands(_self, _cmd):
    obj = ObjCInstance(_self)
    commands = list(obj.originalkeyCommands() or [])
    commands.extend(_key_commands)
    return ns(commands).ptr
```

Simple one, which gets the original list of key commands and adds our ones from the `_key_commands`
list (module global variable). We do not need to add new method explicitly, our `swizzle` function
does it automatically:

```python
swizzle('UIApplication', 'keyCommands', _blackmamba_keyCommands)
```

What happened? A picture is worth a thousand words:

![Swizzle](/images/python/swizzle.png)

The only remaining thing now is to introduce function, which allows us to register keyboard shortcuts —
[register_key_command](https://github.com/zrzka/blackmamba/blob/v1.3.1/blackmamba/uikit/keyboard.py#L305).
Signature of this function follows.

```python
def _register_key_command(input, modifier_flags, function, title=None)
```

* `input` - it can be any string (like letter `a` for example) or
  [UIKeyInput](https://github.com/zrzka/blackmamba/blob/v1.3.1/blackmamba/uikit/keyboard.py#L194) enum,
  which does contain values for special keys like left arrow.
* `modifier_flags` - it can be `int` (if you know the correct value) or you can use
  [UIKeyModifier](https://github.com/zrzka/blackmamba/blob/v1.3.1/blackmamba/uikit/keyboard.py#L103) enum.
* `function` - Python function, no arguments, to call when user presses the shortcut.
* `title` - optional `str`, discoverability title.

How to print _Hallo_ with `Cmd H`?

```python
from blackmamba.uikit.keyboard import (
   register_key_command, UIKeyModifier
)

def hallo():
   print('Hallo')

register_key_command(
   'h',
   UIKeyModifier.command,
   hallo
)
```

Passed function `hallo` is wrapped with a function, which does meet Objective-C runtime specs.

```python
def key_command_action(_sel, _cmd, sender):
    function()
```

Based on `input` and `modifier_flags`, method name is generated. `blackMambaHandleKeyCommandH:`
in this case.
    </p>

`blackMambaHandleKeyCommandH:` method is added to the `UIApplication` with implementation pointing
to the `key_command_action`. `UIKeyCommand` object is created and stored in the `_key_commands` global list.

![Key Command H diagram](/images/python/key-command-h.png)

And that’s it. We have a way how to register custom keyboard shortcut, assign it to a function written
in Python.

Pretty awesome and I can’t still believe it’s possible.

## Keyboard events

Imagine you have a dialog and you would like to use keyboard shortcuts for it as well. Like arrow
keys to change selection, enter to confirm selection, `Cmd .` to close dialog, etc. Global shortcuts
can be reused for this task, but it will bring another sort of issues. We will be forced to remember
which dialog is active, pass events to it from our shortcut handler, forget the dialog when
it’s closed, … Sounds complicated to me.

Let’s solve this task in a similar way, by swizzling `handleKeyUIEvent:` (`UIApplication`). This
method receives physical keyboard events (`type = 4`) and we can handle them there.

> This is private API, don’t do this in your applications.

Because it’s very similar to global keyboard shortcuts, here’re just links to specific implementations:

* [_blackmamba_handleKeyUIEvent](https://github.com/zrzka/blackmamba/blob/v1.3.1/blackmamba/uikit/keyboard.py#L384-L398)
  — our implementation of `handleKeyUIEvent:`,
* [register_key_event_handler](https://github.com/zrzka/blackmamba/blob/v1.3.1/blackmamba/uikit/keyboard.py#L402-L412)
  — event registration,
* [unregister_key_event_handler](https://github.com/zrzka/blackmamba/blob/v1.3.1/blackmamba/uikit/keyboard.py#L431)
  — event unregistration.

And an example how to
[register](https://github.com/zrzka/blackmamba/blob/v1.3.1/blackmamba/uikit/picker.py#L227-L259)
and
[unregister](https://github.com/zrzka/blackmamba/blob/v1.3.1/blackmamba/uikit/picker.py#L261-L263)
keyboard events.

## Drag & drop

There was no drag & drop support prior to Pythonista (311013, beta). I desperately did want to
use it and wrote another script —
[drag_and_drop.py](https://github.com/zrzka/blackmamba/blob/v1.3.1/blackmamba/script/drag_and_drop.py).

This script demonstrates how to create custom classes, classes that implements protocols like
`UITableViewDragDelegate` and how to use blocks. I think it’s pretty self explanatory except one thing.

There’s experimental support of blocks in Pythonista. You can write a Python function and create block
from it. Here’s an example:

```python
def block_imp(_cmd):
   print('Hallo')

block = ObjCBlock(
   block_imp,
   restype=ctypes.c_void_p,
   argtypes=[ctypes.c_void_p, ctypes.c_void_p]
)
```

If you need to pass block as an argument somewhere, you can just pass `block` variable now and that’s it.
Works. But what if you have to implement block, which receives another block as an argument. There’s
no support for it in Pythonista.

I wrote the
[ObjCBlockPointer](https://github.com/zrzka/blackmamba/blob/v1.3.1/blackmamba/util/runtime.py#L76-L165)
class, which can be initialised with the block pointer. And then you can call it. Here’s
[an example](https://github.com/zrzka/blackmamba/blob/v1.3.1/blackmamba/script/drag_and_drop.py#L67)
how to use this class. Drag session delegate, which is supposed to load data and then call another block
when the operation completes. It’s achieved via `ObjCBlockPointer` class.

## Conclusion

Do you need additional packages like me? Just install
[StaSh](https://github.com/ywangd/stash) and use `pip`. I did install `django`, `boto3` and many other
packages. Do you miss some functionality in Pythonista? Install
[Black Mamba](http://blackmamba.readthedocs.io/).

I like Pythonista for iOS a lot. I do use it almost on a daily basis even when it’s not perfect
(nothing’s perfect) and lacks some features. But because it’s Python IDE, we have an access to the
Objective-C runtime, nothing can stop you from enhancing it on your own. Like I did. See the
[gallery](http://blackmamba.readthedocs.io/en/stable/gallery.html)
of
[script](https://github.com/zrzka/blackmamba/tree/v1.3.1/blackmamba/script)
I wrote.

Can the iPad Pro replace my desktop computer? Yup, it can. There’re still some edge cases where I
rather use my MacBook Pro, but the number of them is very low.

Did you ever think that it will be possible to do these kind of things? In an App Store application,
no jailbreak, … Me not, never, until now. And I can’t still kind of believe it.

Learn or die and remember, **sky is the limit** 🙂

Again, many thanks to [Ole](https://twitter.com/olemoritz) for writing this wonderful piece of
software! Please, [buy](https://itunes.apple.com/us/app/pythonista-3/id1085978097?ls=1&mt=8) it to
support Ole’s ongoing effort to make it better.

## Links

* [Pythonista for iOS](http://omz-software.com/pythonista/)
* [Working Copy for iOS](https://workingcopyapp.com)
* [StaSh](https://github.com/ywangd/stash)
* [Black Mamba documentation](http://blackmamba.readthedocs.io/)
* [Black Mamba repository](https://github.com/zrzka/blackmamba)

## Django notes

If you’d like to run Django application on your iPad Pro, don’t forget to pass following arguments:

```python
dir = os.path.abspath(os.path.join(os.path.dirname(path), '../application'))
sys.path.append(dir)
# import Django execute_from_command_line
arguments = list(sys.argv)
arguments.append('runserver')
arguments.append('--noreload')
arguments.append('--nothreading')
arguments.append('-v')
arguments.append('0')
logging.disable(logging.CRITICAL)
try:
    execute_from_command_line(arguments)
except KeyboardInterrupt:
    logging.disable(logging.NOTSET)
    sys.path.remove(dir)
```

Why these arguments? Django is very talkative and it is constantly filling your console with messages.
Every single time there’s a new message, Pythonista will reveal console, tab with your app becomes
inactive and you have to tap on it to reveal your application. Kind of … That’s the reason for
`-v 0` and disabled logging.

No threading & reloading is here, because Pythonista supports only one Python thread.

Still some limitations, but better than nothing. Otherwise it works perfectly.
