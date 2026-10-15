# Diagrams

Architecture and interface diagrams for the SSTV receiver. See the [project overview](../README.md) for how these fit together.

***

## Development Workflow

<p align="center">
  <img src="Core%20&%20OS/images/SSTV%20Design%20Workflow.png" width="650">
</p>

The two-phase plan. Phase 1 builds a working receiver end to end. Phase 2 adds storage, power management, and a refined interface.

***

## Software Structure

<p align="center">
  <img src="Core%20&%20OS/images/RTOS%20Workflow.png" width="560">
</p>

The originally planned operating system layer, in which tasks would reach peripherals through a scheduler running across both cores. This approach was later replaced by a messaging layer written directly against the hardware.

***

## Display and Menu

<p align="center">
  <img src="Display%20&%20Menu/SSTV%20Menu%20UI%20Workflow.png" width="620">
</p>

How the interface moves between screens, from the boot screen through image display, frequency entry, settings, and storage.

<p align="center">
  <img src="Display%20&%20Menu/SSTV%20Menu%20UI%20Implementation.png" width="620">
</p>

Layouts for the individual screens, including live image view, frequency tuning, the saved image gallery, and the station list.

***

## Enclosure

Enclosure models and turntable views are in [CADs & PCBs](CADs%20&%20PCBs/readme.md).
