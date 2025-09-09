---
layout: post
title: 'Package Management == Trust + Support'
---
Package Management has been a topic of heated discussion over many years due to the trade offs that must be negotiated within a software development team. The engineers developing new software may feel empowered by the ability to download code instead of writing it themselves, while the engineers testing or maintaining the software may pay the price of 'dependency hell' while attempting to provide trust and confidence in a software product that is built upon a pile of arbitrary code snippets downloaded from all corners of the Internet.

Admittedly, "all corners of the Internet" is an exaggeration as one of the goals of a package manager is to provide a centralized location to exchange packages and often provide a means to express what platforms the package supports. However when it comes to end goal of the software development team, package managers must also provide a level of trust and confidence so that the software built on top of the packages can have a firm foundation.

Given the nature of all human endeavours we are realistic in our ability to write bug free software and so the first question a software development team should ask when relying on a package from a package manager is "who can I talk to when I have a problem?".

For almost all centralized package managers the answer is "Good luck", and the developer is left to find help on some forum or a Discord channel but the onus on the development team to own and manage the any packages themselves.

## Embedded package management
When it comes to embedded software development there is no established package management solution primarly because the current support model relies on silicon vendors (Silicon Labs, STM, Nordic, Infineon, Espressif, ...) to provide software support due to the highly integrated nature of embedded software. The Zephyr Project is an attempt to move towards a standardized software environment however silicon vendors will often support customers through an internally managed copy and tend to not officially support the latest release on Github. 

In the embedded software space, developers are not just looking for easy access to off-the-shelf software but are acutely aware of the requirement to be responsible for every line of code that runs in the system and have some level of accountability when something does not go to plan. Centralized package managers do not provide the trust and support that developers are looking for.

## How does Yakka do it?

Yakka's approach to package management builds upon the status quo of the embedded space where a customer of a silicon vendor starts their project by downloading the vendor SDK and experimenting with the sample applications. There is trust that the silicon vendor has done their best to ensure their SDK is functional and customers are able to use the software as a basis for building their product. There is also a contract of support (assuming the sales team has been involved) to help the customer build their product, pass certification, and get it on the shelves. From a silicon vendors perspective, if their customer can't get their product into the hands of their customers they won't be putting in purchase orders for devices.

In light of this, Yakka has the concept of a "package registry" that is controlled and managed by the silicon vendor or software foundation. The package registry is equivalent to an SDK where a particular version of the package registry references all the versions of the individual packages that are known to work together. Each package, or 'component' in Yakka terms, has it's own version and can be managed individually to provide customer specific support or inter-SDK release bug fixes.
To keep-it-simple, Yakka implements the package registry as a Git repo providing versioning via branches and tags and reliable security via client and server certificates.

On the vendor/host side, customer specific SDKs can be trivially supported by forking the official package registry, adding custom packages and exposing via an alternate SSH server with restricted access via client certificates.
To include support for third-party code such as Zephyr, a silicon vendor can simply merge a snapshot of the Zephyr package registry at a particular version that has been tested and verified into their own registry and now their customers are able to utilize Zephyr just as easily as any other component.

On the client side, all package fetching can be done with verified server certificates reducing the chance of man-in-the-middle attacks. Package registries can be easily cloned allowing software development teams to run their own internal registry restricting access to external registries and providing a means in which updates to vendor SDKs can be integrated in a controlled manner rather than relying on individual developers to update software on their development machines manually.
> Note:
> It is expected, and recommended as a best practice, that commercial users of Yakka who have a team of developers would manage their own internal package registry rather than have developers fetch components directly from external registries.

-- add pic of package registry --


