---
title: Xcode 12.0 initial git branch name
date: 2020-06-20T10:00:00+01:00
categories:
  - all
tags:
  - xcode-beta
  - xcode-12
  - xcode
---

Xcode 12.0 beta (12A6159) switched initial git branch name: `master` to
`main`. Is there a way how to switch it back or to any other branch name?

<!--more-->

`DVTSourceControl.framework` is responsible for this. The framework is located
in the `Xcode-beta.app` bundle (`Contents/SharedFrameworks`). Call [Hopper](https://www.hopperapp.com) for help :)

Is there a `main` string anywhere?

![Disassembled framework](/images/xcode-12/main-string.png)

We're lucky. It's there and the string is referenced in one place only:

![Disassembled framework](/images/xcode-12/main-string-2.png)

And this place is  the `+[DVTSourceControlBranch initialBranchName]` class method:

![Disassembled framework](/images/xcode-12/initial-branch-name.png)

The method name is pretty clear, but what it does? Is there a string value stored in user defaults (key `DVTSourceControlDefaultNewRepositoryBranchName`)? Yes - use it as an initial branch name. No - fallback to `main`.

Let's test it. Set `voldemort` as a new initial branch name.

```sh
% defaults write com.apple.dt.Xcode.sourcecontrol.Git \
    DVTSourceControlDefaultNewRepositoryBranchName voldemort
```

Create a new Xcode project and keep the _Create Git repository on my Mac_ option enabled.

![](/images/xcode-12/create-git-repo-option.png)

Does it work?

```sh
% git status
On branch voldemort
```

Yep! You can easily switch it back to `master` or any other branch name.
