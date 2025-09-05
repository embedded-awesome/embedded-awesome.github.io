---
layout: post
title: Introducing Yakka
---
Yakka is a declarative, data-focused build tool that aims to provide improved tooling for embedded software ecosystem.
It can operate as a complete build system with dependency and package management or can be used to complement an existing system.

To demonstrate its ability to improve the developer experience for complicated embedded environments the basic Blinky application running in the Zephyr RTOS environment was ported to use Yakka.

The animation below shows Yakka preparing a development workspace by downloading all the tools and software required to compile and link the blinky example application running on the Silicon Labs xG24-RB4187C EFR32xG24 2.4 GHz Radio Board using GCC for ARM as the compiler.

![Yakka Zephyr](/assets/img/zephyr_blinky_demo.gif)

Unlike the original Zephyr build environment, Yakka only requires Git to operate. There are no installation steps, no Python, no device tree compiler, the single 4 MB executable is small enough to be included in the workspace version control repo.

# So what is Yakka doing?

The call to Yakka includes a single command, `link!`, and names four components; `blinky`, `xg24_rb4187c`, `zephyr`, and `gcc_arm`.
The `-f` flag indicates that Yakka should automatically fetch any missing components.

Once `zephyr` is fetched Yakka determines that `silabs_zephyr` is required and begins fetching that component.
The toolchain, `gcc_arm`, is the last component to finish downloading before the `link!` command is executed.

In Yakka, commands are identified by the exclamation mark `!` at the end of the word and are defined within components as "blueprints" but more will be explained in further posts. In this instance, the `link` blueprint is part of the toolchain component `gcc_arm` and is responsible for initiating the dependency tree that generates the project ELF file.

# What's next?
In the upcoming weeks we will be documenting how Yakka works internally and detailing the process of supporting Zephyr.