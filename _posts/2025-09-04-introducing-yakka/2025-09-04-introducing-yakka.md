---
layout: post
title: Introducing Yakka
---
Yakka is a declarative, data-focused build tool that aims to provide improved tooling for embedded software ecosystem.
It can operate as a complete build system with dependency and package management or can be used to complement an existing system.

To demonstrate its ability to improve the developer experience for complicated embedded environments, the basic Blinky application running in the Zephyr RTOS environment was ported to use Yakka.

The animation below shows Yakka preparing a development workspace by downloading all the tools and software required to compile and link the blinky example application running on the Silicon Labs xG24-RB4187C EFR32xG24 2.4 GHz Radio Board using GCC for ARM as the compiler.

![Yakka Zephyr](zephyr_blink_demo.gif)

