<!--
This is the Markdown source of this document. If you can see this comment, you probably want to view it on GitHub instead unless you intend to edit the document.
See comment in the source of Readme.md for details.
-->

# Low priority comments

This document contains notes for things that aren't necessarily important for competition. Don't worry about fixing these before competition, but they are worth considering.

However, you don't necessarily have to put these off until after competition (they will likely make emergency debugging easier at competition), just do the `CodeReview` comments first.

# Table of Contents

<!-- TOC -->

- [Low priority comments](#low-priority-comments)
- [Table of Contents](#table-of-contents)
- [`CargoGuide`/`Hook`](#cargoguidehook)
- [`Hook`](#hook)
    - [Extra call to `releasePanel()`](#extra-call-to-releasepanel)
    - [`Compressor` management](#compressor-management)
- [`NavXMXP` Constructor](#navxmxp-constructor)
- [`Drive`](#drive)
- [Really low priority](#really-low-priority)
    - [`DisabledMode`](#disabledmode)
    - [`Launcher`](#launcher)
    - [Files which should not exist.](#files-which-should-not-exist)

<!-- /TOC -->

# `CargoGuide`/`Hook`

The port numbers passed to `DoubleSolenoid` in both of these should be put in `PortMap.java` and referenced instead of being hard-coded.

# `Hook`

## Extra call to `releasePanel()`

The call to `releasePanel()` in the constructor shouldn't be there (we handle that sort of stuff in `init` usually, and already do.)

I don't think it's hurting anything, but it's poor form.

## `Compressor` management

Ideally the `Compressor` should be managed in its own class. Right now `CargoGuide` implicitly relies on `Hook` existing, which doesn't really make sense.

# `NavXMXP` Constructor

The `Debug.log` call in the constructor is left over from debugging and should be removed.

The `SerialPort.Port.kUSB` passed to `AHRS` should be moved to the `PortMap` instead.

# `Drive`

The exception message in `setCompletionRoutine` is wrong. It should read something like `Tried to perform an autonomous action while one was already in progress!`

# Really low priority

## `DisabledMode`

This class shouldn't have references to `Hook` or `Launcher`, they should be removed.

## `Launcher`

I'd suggest renaming `HIGH` and `LOW` to `ON_SPEED` and `OFF_SPEED` respectively.

`CONVEYER_MOTOR_CONTROLLER`: Convey**o**r is spelled wrong.

I'd suggest renaming this class and the related interface to `Conveyor` instead. This brings it in line with the terminology the rest of the team uses and makes more sense given its function.

## Files which should not exist.

Somehow the `build` folder at the root got into the repository despite being in the `.gitignore`. It should be deleted.
