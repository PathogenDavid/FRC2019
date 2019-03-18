<!--
This is the Markdown source of this document. If you can see this comment, you probably want to view it on GitHub instead unless you intend to edit the document.

This markdown is intended to be rendered using Visual Studio Code, you probably want the following extensions installed:
* AlanWalk.markdown-toc

AlanWalk.markdown-toc has a bug right now. To fix it, add this to your settings.json:
"files.eol": "\r\n",
(Or "\n" if you're not on Windows.)
-->

# FRC 5968 2019 Code Review

This code review consists of a couple different documents:

| Document | Description |
|---|---|
| [General](General.md) | Comments about the code base as a whole and non-autonomous stuff. |
| [Line Alignment](LineAlignment.md) | Things that need to be fixed in order for line alignment to work. |
| [Autonomous](Autonomous.md) | Things that need to be fixed in order for autonomous to work. (Split into `HABLineAuto` and `HatchPanelAuto` tiers.) |
| [Low Priority](LowPriority.md) | General comments that are of lesser concern. |
| [Future](Future.md) | These are thoughts I have on things we should do next time. Don't bother worrying about this until after competition. |
| [My Changes](MyChanges.md) | This is a list of minor changes I made while performing the review. (Mostly just to make it easier for me to read.) |

Additionally, below are some overall notes about the reviews, including an index of types along with their review verdict.

I'm going to assume your brain is slightly fried from orchestra and the fact that I'm doing this review a little late, so sorry if it seems like I'm spoon feeding you a bit. I wanted to err on the side of caution, sorry again for not doing the code review sooner!

# Table of Contents

<!-- TOC -->

- [FRC 5968 2019 Code Review](#frc-5968-2019-code-review)
- [Table of Contents](#table-of-contents)
- [Some General Notes](#some-general-notes)
    - [Risk Management](#risk-management)
    - [Code Removal](#code-removal)
    - [Suggestions, not demands](#suggestions-not-demands)
- [Index of Types](#index-of-types)
    - [Status Descriptors](#status-descriptors)
    - [Index](#index)

<!-- /TOC -->

# Some General Notes

## Risk Management

Code reviews are about risk management. Specifically we're trying to lower the risk of things going wrong at competition. "Wrong" can be a variety of things from the robot not working to the team's performance being impacted by quirks in the way the robot is controlled or missing quality of life features.

This review focuses on both, but primarily the former. I've tried to keep most nitpick comments in the low priority section.

## Code Removal

When I say to remove something: I literally mean straight-up yeet it into the `void`. You can always git it back using version control.

<!-- I cringed slightly at these puns from last year, so I'm doubling down and adding "yeet". Is yeet still cool? -->

Keeping the code around as comments just adds noise. It's OK to comment stuff out haphazardly in the heat of trying to debug something, but keeping it around now just makes it hardder to debug in the heat of things at competition.

After you are done fixing and changing things, I highly recommend doing a final sweep for commented out code and removing stuff or considering if it shouldn't be commented out. (You could also do this as you go along.)

## Suggestions, not demands

As always, all of the comments in all of these documents are suggestions, not demands. If you're confident something I said is wrong or unecessary, feel free to dismiss it.

I tried to keep the `General` document to the most important concerns. As such, if you decide to skip anything in it, I'd stronlgly suggest mentioning it on Discord.

# Index of Types

## Status Descriptors

| Status | Description |
|---|---|
| 🐛 | This file contains bugs. |
| ❗ | This file contains a fragile design or it needs changes to help something else. |
| 😤 | This file isn't the best, but its issues are fairly minor as far as competition is concerned. |
| 👍 | This file looks fine by me (alhough I might still have some minor, pedantic comments.) |
| 💀 | This file is no longer (or never was) necessary and should be deleted. |
| 🙃 | This file is related to autonomous **and** isn't implemented. |
| 🚧 | This file is related to autonomous and I have comments I haven't written yet beyond missing implementation. |

## Index

| | |
|---|---|
| 👍 | AutoMode.java |
| 🙃 | AutonomousMode.java |
| 👍 | CargoGuide.java |
| 👍 | Debug.java |
| 👍 | DisabledMode.java |
| 🐛 | Drive.java |
| 👍 | DriveMode.java |
| 🙃 | FieldInformation.java |
| 👍 | FieldPosition.java |
| 🚧 | HABLineAuto.java |
| 💀 | HardCodedDashboard.java |
| 🚧 | HatchPanelAuto.java |
| 👍 | Hook.java |
| 👍 | ICargoGuide.java |
| 👍 | IDrive.java |
| 🙃 | IEncoder.java |
| 🙃 | IFieldInformation.java |
| 👍 | IGyroscopeSensor.java |
| 👍 | IHook.java |
| 👍 | ILauncher.java |
| 🙃 | ILineDetector.java |
| 👍 | IRobotMode.java |
| 👍 | Launcher.java |
| 👍 | Main.java |
| 👍 | MathUtilities.java |
| 👍 | NavXMXP.java |
| 👍 | NullDrive.java |
| 👍 | PortMap.java |
| 👍 | Robot.java |
| 🐛 | TeleoperatedMode.java |

