+++
title = "Organising my dotfiles with ez-configs"
date = 2023-11-17

[taxonomies]
categories = ["Nix"]
tags = ["nix", "nixos", "flakes", "flake-parts"]

[extra]
lang = "en"
toc = true
+++

<link rel="stylesheet" href="public/style.css">

# Yet another flakes library

I've recently created a [`flake-parts`](https://github.com/hercules-ci/flake-parts) flake module called [`ez-configs`](https://github.com/ehllie/ez-configs). It's goal is to streamline the often verbose process of creating and managing system and home manager configurations in your nix flakes. In this post I'll go over it's features and usage, as well my motivations for creating it.

<!-- more -->

It seems like every nix user is obligated to reinvent the wheel at some point. I'm no exception to that rule, as I've gone through [quite a few iterations](https://github.com/ehllie/dotfiles/commits/main?branch=main&path%5B%5D=flake.nix&qualified_name=refs%2Fheads%2Fmain) of the flake.nix file in my dotfiles repository. Each one would most likely be difficult to understand for anyone other than me. All of that work to just pass the paths of my nixos, darwin and home manager modules to their respective builder functions. A very simple task, but I was always frustrated by how much "boilerplate" code one needed to write to achieve it. The configuration building functions(`nixosSystem`, `darwinSystem`, `homeManagerConfiguration`) make no assumptions about your setup, and as such need to be passed a lot of arguments. I've eventually arrived at a [solution I was satisfied with](https://github.com/ehllie/dotfiles/blob/83ef4ec1360820d231c9770e486eaaba862f7462/flake.nix). I was satisfied with it to such a degree, that I felt like the only way forward from there was creating a flake-parts module out of it, since I've recently wrapped my head around them, and found myself enjoying using them.

# The tools I used

My previous post assumed the reader already had pre-existing knowledge about nix and NixOS. While I won't attempt to give a thorough explanation of them here, I'll do my best to at least provide a basic rundown of what they are, and why I use them. A better introduction can probably be found at the [Zero to Nix](https://zero-to-nix.com), but if you'd like to get my own attempt at doing the same, keep reading. You can skip to the first section you are not familiar with in the table of contents, or skip this section entirely if you are already familiar with all of them.

## Nix?

Nix itself is a package manager that uses the nix expression language to create derivations, that are then built into packages. That's a lot of fancy words in one sentence, so let's try to unpack them.

- Package manager: This one should be familiar to most programmers or Unix-like system users. It's meant to build software and fetch any dependencies that software might require. It's unique among most other system package managers, in the fact it does not put any of the software it installs into `/bin` or `/usr/bin`, or any other usual system directories, but rather inside its own individual package output directory inside the `/nix/store`, and that package output directory name starts with its sha256, usually followed by the name of the package and its version. That prevents conflicts from packages requiring different version of the same dependency, and makes uninstalling them very simple. This approach effectively sidesteps one of the biggest package management problems.
- Nix expression language: Nix uses it's own simple DSL to achieve this goal. In a big simplification, nix expressions can be thought of as JSON, but with functions. The syntax itself is ML based, so it might be unfamiliar to a lot of people without functional language experience, but most nix expressions are either simple attribute sets, object equivalent in JSON, or functions that produce attribute sets.
- [Derivations](https://nixos.org/manual/nix/stable/language/derivations.html): A derivation is essentially a recipe for a bash script that was created using a nix expression. The nix package manager uses the attribute sets created by nix expressions that define steps needed to create a package, all the build time and runtime dependencies it requires, and executes them in a mostly isolated environment. Those derivations are then expected to put the built package in the `$out` directory that's given to them by the package manager.

In summary, Nix lets you abstract packages as functions that as input take their dependencies, construct derivations, essentially glorified bash scripts, using those inputs, and then run those derivations in isolation from the rest of your system to produce an output. This information is not necessary for using nix most of the time, but if you want to build your own software with it, it's useful to know. Most nix users will also most likely not use derivations directly, as nixpkgs, the official package repository, provides tools for building packages written using a [variety of languages and frameworks](https://nixos.org/manual/nixpkgs/stable/#chap-language-support).

## NixOS?

NixOS builds on top of the package manager, but thinking of a package in a looser sense. There's no restriction that requires derivation to produce binary files, or even executable scripts. They can produce just about anything. They just have inputs and an output. NixOS uses that system to create outputs which contain system activation scripts. Things like which services to start, boot loader entries to add, file systems to mount, users to create etc. To make the process of creating system derivations easier, it created a DSL within the nix expression language, called the [nix module system](https://nixos.org/manual/nixos/stable/#sec-configuration-syntax).

Modules are checked for correctness before the system is built with them, they can import other modules, and can define new options and the rules for checking their correctness so that those rules could be used by other modules. Again, seem complicated, and admittedly it is, but modules were created to solve a complicated problem. Declarative configuration of an entire operating system. Additionally, while both of them are called modules, you can distinguish two types of modules. One that provide new options and do things with them, which can be thought of as libraries that abstract over specific software configuration, and the ones that just set the options to create the system, which is what most NixOS users write when they for example want to add themselves as a user on their system, or set the desktop environment they use. The latter is usually simpler, since the former's job is to abstract the more complicated parts.

The module system is not limited to just creating NixOS activation scripts. At a fundamental level, it's just a way of defining options, setting and checking those settings in a composable way. [Home Manager](https://github.com/nix-community/home-manager) and [nix-darwin](https://github.com/LnL7/nix-darwin) both use them, to create a set of modules for defining user home directory configurations and MacOS system configurations, as well as programs to apply those configurations.

## Flakes?

While Nix takes a lot of steps to be reproducible, before flakes there was still one aspect that could differ between each machine when running the same nix expression. The channels, and what version of the channel each user was using. I hadn't had much experience using them, as almost instantly started using flakes, but it was essentially an imperative element to Nix, which required the user to periodically update them from the cli, which would then modify things inside your `/etc/nix`. There is some [controversy](https://discourse.nixos.org/t/why-are-flakes-still-experimental/29317/12) around flakes in the nix community, and they have been labelled as experimental since 2018. Nonetheless, they do provide a first party solution to the problem of reproducibility, and also allow for easier use of software from outside of nixpkgs.

A flake is just a `flake.nix` file, which contains a single attribute set inside it. The main attributes in that set are `inputs` and `outputs`, the former of which defines dependencies your flake will use, and the latter what it will do with them. Whenever a flake is evaluated, a `flake.lock` file is created if not already present, which pins the versions of all of your flake inputs to the state they were when you first created the file, or when you last ran `nix flake update`. The `flake.nix` and `flake.lock` files can be though of as `Cargo.toml` and `Cargo.lock` in rust projects, or `package.json` and `package-lock.json` in js projects. When evaluating a flake, you get the value from passing in the `inputs` to its `outputs` function. There is [a schema](https://nixos.wiki/wiki/Flakes#Output_schema) for the outputs provided by the flake, but there are plenty of non-official outputs non included in that schema, like `homeConfigurations` and `darwinConfigurations` used by home-manager and nix-darwin.

## Flake Parts?

Because flakes are ultimately simple tool, being just nix expressions, they can become quite complex when trying to express more complicated and bigger things. This tends to result in proliferation of common nix glue code, that is rewritten each time someone wants to do something with their flake. Be it providing package outputs and overlays for the project they are building using nix, or like in my case, providing configuration and module outputs for use in `nixos-rebuild`, `home-manager` and `darwin-rebuild`. [Flake parts](https://github.com/hercules-ci/flake-parts) creates a framework for defining those outputs using the module system:

```nix
{
  inputs = {
    flake-parts.url="github:hercules-ci/flake-parts";
    # The rest of your inputs
  };

  outputs = inputs@{ flake-parts, ... }:
    flake-parts.lib.mkFlake { inherit inputs; } {
      # This is a nix module, so you can import other modules,
      # define new options or set them.
    };
}
```

This, or an equivalent nix expression, is all "flake magic" code you will need to invoke. The rest will be done using the module system. It also comes with a convenient website for [searching options](https://flake.parts/options/flake-parts) from a variety of flake-parts libraries, including [ez-configs](https://flake.parts/options/ez-configs).

# How and why ez-configs

# My dotfiles cleanup
