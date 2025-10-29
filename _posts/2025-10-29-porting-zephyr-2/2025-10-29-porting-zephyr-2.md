---
layout: post
title: 'Porting Zephyr. 2 of #'
---
Adding support for Kconfig and Devicetree involves migrating the data from both custom formats to YAML and creating some UI components to help users configure complex systems.

# Data migration
Both Kconfig and Devicetree have their own, unique data format and so we will to convert them to a YAML structure so they can be part of Yakka.

## Devicetree
Starting with Devicetree, as the data is already in a tree format, we can relatively easily convert the `.dts`, `.dtsi`, and `.overlay` files with the exception of the use of C preprocessor logic such as `#ifdef`. It is possible to map the conditional of `#ifdef` to a relationship on a feature on component but they will need to be manually converted. At the time of writing there was 29 files out of 5000+ DTS files with conditional logic. 

```
soc {
  msc: flash-controller@50030000 {
    compatible = "silabs,series2-flash-controller";
    reg = <0x50030000 0x3148>;
    interrupts = <50 2>;

    #address-cells = <1>;
    #size-cells = <1>;

    flash0: flash@8000000 {
      compatible = "soc-nv-flash";
      write-block-size = <4>;
      erase-block-size = <8192>;
    };
  };
```
becomes
```yaml
zephyr:
  dts:
    soc:
      flash_controller_50030000:
        name: 'flash_controller'
        address: 0x50030000
        label: 'msc'
        compatible:
          vendor: 'silabs'
          type: 'series2-flash-controller'
        reg:
          - address: 0x50030000
            size: 0x3148
        interrupts:
          - irq: 50
            priority: 2
        address_cell_count: 1
        size_cell_count: 1
        flash_8000000:
          name: 'flash'
          address: 0x8000000
          label: 'flash0'
          compatible: 'soc-nv-flash'
          write_block_size: 4
          erase_block_size: 8192
```

All instances of `#include` in a DTS file represent a dependency on a component. For example, the EFR32MG24 DTS file contains a dependency on the `armv8-m-dts` component that previously expressed as `#include <arm/armv7-m.dtsi>`.
```yaml
name: EFR32MG24 DTS

requires:
  components:
    - armv8-m-dts
```

Given that we can distribute Devicetree data among arbitrary components, how can we aggregate it to form a single tree of Devicetree data?
There are two ways to aggregate data from multiple components; firstly, within a template there is a `aggregate(<pointer string>)` function that accepts a JSON pointer as a string and will iterate through all components in the project and returns a JSON object of the merged data, the second option which works better for this instance is using a data dependency.

## Data dependencies
A powerful feature of Yakka is the ability to not only define relationships between components or features, but for components to have a dependency on data as expressed as a [JSON Pointer](https://datatracker.ietf.org/doc/html/rfc6901).
Following the pattern already created for defining a `require` relationship between components we can add to the Zephyr component the following:
```yaml
requires:
  data:
    - '/zephyr/dts'
```
This informs Yakka that part of the build process will invovle aggregating all data from every component that comes under `/zephyr/dts` and merge it together. This is stored in the project summary under `/data` and will be accessible for blueprints as `{% raw %}{{ data.zephyr.dts }}{% endraw %}`.

As an example the following is a snippet from our Blinky Zephyr app `yakka_summary.json`:
```json
{
   "choices": {
   },
   "components": {
   },
   "configuration": {
   },
   "data": {
      "zephyr": {
         "dts": {
            "aliases": {
               "button0": "sw0",
               "button1": "sw1",
               "led0": "led0",
               "led1": "led1",
               "si7021": "dht0",
               "wdog0": "watchdog0"
            },
         }
      }
   }
}
```

## Data schema
In addition to aggregating the data, Yakka will also validate the data using a schema that can be defined in any component in the project.
The schema follows the standard as defined in (JSON Schema)[https://json-schema.org/specification] but can be defined in YAML and will be applied to YAML.

As an example we could have a Zephyr application that specifically requires `led2` and express that as a schema requirement:
```yaml
data_schema:
  zephyr:
    properties:
      dts:
        properties:
          aliases:
            required:
              - led2
```
This adds an entry to the `data_schema` defining a `zephyr` object that has a property called `dts` that has a property called `aliases` and that object must have an `led2` property. Running the `link!` command on the blinky project will now show this error:

```
[error]: Validation error in '': /zephyr/dts/aliases - {
   "button0": "sw0",
   "button1": "sw1",
   "led0": "led0",
   "led1": "led1",
   "si7021": "dht0",
   "wdog0": "watchdog0"
} : - required property 'led2' not found in object
```
This may be a silly example but it shows how data requirements paired with a schema allows users to be informed about invalid configurations before any tooling is invoked. You could also simplify this by defining a data entry called `zephyr_devices` and use that to track which Zephyr devices are available in a project instead of referencing the structure of the Zephyr Devicetree.

In that case the schema requirement becomes easier to read:
```yaml
data_schema:
  zephyr_devices:
    required:
      - led2
```

## Kconfig

## Configuration

# Data transformation
