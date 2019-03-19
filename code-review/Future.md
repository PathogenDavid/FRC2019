<!--
This is the Markdown source of this document. If you can see this comment, you probably want to view it on GitHub instead unless you intend to edit the document.
See comment in the source of Readme.md for details.
-->

# Ideas for the future

This file contains miscellaneous thoughts on things we should focus on next time. Don't bother with them until after competition!

* We should have a base interface for peripherals and a manager for them so we don't need to worry about `doPeripheralReinitialization`/`doPeripheralPeriodicProcessing`.
* Can we formalize things like the motor test somehow? It'd be nice to be able to use them in test mode more easily.
  * The main issue with them is that peripherals will have locks on things like motor controllers, and these tests tend to benefit from going to them directly.
  * Maybe we need a more generic Robot class that is switched out entirely?
  * How do you even relinquish a motor controller? I assume there must be some `IDisposable` equivalent in Java.
* We should make our own debug log infrastructure. After the fiasco that was the Riolog's unreliability, I don't trust it anymore. We need something for debugging temporal data.
  * Bonus idea: We should have a remote ImGui for diagnostics. Or something similar that's more minimal than the dashboards most teams deal with.
    * That'd be wild, but doing it in Java would be a pain.
* We should call `AutoMode` something like `AutonomousModeKind` to differentiate it from `AutonomousMode`. (Maybe this is just me being weird.)
<!-- It's definitely just you being weird. You can't even claim that's a C# thing or a C thing, I'm pretty sure it's a Roslyn thing. -->
* I need to do a more in-depth lesson or something on how the continuation functions stuff works in our autonomous mode. It's a nice way to solve this problem, but it's hard for less experienced developers to understand.
* We should allow autonomous sequences instead of discrete continuation functions. It'd be more flexible for something like the intended full version of `HatchPanelAuto`.
  * If we did this, we might need a unified structure for peripherals to report "complete" status to something central. (IE: Last year's code had things besides `Drive` with continuation abilities.)
* Beter experience for composite modes in our infrastructure could be interesting. For instance, having teleop and autonomous at the same time could've been useful/reasonable this year.
