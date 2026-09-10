# Ableton Live Nix Flake

In addition to the usual installer, this project is packaged as a nix flake, both to support NixOS, where the installer does not work, as well as any other Linux system using Nix.

NixOS is only supported via the flake, as NixOS does not allow dynamic library linking, so the standard installer will fail.

Please read through this whole document before using the flake, as there are many otions available.

## Installation

*Flakes are experimental and must be [enabled](https://wiki.nixos.org/wiki/Flakes#Setup).*

The flake builds and patches wine, this can take considerable time.

```
nix run github:shibco/ableton-linux
```

Before Ableton Live can be run, the wine prefix must also be setup. Unlike the standard installer, this requires an extra step.

```
nix run github:shibco/ableton-linux#setup-prefix
```

### Installing Live

Once the prefix is setup, you will be prompted to install Ableton Live. You can copy the command displayed, adding the path to your Ableton Live installer exe. You can also use the `.#wine` flake, which sets the wine prefix and uses the patched wine binary.

```
nix run github:shibco/ableton-linux#wine /path/to/ableton-live-installer.exe
```

You can also choose to auto-install Live while setting up the prefix. With a valid Live installer zip file in `~/Proprietary`, run:

```
nix run github:shibco/ableton-linux#setup-prefix --set-env-var ABLETON_LIVE_AUTOINSTALL 1
```

Or with your Live installer in another directoy run:

```
nix run github:shibco/ableton-linux#setup-prefix --set-env-var ABLETON_LIVE_AUTOINSTALL 1 --set-env-var LIVE_AUTOINSTALL_DIR /path/to/dir
```

### System Install as a Flake Input

You can use the default flake as a flake input. You still need to run `.#setup-prefix` manually.

With the flake provided as an input, you can add ableton-linux to your system packages (where `inputs.ableton-linux` is the flake input)

```
inputs.ableton-linux.packages.x86_64-linux.default
```

This will add an `ableton-live` command, as well as a .desktop file, for launching Live. This also adds the `ableton-wine` command, which is a shortcut for setting the wine prefix to `~/.wine-ableton` and running the patched wine, the same as `nix run .#wine`.

A basic example of installing this way is provided [here](#basic-system-flake-setup).

## Running Live

With the prefix setup and Live installed, you can run it with the default flake.

```
nix run github:shibco/ableton-linux
```

If the system package is installed, you can run it with `ableton-live` or through a program launcher with the .desktop file.

## Alternative Flake References

Although `github:shibco/ableton-linux` is a great flake location, running live with `nix run github:shibco/ableton-linux` will result in redownloading and rebuilding wine everytime there is a new commit to `main` in the github repo.

There are [several other ways](https://nix.dev/manual/nix/2.34/command-ref/new-cli/nix3-flake.html#flake-references) to reference the flake. In each case, replace `github:shibco/ableton-linux` with the alternative reference.

To pin a specific commit, either to avoid updating, or to test a particular version or pull request, add the commit ID, eg. `github:shibco/ableton-linux/c2092f702531712950649c4f957caff8203b2199`.

To follow a specific branch, add the branch name, eg. `github:shibco/ableton-linux/main`.

To use a local copy of the repo, download the repo and use the path to the local repo. `/path/to/flake/dir` or for relative path, `./relative/path/flake/dir`. Most sinply, use `.` for the current dir.

## Updating ableton-linux

Updating is a two-part process. If you are using `nix run`, the default flake will automatically rebuild when the flake reference has updated. If used as a flake input, use `nix flake update`.

After an update, it is recommended to run
```
nix run github:shibco/ableton-linux#setup-prefix -- --refresh
```
to apply any changes to the wine prefix. This is an idempotent command, and can be run anytime and repeatedly, even without `-- --refresh`, which just skips steps not needed for an existing prefix.

## Additional Flakes

You can also view all flakes in [flake.nix](flake.nix).

### Default

`.` - runs Live with the patched wine.

### Wine

`.#wine` - runs the patched wine runtime in the `.wine-ableton` prefix. Use this for installing plugins, eg. `nix run github:shibco/ableton-linux#wine plugin-installer.exe`

If the system package is installed, `ableton-wine` does the same thing, eg. `ableton-wine plugin-installer.exe`.

### Setup Prefix

`.#setup-prefix` - creates and prepares the wine prefix. Updates existing prefix. Can be run with `-- --refresh` to update the prefix.

### Check NTSync

`.#check-ntsync` - runs the `check-ntsync.sh` script to check if the NTSync kernel module is enabled and working. 

### Audio Report
`.#audio-report` - runs the `audio-report.sh` script, for troubleshooting audio issues. Prints the read-only audio diagnostic snapshot an issue report is expected to carry.

### Setup Realtime
`.#setup-realtime` - Install the distribution-canon pro-audio profile (rtprio, swappiness, governor; needs sudo)

### Setup Link
`.#setup-link` - Set up Ableton Link networking (firewall port 20808) and enable the ableton-linkd user service

Commands include `enable`, `disable`, and `status`, eg `nix run .#setup-link status`.

`enable` also supports setting the mode to `session` or `always`, eg `nix run .#setup-link enable -- --mode=session`

## Basic System Flake Setup

This is an example installation as a flake input, with the system package. There are many other ways to do this. 

This example uses the basic flake on the [NixOS Wiki](https://wiki.nixos.org/wiki/NixOS_system_configuration#Accessing_flake_inputs), and allows the continued use of an already setup `configuration.nix`.

`/etc/flake.nix`
```nix
{
  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
  inputs.ableton-linux.url = "github:shibco/ableton-linux";

  outputs = { self, nixpkgs, ... }@inputs: {
    # replace nixos with your hostname
    nixosConfigurations.nixos = nixpkgs.lib.nixosSystem {
      specialArgs = { inherit inputs; };
      modules = [ ./configuration.nix ];
    };
  };
}
```

Then in your `configuration.nix` add `inputs` to the arguments, eg:
```
{ config, pkgs, inputs, ... }:
```

Then add to your system packages, eg:
```
environment.systemPackages = with pkgs; [
  ...
]
++ [
  inputs.ableton-linux.packages.x86_64-linux.default
];
```

This is only an example setup, and must be modified to match your configuration.
