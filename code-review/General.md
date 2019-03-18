<!--
This is the Markdown source of this document. If you can see this comment, you probably want to view it on GitHub instead unless you intend to edit the document.
See comment in the source of Readme.md for details.
-->

# General comments and concerns

This document contains notes for things that you should either fix or carefully evaluate before competition.

Sections with 🐛 are problems I would classify as bugs.

# Table of Contents

<!--
Automatic updates are disabled on this table of contents because the emojis break the anchors.
It should be updated manually on occasion and have the emojis removed from the links manually.
-->
<!-- TOC updateOnSave:false -->

- [General comments and concerns](#general-comments-and-concerns)
- [Table of Contents](#table-of-contents)
- [`TeleoperatedMode`](#teleoperatedmode)
    - [Add comments to delimit the sections.](#add-comments-to-delimit-the-sections)
    - [🐛 Linear tolerance should be applied to the magnitude](#-linear-tolerance-should-be-applied-to-the-magnitude)
    - [🐛 `ROTATION_SPEED_THRESHOLD` is applied too late.](#-rotation_speed_threshold-is-applied-too-late)
    - [Add teleop override for autonomous driving modes](#add-teleop-override-for-autonomous-driving-modes)
    - [Consider adding a second controller for the peripherals](#consider-adding-a-second-controller-for-the-peripherals)
    - [Add panic orientation reset](#add-panic-orientation-reset)
- [`Drive`](#drive)
    - [Reset motor controllers to factory defaults](#reset-motor-controllers-to-factory-defaults)
    - [Remove `driveManualImplementation`](#remove-drivemanualimplementation)
    - [`lookAt` and `maintainHeading` should set the `driveMode` to `DRIVERCONTROL`](#lookat-and-maintainheading-should-set-the-drivemode-to-drivercontrol)
    - [🐛 `stop` should call `lookAt`](#-stop-should-call-lookat)
    - [🔥🐛🔥 **`init` must not call `resetYaw` every time**](#-init-must-not-call-resetyaw-every-time)

<!-- /TOC -->

# `TeleoperatedMode`

## Add comments to delimit the sections.

`periodic` is a fairly long method. I think it'd make it easier to understand if it were broken up into sections by some comments.

My suggestions would be:

| Start Line | Section Name |
|----|----|
| 37 | Process linear motion controls |
| 52 | Process angular motion controls |
| 61 | Process peripheral controls |

## 🐛 Linear tolerance should be applied to the magnitude

OK, so there's actually two bugs here, but by fixing one you fix the other:

* Tolerance on the left stick is applied per-axis, which makes it impossible to go forward and slightly to the side.
* The tolerance for the left stick and the right stick is the same. The tolerance of the right stick was set very high at some point (not sure why), which makes linear driving at slow speeds impossible.
  * As a bonus bug, this makes it really annoying to go at shallow angles.
  * I have no idea why this doesn't make driving awful, but it definitely isn't making it good.

Here's my recommendation:
* ~~Rename `TOLERANCE` to `RIGHT_STICK_TOLERANCE`.~~ Actually, it should be removed. It isn't necessary as seen in the next section.
  * Additionally, I highly recommend changing it back down to `0.1` unless we can remember why it is set so high, because a 50% deadzone is pretty high.
  * We need to remember we changed it though. If things feel weird at competition, we'll want to change it back.
* Add a new `LEFT_STICK_TOLERANCE` with the value `0.1`
* Delete `getStickLeftX` and `getStickLeftY` (This is an old holdover from 2018 which I think is a old holdover from 2017.)
* Modify the start of `periodic` to look something like this:

```java
double leftX = xboxController.getX(Hand.kLeft);
double leftY = -xboxController.getY(Hand.kLeft);
double leftMagnitude = Math.sqrt(Math.pow(leftX, 2.0) + Math.pow(leftY, 2.0));

if (leftMagnitude < LEFT_STICK_TOLERANCE) {
    leftX = 0.0;
    leftY = 0.0;
} else {
    leftX = Math.pow(leftX, CONTROL_EXPONENT);
    leftY = Math.pow(leftY, CONTROL_EXPONENT);
}

drive.driveManual(leftX, leftY);
```

## 🐛 `ROTATION_SPEED_THRESHOLD` is applied too late.

Right now the logic for deciding whether or not to `maintainHeading` is applied after a power curve is applied to `rotationSpeed`.

This logic is written to try and say "If the rotation speed is less than 30%, maintain heading."

Unfortunately, applying it after the power curve means the actual behavior is "If the rotation speed is less than **67%**, maintain heading."

This means the user has to push the right stick around 2/3 of the way before the robot reacts. It also means it's impossible to rotate slowly.

I think the easiest way to fix this would be to eliminate the first `if..else` and apply the power curve right before `drive.lookAt`. Something like this:

```java
// ...
double rotationSpeed = Math.sqrt(Math.pow(rightX, 2) + Math.pow(rightY, 2));

if(rotationSpeed < ROTATION_SPEED_THRESHOLD) {
    if(!headingIsMaintained) {
        drive.maintainHeading();
        headingIsMaintained = true;
    }
} else {
    rotationSpeed = Math.pow(rotationSpeed, RIGHT_CONTROL_EXPONENT);
    drive.lookAt(angle, rotationSpeed);
    headingIsMaintained = false;
}
```

Additionally, the existing `CONTROL_EXPONENT` should be renamed to `LEFT_STICK_EXPONENT` and a new `RIGHT_STICK_EXPONENT` constant should be added for this. (Keeping it `3.0` is probably fine.)

I also think it wouldn't be a bad idea to change `ROTATION_SPEED_THRESHOLD` to `RIGHT_STICK_THRESHOLD` to bring it in line with `LEFT_STICK_THRESHOLD` added earlier. Do note that the old `THRESHOLD` was eliminated by this change, so if you didn't remove it earlier, now is a good time.

## Add teleop override for autonomous driving modes

This isn't strictly necessary unless we implement autonomous or semi-autonomous (like line alignment. However, it's easy to do and I think it's worth it in the spirit of technical correctness.

`TeleoperatedMode.periodic` should ignore driver controls when `IDrive` reports it isn't in `DRIVERCONTROL` mode, but if the driver pushes the left stick far enough then it should allow overriding the autonomous mode.

I think the easiest place to apply this is in the `LEFT_STICK_TOLERANCE` check added earlier. Using the example code I outlined earlier, it'd looke something like this:

```java
if (leftMagnitude < LEFT_STICK_TOLERANCE) {
    leftX = 0.0;
    leftY = 0.0;

    // If the left stick isn't pushed far enough to drive and the drive base is in an autonomous mode, don't process any driver controls.
    if (drive.getCurrentDriveMode() != DriveMode.DRIVERCONTROL) {
        return;
    }
} else {
    // ...
}
```

Remember that this early return lets you skip the rest of the `periodic` method, so neither the driving nor the peripheral controls will run. In theory we could give them access to peripheral controls, but I can't think of a practical reason to do so.

## Consider adding a second controller for the peripherals

This would allow one driver to focus on driving and the other on everything else.

All that would need to change would be:
1. Adding a new field to `TeleoperatedMode` for player 2's controller.
2. Add a new entry in `PortMap.USB`.
3. Adding the initialization of the second controller to the constructor.
4. Changing the `xboxController` references in `periodic` to the new field for the peripherals.

## Add panic orientation reset

Right now if the robot reboots mid-match or something else terrible, it'll be practically impossible to drive the robot because the field orientation will be incorrect.

We should add an emergency reset for this case.

My suggestion for controls would be a combination of the start and back buttons. They're hard to press on accident (they're the two little ones in the middle), and requiring both means it's basically impossible to do it unintentionally.

To add this:
1. Give `TeleoperatedMode` access to the `IGryoscope`
2. Add `stop` to `IDrive` make `Drive.stop` public. (Make sure to address notes in `Drive`.)
2. Add logic for resetting the gyro to **the very beginning** `periodic`:

```java
// If the driver holds both center buttons, force reset orientation.
boolean player1Panic = player1.getBackButton() && player1.getStartButton();
boolean player2Panic = player2.getBackButton() && player2.getStartButton();
if (player1Panic || player2Panic) {
    Debug.logPeriodic("Panic-resetting orientation!");
    gyro.resetYaw();

    // Stop driving to ensure the robot doesn't use old drive state.
    // (Otherwise it'll try to drive to whatever angle it was last instructed, which would be bad.)
    drive.stop();

    // Skip all controls processing for this periodic update to avoid weirdness if they're pressing the sticks while holding the panic buttons.
    return;
}
```

# `Drive`

## Reset motor controllers to factory defaults

We talked about this but never got around to looking how to do it.

In the constructor for `Drive`, add calls to [configFactoryDefault](https://www.ctr-electronics.com/downloads/api/java/html/classcom_1_1ctre_1_1phoenix_1_1motorcontrol_1_1can_1_1_base_motor_controller.html#a211e713e1b45a4556d7c8b3b9c3935af) for each controller before the `setNeutralMode` block.

## Remove `driveManualImplementation`

This method is an artifact of a quirk in the 2018 `Drive` implementation. (It is to work around the way `Drive.handleActionEnd` is implemented in the 2018 code.)

Copy the method's contents directly into `driveManual` and remove the method. Modify `stop` to call `driveManual` instead.

## `lookAt` and `maintainHeading` should set the `driveMode` to `DRIVERCONTROL`

(Both of these methods could potentially be used in autonomous depending on how autonomous were implemented, but I think it's easy to make it teleop-only.)

You should set the `driveMode` to `DRIVERCONTROL` in both methods. It's not a huge deal, but could lead to incorrect state.

`driveMode = DriveMode.DRIVERCONTROL;`

## 🐛 `stop` should call `lookAt`

Right now `stop` will stop linear driving, but not rotational driving. It should call `lookAt(0.0, 0.0)` to disable rotational driving.

## 🔥🐛🔥 **`init` must not call `resetYaw` every time**

Right now `gyroscope.resetYaw` is called in `init`. This will cause the robot to reset its field orientation each time it changes modes. This means the field orientation will definitely be wrong if the robot is moved during autonomous and/or the sandstorm.

We still need to call it though. I can't find hard documentation or code stating whether or not the NavX automatically zeroes the yaw. I know for sure we can't call it in the `NavXMXP` constructor because the NavX connection takes some time to initialize.

As such, I think the best way to deal with this is to only call `resetYaw` when `init` is called for autonomous.

You can do it with [`DriverStation.isAutonomous`](http://first.wpi.edu/FRC/roborio/release/docs/java/edu/wpi/first/wpilibj/DriverStation.html#isAutonomous()) like this:

```java
if (DriverStation.getInstance().isAutonomous()) {
    gyroscope.resetYaw();
}
```
