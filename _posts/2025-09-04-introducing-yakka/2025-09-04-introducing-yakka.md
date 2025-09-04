---
layout: post
title: Introducing Yakka
---
Yakka is a declarative, data-focused build tool that aims to provide improved tooling for embedded software ecosystem.
It can operate as a complete build system with dependency and package management or can be used to complement an existing system.

To demonstrate its ability to improve the developer experience for complicated embedded environments, the basic Blinky application running in the Zephyr RTOS environment was ported to use Yakka.

The animation below shows Yakka preparing a development workspace by downloading all the tools and software required to compile and link the blinky example application running on the Silicon Labs xG24-RB4187C EFR32xG24 2.4 GHz Radio Board using GCC for ARM as the compiler.

![Yakka Zephyr](/assets/img/zephyr_blinky_demo.gif)

So what is Yakka doing?

The call to Yakka includes a single command, "link!", and names four components; "blinky", "xg24_rb4187c", "zephyr", and "gcc_arm".
The "-f" flag indicates that Yakka should automatically fetch any missing components.

To begin with, Yakka finds three of the four components in the component registry ("gcc_arm", "zephyr", and "xg24_rb4187c") and begins to download those. The "xg24_rb4187c" component downloads quickly and requires two further components; "silabs_platform", and "cmsis_core" which are downloaded next.
Once "zephyr" is fetched Yakka determines that "silabs_zephyr" is required and begins fetching that component.
The toolchain, "gcc_arm", is the last component to finish downloading before the "link!" command is executed.

In Yakka, commands are identified by the exclamation mark `!` at the end of the word and are defined within components as "blueprints" but more will be explained in further posts.
