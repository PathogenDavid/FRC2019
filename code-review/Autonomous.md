<!--
This is the Markdown source of this document. If you can see this comment, you probably want to view it on GitHub instead unless you intend to edit the document.
See comment in the source of Readme.md for details.
-->

# Changes needed to make autonomous work

This document contains information on making (basic) autonomous work.

<!-- Automatic robots are happy robots. 🤖 -->

As per our discussion, this will focus on making the `HABLineAutonomous` work. However, I think a simplified version of `HatchPanelAuto` is feasible, so the last few sections talk about getting it up and going.

# Table of Contents

<!-- TOC -->

- [Changes needed to make autonomous work](#changes-needed-to-make-autonomous-work)
- [Table of Contents](#table-of-contents)
- [Cleanup](#cleanup)
- [Implement the encoders](#implement-the-encoders)
- [Implement distance driving in `Drive`](#implement-distance-driving-in-drive)
    - [The public interface](#the-public-interface)
    - [The internal logic](#the-internal-logic)
- [Fixing `HABLineAuto`](#fixing-hablineauto)
- [Fixing `HatchPanelAuto`](#fixing-hatchpanelauto)

<!-- /TOC -->

# Cleanup

First off, there's a lot of little infrastructure things left over from last year that aren't super important this year and I don't think it's worth the effort to fix them.

Delete the following types:

* `AutonomousMode`
* `AutoMode`
* `FieldInformation`
* `IFieldInformation`

This will break `Robot`, so let's fix it.

In the past, `AutonomousMode` was responsible for dynamically switching between different variations of autonomous based on the match conditions.

We don't really need this, and since we don't intend to add actual dashboard funcionality for switching we should just hard-code which autonomous mode is used in `Robot`.

In the constructor for `Robot`, there's this line to set the autonomous mode:

```java
autonomousMode = new AutonomousMode(drive, hook);
```

Change this line to initialize `HABLineAuto` instead, then add a commented-out version with `HatchPanelAuto`. (Since we were going to hard-code the field informaiton, we might as well skip the middle-man and hard-code it here.)

```java
autonomousMode = new HABLineAuto(drive);
//autonomousMode = new HatchPanelAuto(drive, hook);
```

# Implement the encoders

Right now there's no implementations of `IEncoder`, and we'll need one for distance driving.

Thankfully, we can just re-use last year's code!

Last year had two different `IEncoder` implementations because we were having issues with something or something. We don't need the alternate implementation, so just copy in [`TalonEncoder`](https://github.com/Cyborg-Indians-5968/FRC2018/blob/b6ba9c1055c3c3d1c355cf0f559a8f839aca39ca/src/org/usfirst/frc/team5968/robot/TalonEncoder.java).

# Implement distance driving in `Drive`

## The public interface

Since we're only driving straight, let's eliminate the implication from the interfact that you can strage with `driveDistance`. A bonus side-effect of this is that we can reuse last year's code for this.

In `IDrive` and `Drive`, remove the `xDirectionSpeed` parameters from both overloads of `driveDistance`, leaving only the `yDirectionSpeed`.

Additionally, you should fix the continuation function overload to match the non-continuation one. (For some reason one of them is `distance, xSpeed, ySpeed` and the other is `xSpeed, ySpeed, distance, continuation`.)

Implement the non-continuation overload to call the continuation one, just like we did with line alignment.

Finally, implement the continuation overload to save the speed, distance, completion routine, set the drive mode, and reset the encoders.

Don't forget to also:

* Add a field to store the distance to travel.
* Add fields for the encoders.
* Add instantiation of the encoders.
  * Look at [last year's code](https://github.com/Cyborg-Indians-5968/FRC2018/blob/b6ba9c1055c3c3d1c355cf0f559a8f839aca39ca/src/org/usfirst/frc/team5968/robot/Drive.java#L51-L58) for reference.
  * Unlike most peripherals, it's OK to create the encoders in `Drive` instead of `Robot` because the encoders are so tightly coupled to the `TalonSRX` motor controllers.

In the end, you should have something like this for the implementation side of things:

```java
@Override
@Override
public void driveDistance(double distanceInches, double xDirectionSpeed) {
    driveDistance(distanceInches, xDirectionSpeed, null);
}

@Override
public void driveDistance(double distanceInches, double xDirectionSpeed, Runnable completionRoutine) {
    setCompletionRoutine(completionRoutine);
    xDirectionSpeed = 0.0;
    yDirectionSpeed = speed;
    driveMode = DriveMode.DRIVINGSTRAIGHT;
    this.distanceInches = distanceInches;
    leftEncoder.reset();
    rightEncoder.reset();
}
```

(Keen eyes will notice that this is very similar to [last year's `driveDistance`](https://github.com/Cyborg-Indians-5968/FRC2018/blob/b6ba9c1055c3c3d1c355cf0f559a8f839aca39ca/src/org/usfirst/frc/team5968/robot/Drive.java#L92-L102) 👀)

## The internal logic

Earlier during line alignment, we added a placeholder for the driving straight logic:

```java
// ...
} else if (driveMode == DriveMode.AUTODRIVINGTRAIGHT) {
        throw new IllegalStateException("Auto-driving straight is not implemented!");
} else // ...
```

It's time to implement this.

The logic will be very similar to [last year's code](https://github.com/Cyborg-Indians-5968/FRC2018/blob/master/src/org/usfirst/frc/team5968/robot/Drive.java#L206-L219) again, but we should change two things:

* Remove the logic for "smart drive straight".
  * This was an experiment that didn't work right, and it wasn't even used in the end last year. (If you look up, you can see it's been disabled.)
  * It's a pseudo-PID thing that is supposed to compensate for motor drift.
* We don't have a `setMotors` anymore. (You'll recall we removed it because it makes less sense with our drive base.) So we need to change how we set motor power.

The implementaiton will look something like this:

```java
// ...
} else if (driveMode == DriveMode.AUTODRIVINGTRAIGHT) {
    leftMotorControllerLead.set(ControlMode.PercentOutput, yDirectionSpeed);
    rightMotorControllerLead.set(ControlMode.PercentOutput, yDirectionSpeed);
    middleMotorControllerLead.set(ControlMode.PercentOutput, 0.0);

    // Check if we've completed our travel
    double averageDistanceTraveled = (leftEncoder.getDistance() + rightEncoder.getDistance()) / 2.0;
    if (averageDistanceTraveled > distanceInches) {
        handleActionEnd();
    }
} else // ...
```

# Fixing `HABLineAuto`

John's heart was in the right place with `HABLineAuto`, but I think you confused him slightly with your initial skeleton you gave him. (To be fair, it looks like you based your skeleton on last year's [`BaselineAuto`](https://github.com/Cyborg-Indians-5968/FRC2018/blob/master/src/org/usfirst/frc/team5968/robot/BaselineAuto.java), which was a little obtuse as well.

The simplest implementation looks like this:

```java
public class HABLineAuto implements IRobotMode {
    private IDrive drive;

    private static final double DRIVE_DISTANCE = 64.0 // inches
    private static final double DRIVE_SPEED = 0.5;

    public HABLineAuto(IDrive drive) {
        this.drive = drive;
    }

    @Override
    public void init() {
        drive.driveDistance(DRIVE_DISTANCE, DRIVE_SPEED);
    }

    @Override
    public void periodic() {
        // Nothing to do.
    }
}
```

If you boil down `BaselineAuto` from last year, you can see it's basically the same thing.

I got 64 inches from John's old `HABLineAuto`, I'm assuming it's right. If you have time, you should double check that it makes sense.

# Fixing `HatchPanelAuto`

So I think a non-rotation `HatchPanelAuto` is feasible. From what I can remember/tell, `HatchPanelAuto`'s intended implementation was going to be:

1. Start with a hatch panel in posession
2. Drive forward until almost at cargo ship.
3. Align to the line.
4. Drive forward to stamp the hatch panel on.
5. Release the hatch panel.
6. Drive backwards so we get points. (You can't be supporting the hatch panel at the end of autonomous or you won't get points.)
7. Drive over to the loading station, align to line, get hatch panel.
8. Drive back to cargo ship, align to line, release hatch panel.

OK, so I glossed over things a bit after step 6. In reality 7 and 8 are really complex.

I think doing up until step 6 is easy enough that we should at least try.

I think even if we implemented the extra logic necessary to make steps 7 and 8 work, it'd be infeasible to get it working during competition. There's just too much to go wrong with measurements and drift and such.

Since `HABLineAuto` was so simple to give you a concrete idea of how to accomplish this, here is an example that looks suspiciously like what you need to do. 😬

```java
public class HatchPanelAuto implements IRobotMode {
    private IDrive drive;
    private IHook hook;

    //TODO: Figure out distances here!
    private static final double APPROACH_DISTANCE = 100.0; // inches
    private static final double DOCK_DISTANCE = 6.0; // inches
    private static final double BACK_OFF_DISTANCE = 24.0; // inches

    // This assumes the robot is places right of the hatch and needs to drive left to find the line.
    private static final double ALIGNMENT_SPEED_AND_DIRECTION = -1.0;

    private static final double APPROACH_SPEED = 0.75;
    private static final double DOCK_SPEED = 0.25;
    private static final double BACK_OFF_SPEED = 0.2; // We want this to be very slow so we know the hook falls down before we back up since we don't have a way to wait 0.5 seconds or something.

    public HatchPanelAuto(IDrive drive, IHook hook) {
        this.drive = drive;
        this.hook = hook;
    }

    @Override
    public void init() {
        drive.driveDistance(APPROACH_DISTANCE, APPROACH_SPEED, () -> andThenAlign());
    }

    @Override
    public void periodic() {
        // Nothing to do.
    }

    private void andThenAlign() {
        drive.driveToLine(ALIGNMENT_SPEED_AND_DIRECTION, () -> andThenDock());
    }

    private void andThenDock() {
        drive.driveDistance(DOCK_DISTANCE, DOCK_SPEED, () -> andThenReleaseAndBackOff());
    }

    private void andThenReleaseAndBackOff() {
        hook.releasePanel();
        drive.driveDistance(BACK_OFF_DISTANCE, BACK_OFF_SPEED);
    }
}
```

Note that I didn't figure out the distances. (John didn't either.) So figuring out reasonable distances based on the field drawings will be an exercise for you or someone you delegate to.
