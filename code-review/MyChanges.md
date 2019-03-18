<!--
This is the Markdown source of this document. If you can see this comment, you probably want to view it on GitHub instead unless you intend to edit the document.
See comment in the source of Readme.md for details.
-->

# Changes I made

This document summarizes the minor changes I made to the code base while reviewing.

# `Robot`

* I reordered the peripherals to be alphabetic in all sections.
  * That way they're in the same order in their definitions, the constructor, `doPeripheralReinitialization`, and `doPeripheralPeriodicProcessing`.
  * Makes it easier to spot check that everything is there that should be. (Although the fact that some peripherals don't have `init`/`periodic` methods complicates this slightly.
* Moved `MotorTest` into its own file.

# `Drive`

* I added separator comments to delimit the different types of methods. (Autonomous, teleop, and internal.)
