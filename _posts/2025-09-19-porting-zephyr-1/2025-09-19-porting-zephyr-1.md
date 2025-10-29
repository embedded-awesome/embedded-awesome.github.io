---
layout: post
title: 'Porting Zephyr. 1 of #'
---

Today we will look at some ways in which Yakka blueprints can be used to replace the Python code used in Zephyr.

# Generating syscalls
An important part of building Zephyr is identifying the syscall functions and generating the trampoline calls that help isolate the kernel from the application.
Zephyr does this using two Python files (`parse_syscalls.py`, `gen_syscalls.py`) however we can recreate the functionality using blueprints.

Firstly there is the matter of listing the files that contain syscalls and require analysis. The second issue is that, for unknown reasons, Zephyr generates matching syscall files but stores them in a different directory structure. To address that choice, we can store the names of the syscall files we need and the source location of those files relative to the zephyr code base.

This data, and the blueprints we investigate next, are stored in `zephyr.yakka` that resides at the root of the Zephyr component. If you have an existing `zephyrproject` project by following the Zephyr introduction documentation this would be the `zephyrproject/zephyr` folder. For the Blinky example application when using Yakka, the directory structure looks as follows:

```shell
|- .yakka
|- components
|  |- cmsis_core
|  |- silabs_platform
|  |- silabs_zephyr
|  |- xg24_rb4187c
|  |- zephyr
|     |- zephyr.yakka
|- output
```

```yaml
# zephyr.yakka
name: Zephyr 

zephyr:
  syscalls:
    file_map:
      'device.h': 'device.h'
      'cache.h': 'cache.h'
      'kernel.h': 'kernel.h'
      'time_units.h': 'sys/time_units.h'
      'kobject.h': 'sys/kobject.h'
      'log_msg.h': 'logging/log_msg.h'
      'libc-hooks.h': 'sys/libc-hooks.h'
      'random.h': 'random/random.h'
      'log_ctrl.h': 'logging/log_ctrl.h'
      'entropy.h': 'drivers/entropy.h'
      'gpio.h': 'drivers/gpio.h'
```

With this data we can break down the syscall generation into two steps; first analyze the kernel header file to identify and extract the data from the appropriately tagged functions called `generate_syscall_yaml`, and second to generate the header files used by the application which contain trampoline functions for each syscall called `generate_syscall_file`.

```yaml
blueprints:
  generate_syscall_file:
    group: "Generating"
    regex: {% raw %}'{{project_output}}/generated/zephyr/syscalls/(.*\.h)'{% endraw %}
    depends:
      - {% raw %}'{{project_output}}/syscall_data/{{$(1)}}.syscalls.yaml'{% endraw %}
    process:
      - template:
          template_file: {% raw %}'{{curdir}}/gen_syscall.template.jinja'{% endraw %}
          data_file: {% raw %}'{{project_output}}/syscall_data/{{$(1)}}.syscalls.yaml'{% endraw %}
      - save:
  
  generate_syscall_yaml:
    group: "Analyzing"
    regex: {% raw %}'{{project_output}}/syscall_data/(.*).syscalls.yaml'{% endraw %}
    depends:
      - {% raw %}'{{curdir()}}/include/zephyr/{{ at(data.zephyr.syscalls.file_map, $(1)) }}'{% endraw %}
    process:
      - cat: {% raw %}'{{curdir()}}/include/zephyr/{{ at(data.zephyr.syscalls.file_map, $(1)) }}'{% endraw %}
      - regex:
          prefix: {% raw %}"name: {{ not_dir($(1)) }}\nfunctions:\n"{% endraw %}
          search: '(?:__syscall|__syscall_always_inline)\s+([^(]+[*\s])([^(]+)\(([^)]*)\)'
          to_yaml:
            - return   # $1
            - function # $2
            - arg_list # $3
      - save:
```

The `generate_syscall_yaml` blueprint is equivalent to `parse_syscalls.py` from `zephyr/scripts/build/` and the `search` value in the regular expression in the `regex` step is the exact expression taken from `parse_syscalls.py` (line 36 in the version referenced at the time of writing).
Focusing on this blueprint we can analyze each line.

1. `group: "Analyzing"` informs Yakka that each instance of this blueprint should be accounted for in it's own group of tasks titled "Analyzing". This appears as a unique progress bar when executing a command that includes the syscalls as part of the dependency tree and appears in the terminal as:
```shell
Analyzing[==========================================>       ] 84% 11/13
```
2. `regex: {% raw %}'{{project_output}}/syscall_data/(.*).syscalls.yaml'{% endraw %}`
This indicates our blueprint is a regular expression that matches any file under the `syscall_data` folder of the project output that ends in `.syscalls.yaml`. We also capture the name of the file, excluding the `.syscalls.yaml` section, and that can be used elsewhere in the blueprint as part of a template as `{% raw %}{{ $(1) }}{% endraw %}`
3. The {% raw %}`{{curdir()}}/include/zephyr/{{ at(data.zephyr.syscalls.file_map, $(1)) }}`{% endraw %} dependency has two template entries to idenitfy the location of the header file that will be analyzed.  
The first template expression {% raw %}`{{curdir()}}`{% endraw %} calls the built-in `curdir` function that returns the directory/path of the component in which the blueprint is defined. This, being defined in `zephyr.yakka` contained the `zephyr/` folder will evaluate to `components/zephyr/` 
The second template expression {% raw %}`{{ at(data.zephyr.syscalls.file_map, $(1)) }}`{% endraw %} uses the `at` function to lookup the value in the object `data.zephyr.syscalls.file_map` for the key `$(1)`. We know that `$(1)` references the first matching group of the regex expression and as a concrete example it would evaluate to `device.h`. From the lookup table we have created earlier we can see that `device.h` is a simple mapping to `device.h`.  
The whole expression evaluates to `components/zephyr/include/zephyr/device.h`.  
> You may have noticed that we referenced the lookup table data contained in this component using `data.zephyr.syscalls.file_map` and wonder how does that work. A further explanation of the template engine and how it interacts with the data model is coming soon.
4. We now move on to the `process` that describes the sequence of actions that will be executed, the first being `cat: {% raw %}'{{curdir()}}/include/zephyr/{{ at(data.zephyr.syscalls.file_map, $(1)) }}'{% endraw %}`.  
Every process step is a key value pair where the key references a built-in command or a "tool" and the value is the argument. In this instance the blueprint calls the built-in `cat` command that reads the content of the file named in the argument and stores it in the internal buffer that is passed to subsequent commands in a similar manner as the pipe operator used in a terminal or in the template engine. Each step in a blueprint process has an implicit pipe allowing commands to reference the output of the previous command if required.
5. The second step of the process utilizes a powerful built-in command called `regex` that performs a regular expression search or match. In this instance the the command has three "arguments" that dictate how the `regex` is to act; `prefix`, `search`, and `to_yaml`, and the input is the output of the previous `cat` command.  
The `prefix: {% raw %}"name: {{ not_dir($(1)) }}\nfunctions:\n"{% endraw %}` line is an option of `regex` to prepend a prefix string to the output. It has a template element, `not_dir($(1))`, that returns the non-directory part of a given path.  
The `search: '(?:__syscall|__syscall_always_inline)\s+([^(]+[*\s])([^(]+)\(([^)]*)\)'` line is our regular expression that will trigger for every match in 


Here is a copy of a generated `device.h.syscalls.yaml` file:
```yaml
name: device.h
functions:
- return: const struct device *
  function: device_get_binding
  arg_list: const char *name
- return: "bool "
  function: device_is_ready
  arg_list: const struct device *dev
- return: "int "
  function: device_init
  arg_list: const struct device *dev
- return: "int "
  function: device_deinit
  arg_list: const struct device *dev
- return: const struct device *
  function: device_get_by_dt_nodelabel
  arg_list: const char *nodelabel

```
For your reference, the `zephyr/include/zephyr/device.h` header file contains function declarations, some of which are prefixed with `__syscall` as follows:
```c
__syscall const struct device *device_get_binding(const char *name);
```