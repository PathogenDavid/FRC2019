<!--
This is the Markdown source of this document. If you can see this comment, you probably want to view it on GitHub instead unless you intend to edit the document.
See comment in the source of Readme.md for details.
-->

# Changes needed to make line alignment work

This document outlines the changes that are necessary to make line alignment work.

# Table of Contents

<!-- TOC -->

- [Changes needed to make line alignment work](#changes-needed-to-make-line-alignment-work)
- [Table of Contents](#table-of-contents)
- [Implementing `ILineDetector`](#implementing-ilinedetector)
- [Adding helper for line detector calibration](#adding-helper-for-line-detector-calibration)
- [Implement `Drive.handleActionEnd`](#implement-drivehandleactionend)
- [Add methods to `IDrive` for starting the line alignment](#add-methods-to-idrive-for-starting-the-line-alignment)
- [Implement strafing](#implement-strafing)
- [Use line detection in teleop](#use-line-detection-in-teleop)

<!-- /TOC -->

# Implementing `ILineDetector`

You'll recall from when we were testing with an Arduino, our line detector is an analog sensor. In WPILib, analog sensors are read using [`AnalogInput`](http://first.wpi.edu/FRC/roborio/release/docs/java/edu/wpi/first/wpilibj/AnalogInput.html).

To implement `AnalogLineDetector`, you'll need the following:

1. [Instantiate](http://first.wpi.edu/FRC/roborio/release/docs/java/edu/wpi/first/wpilibj/AnalogInput.html#%3Cinit%3E%28int%29) an instance of `AnalogInput` with a port from the port map. (Port 0 is fine.)
2. Add a new method to `ILineDetector` and your implementation with the signature `double getRawValue()`
    * This will be used to implement `isOnLine` and aid with calibration.
    * The implementation of this method should return the voltage reported by the sensor.
    * There's multiple ways to get the voltage from the `AnalogInput`. I think [`getVoltage`](http://first.wpi.edu/FRC/roborio/release/docs/java/edu/wpi/first/wpilibj/AnalogInput.html#getVoltage%28%29) should be good enough.
3. Add a `double` constant for the line threshold. (The range is going to 0..5, so 2.5 is fine to start. We'll calibrate it on practice day.)
4. Add an implementation for `isOnLine` that returns true if the sensor reports a voltage (as reported by `getRawValue`) above the threshold.

Finally, make sure `Robot` is modified to store/instantiate a new instance of this line detector.

# Adding helper for line detector calibration

We need a way to calibrate the line detector before competition.

I think the easiest way to accomplish this is simply to have the robot print out sensor values in disabled mode.

Modify `DisabledMode` to take/store a reference to `ILineDetector` and modify `DisabeldMode.periodic` to call `Debug.logPeriodic` to report the values of `ILineDetector.getRawValue` and `isOnLine`.

# Implement `Drive.handleActionEnd`

This private method was in the 2018 code, but didn't make it when you copied things over for the initial 2019 code skeleton.

Additionally, this method was impacted by the changes we made to `Drive.stop` earlier. Here's a suitable new implementation:

```java
private void handleActionEnd() {
    // Save currentCompletionRoutine before calling stop because it'll clear it.
    // (We can't dispatch now because then the call to stop would wipe out any state set by the completion routine.)
    Runnable oldCompletionRoutine = currentCompletionRoutine;

    // Stop all drive movement.
    stop();
    
    // Dispatch the completion routine if there was one configured.
    if (oldCompletionRoutine != null) {
        currentCompletionRoutine = null;
        oldCompletionRoutine.run();
    }
}
```

# Add methods to `IDrive` for starting the line alignment

We need methods on `IDrive` (and `Drive`) to allow interested parties to initiate line alignment.

I think the easiest way to implement this will be to allow specifying a strafing speed within `-1.0..1.0`. (Negative looks for lines to the left, positive looks for lines to the right.)

You need an overload for with and without a continuation function. For example:

```java
public void driveToLine(double strafeSpeed, Runnable completionRoutine);

public void driveToLine(double strafeSpeed);
```

The implementation without the continuation function is implemented simply as calling the one with the contiuation function:

```java
public void driveToLine(double strafeSpeed) {
    driveToLine(strafeSpeed, null);
}
```

The implementation simply needs to save the continuation function, store the strafing speed, and change the drive mode. You could either add a new field for the speed or reuse an existing one. Either is fine:

```java
public void driveToLine(double strafeSpeed, Runnable completionRoutine) {
    setCompletionRoutine(completionRoutine);
    xDirectionSpeed = strafeSpeed;
    driveMode = DriveMode.LINEALIGNMENT;
}
```

# Implement strafing

To implement strafing, we need to modify `Drive.periodic` to consider the drive mode (it currently doesn't) and to implement the actual strafing behavior.

First, make sure `Drive` has access to an instance of the `ILineDetector` sensor.

Last year the code was very simple in `Drive.periodic`. This year is more complex, so I think breaking things up is warranted.

Rename `Drive.periodic` to `Drive.manualControlPeriodic` and make it private. (Don't forget to remove the `@Override` designation too.)

Add a new `Drive.periodic` that considers the drive modes. (For the `DRIVERCONTROL` drive mode, it should call the method you just renamed.) It will look something like this:

```java
@Override
public void periodic() {
    if (driveMode == DriveMode.DRIVERCONTROL) {
        manualControlPeriodic();
    } else if (driveMode == DriveMode.AUTODRIVINGTRAIGHT) {
        throw new IllegalStateException("Auto-driving straight is not implemented!");
    } else if (driveMode == DriveMode.AUTOROTATING) {
        throw new IllegalStateException("Auto-driving rotation is not implemented!");
    } else if (driveMode == DriveMode.LINEALIGNMENT) {
        leftMotorControllerLead.set(ControlMode.PercentOutput, 0.0);
        rightMotorControllerLead.set(ControlMode.PercentOutput, 0.0);
        middleMotorControllerLead.set(ControlMode.PercentOutput, xDirectionSpeed);

        if (lineDetector.isOnLine()) {
            handleActionEnd();
        }
    } else {
        throw new IllegalStateException("The drive base controller is in an invalid drive mode.");
    }
}
```

# Use line detection in teleop

To initiate line detection from teleoperated, we simply need to add new controls for the funcionality.

Unfortunately we're running out of controls if we're stick to one controller. I think the D-pad would work well for this functionality. (That's the little plus shapped button at the left-bottom. It's actually four buttons.)

The D-pad is a little obtuse to use in WPILib. To access it, you use the [getPOV](http://first.wpi.edu/FRC/roborio/release/docs/java/edu/wpi/first/wpilibj/GenericHID.html#getPOV%28%29) method:

```java
final int RIGHT = 90;
final int LEFT = 270;

if (xboxController.getPOV() == LEFT) {
    drive.driveToLine(-LINE_ALIGNMENT_SPEED);
} else if (xboxController.getPOV() == RIGHT) {
    drive.driveToLine(LINE_ALIGNMENT_SPEED);
}
```

Make sure to define a constant for the line alignment speed. I'd suggest something low to start, like `0.5`.
