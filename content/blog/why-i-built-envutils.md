+++
title = "Why I Built EnvUtils"
date = "2023-08-07T21:35:24+01:00"
draft = false
description = "I missed inline env overrides in windows so I built powershell module: EnvUtils"

tags = []
+++

TL;DR You can find EnvUtils module [on Github](https://github.com/lpedrosa/EnvUtils).

Before experimenting with using Windows as a daily driver for work, I used to use Ubuntu.[^1]

One thing I like about the Unix shell is that you can prefix a command with an environment variable,
and the environment value will _only be overridden for that single command_.

```sh
# assuming FAVOURITE_FRUIT was not set before
FAVOURITE_FRUIT=kiwi python -c 'import os; print(os.getenv("FAVOURITE_FRUIT))'
kiwi
```

You might have come across this, if you have used the [express](https://expressjs.com/) web framework
for example, and you wanted to set `NODE_ENV` to `production` so you could check if your production
settings are correct.

The cool thing about this feature is that it will restore the variable to whatever value it had
before you ran the command.

```sh
# let's set this to a value
export FAVOURITE_FRUIT=apple

FAVOURITE_FRUIT=kiwi python -c 'import os; print(os.getenv("FAVOURITE_FRUIT))'
kiwi

# the value should be restored
echo $FAVOURITE_FRUIT
apple
```

## What about Windows?

While you cannot do the same thing on Windows, here is an attempt using `cmd`:

```bat
set FAVOURITE_FRUIT=kiwi & python -c "import os; print(os.getenv('FAVOURITE_FRUIT'))" & set FAVOURITE_FRUIT=
kiwi
```

And another using PowerShell:

```powershell
$env:FAVOURITE_FRUIT='kiwi'; python -c 'import os; print(os.getenv("FAVOURITE_FRUIT"))'; $env:FAVOURITE_FRUIT=''
```

In both cases, we get a similar effect to the Unix example, but it is important to note that we
_lose the ability to restore the variable to its previous value_.

While you can modify the examples to make that work, they will no longer be _easy to type as a
one-liner_ or even _easy to read_.

## Exploring a solution using PowerShell

I was already experimenting with using PowerShell as my main shell, while occasionally using
[WSL](https://en.wikipedia.org/wiki/Windows_Subsystem_for_Linux) as an escape hatch, for when I
couldn't be bothered to learn the proper way to do something.[^2]

Looking at the PowerShell docs, there was a built in command that looked similar to what I wanted to
achieve: `Invoke-Command`.

This cmdlet allowed you to run commands on a local or remote computer. What made me particularly
interested in it, was the the `-ScriptBlock` parameter:

```powershell
Invoke-Command -ScriptBlock { <# write your command here #> }
```

What if I could write something similar, that took the environment overrides as a parameter, much
like our original example.

## Building my own cmdlet

After learning enough PowerShell to become dangerous, I came up with the following:

```powershell
# let's give it a value, to prove it restores the environment
$env:FAVOURITE_FRUIT = "apple"

# run the one-liner
Invoke-Environment -Environment @{ FAVOURITE_FRUIT = "kiwi" } { python -c "import os; print(os.getenv('FAVOURITE_FRUIT'))" }
kiwi

# check that the value was restored
$env:FAVOURITE_FRUIT
apple
```

`Invoke-Environment` covers all my use cases:

- short enough to be easy to type
- easy to read and to remember what it does
- keeps the environment intact between calls

## Making it work with `dotenv` files

After using `Invoke-Environment` a couple of times, I wanted to keep a record of all the environment
overrides in a file, so that I could also use it on docker. For that, I had to support `dotenv` files.

Here is the typical usage of a `dotenv` file with docker:

```sh
# filename: .env
LOG_LEVEL=debug
TARGET_URL=https://example.com

# you can inject the above dotenv file into a container
docker run --env-file .env <image>
```

And with `Invoke-Environment`:

```powershell
Invoke-Environment -EnvironmentFile .env { <# write your command here #> }
```

Supporting the `dotenv` format meant writing a basic parser to convert the file into a PowerShell
Hash Table. This was such a useful feature that I exposed it as a cmdlet:

```powershell
$Environment = ConvertFrom-Environment .env
$Environment["LOG_LEVEL"]
debug
```

## Make the environment last between commands

One last common pattern that I wanted to be able to do was the ability to source a `dotenv` file for
a given shell session, so I could use the overrides between command invocations. I still wanted to
keep the `dotenv` file format, and avoid pre-appending `set` to each line just so I could run
`call .env`.

That being said, I added the following commands to support this workflow: `New-Environment`,
`Get-Environment`, and `Remove-Environment`.

Here is an example of a workflow using these cmdlets:

```powershell
# load the env into the shell session
New-Environment -EnvironmentFile .env

# run your commands
# command-1 arg1 agr2
# command-2 arg1

# you can view your session overrides at any point
Get-Environment

# you can restore the environment to the previous state
# by closing your shell or by running
Remove-Environment
```

## Packaging it as a PowerShell module

You can find all the above cmdlets packaged as a module on [Github](https://github.com/lpedrosa/EnvUtils).

Clone the project into a folder in your `env:$PSModulePath` to start using it, and [let me know](https://github.com/lpedrosa/EnvUtils/issues)
if you find any bugs.

I have kept the _destructive_ commands well behaved e.g by supporting `-WhatIf` and `-Confirm` switches.

A PowerShell gallery will soon follow (possibly with a `1.0.0` release).

[^1]: Started as an experiment that never stopped. I will write more about this soon.
[^2]: I rarely reach for WSL these days, unless I want to test something on an unix shell and I
    don't want to start a container, or there's something in `coreutils` that I really need.
