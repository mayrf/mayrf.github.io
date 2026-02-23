+++
date = '2026-02-23T23:54:24+01:00'
draft = false
title = 'Home Manager Silently Disables cache.nixos.org'
+++

# Home Manager Silently Disables cache.nixos.org

**TL;DR:** Setting `nix.settings.substituters` in Home Manager overrides your system config and disables `cache.nixos.org`, causing everything to build from source. Remove it from Home Manager.

## The Problem

If you're building packages from source instead of fetching from cache, check this:

```bash
$ nix show-config | grep substituters
# Is cache.nixos.org missing?
```

## The Culprit

Home Manager creates `~/.config/nix/nix.conf` which **replaces** system-level substituters:

```nix
# In Home Manager - this is the problem!
nix.settings = {
  substituters = [ "https://some-other-cache.com" ];
};
```

This is a [known issue](https://github.com/NixOS/nix/issues/4561) where setting substituters in Home Manager accidentally disables cache.nixos.org.

## The Fix

**Remove `nix.settings.substituters` from Home Manager:**

```nix
# BAD
nix.settings.substituters = [ ... ];

# GOOD - keep other settings, just not substituters
nix.settings = {
  experimental-features = [ "nix-command" "flakes" ];
  warn-dirty = false;
};
nix.settings.substituters = [ ... ];
```

Rebuild and verify:

```bash
home-manager switch  # or sudo nixos-rebuild switch
nix show-config | grep substituters  # Should show cache.nixos.org
```

## Why This Happens

**The real problem:** When you set `nix.settings.substituters` in Home Manager, it writes to `~/.config/nix/nix.conf`. This user-level configuration can override your system-level substituters from `/etc/nix/nix.conf`.

```nix
# In your NixOS config
nix.settings.substituters = [ 
  "https://cache.nixos.org"
  "https://nix-community.cachix.org"
];

# In your Home Manager config
nix.settings.substituters = [ "https://my-cache.com" ];

# Result when running nix commands as your user:
# You may only get: [ "https://my-cache.com" ]
# Losing access to: cache.nixos.org AND nix-community.cachix.org 
```

When Nix reads configuration, it processes both system (`/etc/nix/nix.conf`) and user (`~/.config/nix/nix.conf`) config files. For certain settings like `substituters`, the user config can take precedence, effectively replacing the system configuration instead of merging with it.

**The inconsistency:** NixOS modules automatically include `cache.nixos.org` as a default when you configure `nix.settings.substituters`, but Home Manager doesn't provide this same behavior.

### What About Standalone Home Manager?

If you're using Home Manager in **standalone mode** (on a non-NixOS system), you likely need to configure substituters in Home Manager since there may not be a system config. In this case, explicitly include all caches you want:

```nix
# Standalone Home Manager - include cache.nixos.org explicitly
nix.settings = {
  substituters = [
    "https://cache.nixos.org"
    "https://my-personal-cache.com"
  ];
  trusted-public-keys = [
    "cache.nixos.org-1:6NCHdD59X431o0gWypbMrAURkbJ16ZPMQFGspcDShjY="
    "my-personal-cache.com-1:..."
  ];
};
```

**On NixOS:** The safest approach is to not configure `substituters` in Home Manager at all. Add all caches (system-wide and user-specific) to your NixOS system configuration instead, where they'll be properly managed with the cache.nixos.org default.

## Related Issues

- [GitHub #4561](https://github.com/NixOS/nix/issues/4561) - "cache.nixos.org should only be disabled via a flag"
- [GitHub #158356](https://github.com/NixOS/nixpkgs/issues/158356) - Substituters behavior changes
- [Discourse](https://discourse.nixos.org/t/home-manager-doesnt-use-extra-substituters/33196) - Home Manager substituters problems

---

*Don't configure `substituters` in Home Manager on NixOS—let your system handle it.*
