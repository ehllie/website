+++
title = "Personal nix builders behind Cloudflare Tunnel"
date = 2023-11-16

[taxonomies]
categories = ["Nix"]
tags = ["nix", "nixos", "cloudflare", "sops"]

[extra]
lang = "en"
toc = false
+++

# Finally a post

It's been over half a year since my first post, and I never came around to writing the post I had originally promised. Nonetheless, I hope you'll find what I have to say about my experience working with the nix ecosystem interesting.

<!-- more -->

This post won't be covering a lot of the terms I'm going to be using like nix, nixos, nix-darwin, home-manager etc. Because of that, people not familiar with them already might find themselves at least a little confused.

I have a fair bit of experience using nix, having started using it over a year ago, and having made a couple minor nixpkgs contributions since. I'm constantly trying to expand the number things I can do with it.

One of those things I want to tackle is automatic system provisioning and deployment using [nixops](https://github.com/NixOS/nixops). One caveat of that, is that my main, and essentially only machine, is an M1 MacBook Pro. I'm very happy with it, nix tools work flawlessly for me most of the time, but in order to use nixops I would need to access a remote builder that matches the deployment platform. Considering most of the time those deployments would be `x86_64-linux`, I need to configure a `x86_64-linux` machine running nixos for remote nix builds.

# Configuring nix.buildMachines

Very conveniently, nix-darwin seems to have an [option which could do just that](https://daiderd.com/nix-darwin/manual/index.html#opt-nix.linux-builder.enable). It creates a guest linux machine and ssh keys to access it, as well as appends its configuration to `nix.buildMachines`.

```nix
# Part of my darwin machine, nix-darwin configuration module.
{ inputs, pkgs, ... }:
let
  unstable-pkgs = import inputs.nixpkgs-unstable { inherit (pkgs.stdenv) system; };
  inherit (unstable-pkgs.darwin) linux-builder;
in
{
  nix = {
    distributedBuilds = true;

    linux-builder = {
      # We need to pull in the package from nixpkgs-unstable,
      # since it's not available in the stable 23.05 release I'm using
      package = linux-builder;
      enable = true;
    };
  };
}

```

Very easy so far. The issue is, that the guest machine is an `aarch64-linux`. While it's true that plenty of cloud platforms provide `aarch64` VM options, there's still plenty of software that does not due to upstream issues.

I do have an idle intel laptop that hasn't seen any use for almost a year now, and this seems like a great opportunity to put it to use. After some minor issues with getting a fresh system install set up, I've managed to deploy this minimal configuration to the machine.

```nix
{ config, inputs, lib, modulesPath, pkgs, ... }:
{
  services = {
    openssh = {
      enable = true;
      settings.PasswordAuthentication = false;
    };
  };

  networking = {
    hostName = "dell-builder";
    networkmanager.enable = true;
  };

  users.users = {
    builder = {
      isNormalUser = true;
      description = "Nix Builder";

      # The first key is the one we will use for authenticating build jobs, the second one is for directly connecting via ssh and debugging
      openssh.authorizedKeys.keys = [
        "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGWKSe5h51wlK0jkQidL1EVdIiswlMCjUjmOhN7USzbr ellie@EllMBP.local"
        "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIB3C7/YxpoLu57b5XM2L0FVoRR5Qhju/9wxY082kmGCx"
      ];
    };

    root.openssh.authorizedKeys.keys = [
      "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIB3C7/YxpoLu57b5XM2L0FVoRR5Qhju/9wxY082kmGCx"
    ];
  };

  nix = {
    extraOptions = "experimental-features = nix-command flakes";

    # This is here so that the builder user is allowed to create arbitrary, input-addressed derivations.
    settings.trusted-users = [
      "root"
      "builder"
    ];

    gc = {
      automatic = true;
      dates = "daily";
      options = "--delete-older-than 7d";
    };
  };
```

I've skipped hardware configuration that's not relevant to the thing we need here. But beyond configuring the build host, I also need to configure my MacBook to know that a build machine is even available for it. That was quite simple too, as those options are pretty well documented inside both [`nix-darwin`](https://daiderd.com/nix-darwin/manual/index.html#opt-nix.buildMachines) and [`nixos`](https://search.nixos.org/options?channel=23.05&query=nix.buildMachines). These are the new parts added to the previous darwin module:

```nix
{
  nix.buildMachines = [{
    # I've set a static ip address for the builder inside my DHCP server settings
    hostName = "192.168.1.4";
    sshUser = "builder";
    system = "x86_64-linux";
    supportedFeatures = [ "kvm" "benchmark" "big-parallel" ];
    publicHostKey = "c3NoLWVkMjU1MTkgQUFBQUMzTnphQzFsWkRJMU5URTVBQUFBSU1mekdFaTJBTk5wdENta3h2ZXJSckdvWFY5R2Z2MWtya2ZtdElRbXV2NjAgcm9vdEBuaXhvcwo=";
    sshKey = "/etc/nix/dell-builder_ed25519";
    maxJobs = 8;
  }];
}
```

And that works just fine! I can compile `x86_64-linux` packages, and the nix-daemon knows to hand off the build jobs to that builder. But there is still one issue. This will only work while I'm at home inside my local network. I'd like to have something that will let me use the builder from anywhere, but I'd also like to avoid exposing my public IP address to the world at large.

# Cloudflare tunnel

I have heard about an interesting service that could solve my problem a while back. It allows you to setup a service on your server which establishes a connection with Cloudflare, and then you can setup a CNAME record in your DNS configuration pointing to the created tunnel. All traffic runs through a socket between your server and the Cloudflare proxy, so it's not possible to find your public IP address from DNS record, as they will just resolve to Cloudflare's IP.

Cloudflare tunnels have configuration [options in nixos](https://search.nixos.org/options?channel=23.05&query=cloudflared.tunnels), and it was pretty simple to adapt the [official clodflare documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/get-started/create-local-tunnel/). I preferred using the locally-managed tunnel over the remotely-managed one, as the former lets me keep my configuration inside my VCS, and also seems to have more support inside nixos. Deployment on the builder requires a credentials file, which to the best of my knowledge can be created anywhere using the `cloudflared` cli. Secrets are not natively supported in the nix store, but there are a couple ways around that issue, and my favourite one is [`sops-nix`](https://github.com/Mic92/sops-nix). Explaining it is a bit out of the scope for this post, but if you're interested in how that works, here's [a good article](https://lgug2z.com/articles/handling-secrets-in-nixos-an-overview/) on the topic I've found recently. After setting up secrets in my repository, these are the changes I've made to the builder host:

```nix
{ config, inputs, lib, modulesPath, pkgs, ... }:
let inherit (config.sops.secrets) tunnel-credentials; in
{
  imports = [
    inputs.sops-nix.nixosModules.default
  ];

  sops.secrets.tunnel-credentials = {
    owner = "cloudflared";
    group = "cloudflared";
    name = "tunnel-credentials.json";
    format = "binary";
    sopsFile = ../sops/builder/tunnel-credentials;
  };

  services = {
    cloudflared = {
      enable = true;
      tunnels."b0c2f8d9-05ba-4c16-8f03-36cd9bea5c52" = {
        credentialsFile = tunnel-credentials.path;
        default = "http_status:404";
        ingress = {
          "builder.ehllie.xyz".service = "ssh://127.0.0.1:22";
        };
      };
    };
  };
}
```

I've also updated the `hostName` option inside my darwin module's `nix.buildMachines` list item to `"builder.ehllie.xyz"`.
The configuration deployed, `journalctl` at the builder showed it's running properly, the Cloudflare dashboard showed it as healthy. However I could still not connect to `root@builder.ehllie.xyz` over SSH. I spent a fair bit of time trying to troubleshoot the issue. Thinking that maybe the connection is still going through my router, setting up NAT rules, reading through connection logs on my server. Ultimately I was pointed out a section in the manual by a friend of mine, which covered [using SSH through the tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/use-cases/ssh/#connect-to-ssh-server-with-cloudflared-access). Turns out the tunnels don't forward all traffic by default, and they are not completely transparent to SSH connections. As such a `proxyCommand` rule needs to be setup for your host when connecting via ssh. This is the recommended Cloudflare configuration I've adapted into a `home-manager` module:

```nix
{ pkgs, ... }:
{
  programs.ssh = {
    matchBlocks."builder.ehllie.xyz" = {
      proxyCommand = "${pkgs.cloudflared}/bin/cloudflared access ssh --hostname %h";
    };
  };
}
```

Notably, it needs to be used by your non-root user, as well as root. That is because nix-daemon runs as root, and has no knowledge of your personal SSH configuration. Luckily `home-manager` can be used as a `nixos` or `nix-darwin` module, so it was possible to include that bit of configuration inside my nix modules without much friction.

# I hope that was fun

I personally feel like this was a fun side adventure. I haven't made any progress in learning `nixops` like I had set out to do, but I still ended up with having 3/4 default nixpkgs platforms available to me. Being able to SSH into my home network from anywhere in the world, while preserving some degree of safety, is a nice bonus too. I don't know how reliable this solution is going to be long term yet, but from my experience using nixos on servers I'm quite optimistic. All the things I've covered in this post are available in my public dotfile repository, so if you're interested, feel free to see the [relevant commit](https://github.com/ehllie/dotfiles/commit/e50215149d5c86e67811719fc2d7f827de6323f5).

Thank you for your time,

Ehllie
