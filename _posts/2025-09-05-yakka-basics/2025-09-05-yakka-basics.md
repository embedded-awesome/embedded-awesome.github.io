---
layout: post
title: Yakka Basics
---
During the development of a build process there is contention between managing the data that describes the source files, build flags, etc and managing the conditionals that control which data and build steps are required to generate a particular output.
Many build systems decide to store both data and conditional logic in an existing programming language or create their own language while Yakka leans into defining a strong data model and using data transforms as a basis of building more complicated systems.
To begin with, Yakka supports defining Yakka files in YAML due to the human readability however it will be possible to use alternate data formats such as JSON or TOML (ideally the file format would be auto-detected however it would be possible to support `.yakka.json` or `.yakka.toml` as a file extension).

The basic unit within Yakka is a `component` which contains data and has relationships to other components of the project. The content of a component can have conditional inclusion controlled by `features`.
Components can have relationships not only to other components but also have relationships to features, and the three relationship types are `requires`, `supports`, and `provides`.

Below is the Yakka file for the Blinky app component from Zephyr called `blinky.yakka`:
```YAML
name: Blinky app

sources:
- src/main.c

requires:
  components:
    - zephyr
```

The `name` entry is mandatory and provides a human readable name for the component.
The filename `blinky.yakka` defines the string used to reference the component, in this instance it is `blinky`. Renaming the file will change the identifier used to reference that component. The location of the file defines the root folder for that component which is used when referencing files such as `sources`.
Multiple components can share the same root folder as they all need to be uniquely named.
Given that the content of a component is relative to the location of the component file, it is possible to structure the workspace arbitrarily as Yakka will scan the workspace for missing components.

The final part of the file defines a relationship between the `blinky` component and the `zephyr` component expressing that the Blinky app requires a component called `zephyr` to be part of the project. Yakka will search the workspace for such a component, which you can guess will be called `zephyr.yakka`, and will include that as part of the project.
If that component cannot be found, it will look through any component registries and, if enabled by the `-f` flag, will automatically download the component to the workspace.

Yakka is designed to flexible for all kinds of activities and so does not have a an internal concept of 'source files' or 'build flags' but relies on components to define those concepts via a schema and define the data transforms that are performed through blueprints. When compiling embedded C code, the toolchain is the component that understands `sources` and uses that information to call GCC appropriately.

New blog posts will expand on blueprints and how they work...