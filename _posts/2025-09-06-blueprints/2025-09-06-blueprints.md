---
layout: post
title: Yakka Blueprints
---
In this post we will investigate and learn about Yakka blueprints and how to leverage them to perform almost any action required by a build process.
To begin with we will look at a basic blueprint and break down how it works...

```yaml
blueprints:
  my_blueprint:    # Blueprint name, can be almost anything
    process:       # The sequence of steps to "execute" the blueprint
      - echo: "My first blueprint" # The first, and only, step
```
Blueprints have a string with which it is identified, in this example it is `my_blueprint`. A blueprint can also list a number of dependencies using `depends` but this example has no dependencies at the moment. Finally, a blueprint has a `process` which is a sequence of steps, each step referencing a tool with arguments. Yakka has a number of built-in tools such as `echo` but components are also able to define their own tools.

Blueprints can be part of any Yakka component and so to test it we need to add it to a component and the only missing piece of information is a `name` and saving it as a `.yakka` file.

```yaml
# test.yakka
name: Blueprint test
blueprints:
  my_blueprint:
    process:
      - echo: "My first blueprint"
```

To call that blueprint from the command line we simply add a `!` to the end of the blueprint name and list our test component as part of the project. The command line should looke like `> yakka my_blueprint! test` which will output:

```
Processing[>                                                 ] 0% 0/1                                                                                                                      
[2025-09-06 09:21:27.058] [console] [info] My first blueprint
Processing[==================================================] 100% 1/1                                                                                                                    
Complete in 6 milliseconds
```

Blueprints typically require additional flexibility and Yakka provides that by access to the built-in template engine and the regex mechanism.

## Using the template engine
The template engine can be used in the name of the blueprint, in the dependencies, and in the process steps.
Yakka provides a number of special variables to assist referencing common data, such as `project` and `project_output` which reference the current roject name and the output folder for the current project.

Expanding our blueprint to generate a file for our project we can do the folling:
```yaml
name: Blueprint test
blueprints:
  my_blueprint:
    process:
      - echo: "My first blueprint"
      - save: '{% raw %}{{project_output}}{% endraw %}/data.txt'
```

This will save the output of the `echo` command to a file called `data.txt` found in the output folder for our project. Given our project only contains the `test` component we can determine that the `{% raw %}{{project_output}}{% endraw %}` template variable will expand to `./output/test/`.

This blueprint is functional but it doesn't allow our generated `data.txt` file to be part of a dependency tree. How can the build system know that the output of our blueprint is the `data.txt` file that some other build step requires?

To improve our blueprint further we change the name of our blueprint to the output target as follows:
```yaml
name: Blueprint test
blueprints:
  '{% raw %}{{project_output}}{% endraw %}/data.txt':
    process:
      - echo: "My first blueprint"
      - save:
```
Note that the `save` tool has built-in logic that assumes if no argument is given then the target filename is the blueprint name. Also note that now we've renamed our blueprint, our original command line with `my_blueprint! test` no longer works.
Since we know what the templated blueprint name will expand to we can still test this blueprint by modifying our command line to use the expanded name.

```> yakka output/test/data.txt! test```

This can be quite verbose for large project names and so to make things easier we can add another blueprint that will depend on output that we want to test.

```yaml
name: Blueprint test
blueprints:
  '{% raw %}{{project_output}}{% endraw %}/data.txt':
    process:
      - echo: "My first blueprint"
      - save:
  
  my_blueprint:
    depends:
      - '{% raw %}{{project_output}}{% endraw %}/data.txt'
```

With this new blueprint we can go back to calling Yakka with `my_blueprint! test` to generate `data.txt`.

## Using regular expressions
To create blueprints that are generic, Yakka provides support for regular expressions with capture groups to enable complicated behaviours.

Starting from our original, simple blueprint we are going to make the blueprint match to multiple blueprint names and generate unique output.
```yaml
# test.yakka
name: Blueprint test
blueprints:
  regex_blueprint: # This name is no longer used for blueprint matching
    regex: '(.*)_blueprint' # This regular expression is used for blueprint matching
    process:
      - echo: '{% raw %}{{$(1)}}{% endraw %} first blueprint'
```
Here we have added a new entry in the blueprint under the `regex` key that defines a regular expression that is used to match against any commands given on the command line or match any dependency of another blueprint. Note that by providing a `regex` entry will disable the use of the blueprint name, renamed in the example to `regex_blueprint`, as a blueprint.
The output we are echoing has also been modified to use the template engine which supports references to capture groups using the `$()` function (Note it is implemented as a function as the variable name `$1`, `$2`, etc are valid YAML keys).

If we call Yakka with `my_blueprint! test` we will see the same output as we saw before however we are now able to also call `your_blueprint! test` and see the output `[2025-09-06 16:01:21.793] [console] [info] your first blueprint`. Feel free to experiment and concatenate arbitrary strings to `_blueprint!`.

The final aspect to note is that the `regex` value in a blueprint can also be a template which is evaluated before being interpreted as a regular expression. For example:
```yaml
# test.yakka
name: Blueprint test
blueprints:
  regex_blueprint_command:
    regex: '(.*)_blueprint'
    depends:
      - '{% raw %}{{project_output}}/{{$(1)}}{% endraw %}_data.txt'
  
  regex_blueprint_for_output:
    regex: '{% raw %}{{project_output}}{% endraw %}/(.*)_data.txt'
    process:
      - echo: '{% raw %}{{ $(1) }}{% endraw %} blueprint'
      - save:
```
