# nixos-conda-r-jupyter-guide
Guide for running R (IRkernel) in JupyterLab on NixOS using nix-ld and micromamba. Fixed issues: missing /usr/bin/which, R base packages not found, R kernel not connecting.

# Running R in JupyterLab on NixOS

### The Problem

You're a data scientist, and your workflow involves R, Jupyter Lab, and Python. You have an env.yaml file for a project in Miniconda and want to use Conda to set up the environment in NixOS. 

BUT NixOS does not follow the Filesystem Hierarchy Standard (FHS). This breaks Conda in two ways:

1. **No dynamic linker at `/lib64/ld-linux-x86-64.so.2`**
   Every precompiled Conda binary expects this file to exist. On NixOS it doesn't,
   so every Conda binary crashes immediately with "No such file or directory".

2. **No `/usr/bin/which`**
   Conda's R package (`r-base`) was compiled on a standard Linux distro where
   `/usr/bin/which` exists. R hardcodes this path internally — it's not resolved
   via `$PATH`. When R starts, it calls `/usr/bin/which` to initialize core
   packages (`utils`, `stats`). On NixOS this file doesn't exist, so R starts
   but cannot load its own base packages, causing the Jupyter R kernel to crash
   in a loop with:
   ```
   package 'utils' in options("defaultPackages") was not found
   package 'stats' in options("defaultPackages") was not found
   ```

### The Solution

#### Enable nix-ld and install micromamba
#### Create the `/usr/bin/which` symlink
#### Set up the micromamba shell hook


nix-ld provides the dynamic linker that Conda binaries need. The `libraries` list
can stay empty — Conda packages ship their own shared libraries. Only add entries
if you later hit a specific `libXYZ.so: cannot open shared object file` error.

### Your config.nix should look like this
```
# nix-ld activation
  programs.nix-ld = {
    enable = true;
    libraries = with pkgs; [
      # Basics
      zlib
      zstd
      stdenv.cc.cc.lib    # libstdc++
      glibc
      openssl

      # For R
      curl
      libxml2
      libxslt
      bzip2
      xz
      libffi
      readline
      ncurses
      icu
      pcre2

      # Graphics stack (R plot devices, Cairo-based rendering)
      cairo
      pango
      harfbuzz
      fribidi
      freetype
      fontconfig
      libpng
      libjpeg
      libtiff
      libwebp
      pixman
      graphite2

      # X11 (R graphical output)
      xorg.libX11
      xorg.libXext
      xorg.libXrender
      xorg.libxcb
      xorg.libXau
      xorg.libXdmcp
      xorg.libICE
      xorg.libSM
      xorg.libXt

      # Jupyter / ZeroMQ
      zeromq
      libsodium

      # Scientific computing (GSL, BLAS)
      gsl
      openblas

      # Misc
      libssh2
      libuuid
      sqlite
    ];
  };

  system.activationScripts.usrBinWhich = ''
      mkdir -p /usr/bin
      ln -sf ${pkgs.which}/bin/which /usr/bin/which
  '';
  environment.systemPackages = with pkgs; [
    ...
    micromamba
  ];

```




#### Add this two lines directly to `~/.bashrc`

```
eval "$(micromamba shell hook --shell=bash)"
export MAMBA_ROOT_PREFIX="$HOME/.mamba"
```

which is empty by default on NixOS, that's normal.

#### Now apply and test

```bash
sudo nixos-rebuild switch
source ~/.bashrc

# Open a NEW terminal, then:
micromamba env create -f env.yaml
micromamba activate OMICS_r
R --vanilla -e "library(utils); library(stats); print('ok')"  # should print "ok"
R
IRkernel::installspec()  # run inside R, registers kernel with Jupyter
jupyter kernel --kernel=ir --debug
jupyter lab
```

### Tips for the environment.yaml

If you received a `.yaml` file from a colleague on Debian/Ubuntu, clean it up before use. If your colleague created the YAML file in Windows, find another colleague and remove additional the Windows-specific packages such as m2w64-*, pywin, pywinpty, ...:

- **Remove build hashes** (e.g., `=r41hc72bb7e_1004`) — these pin to the exact
  build on their machine and will cause `ResolvePackageNotFound` on yours
- **Remove compiler toolchain packages** (`gcc_impl_linux-64`, `gfortran_impl_linux-64`,
  `gxx_impl_linux-64`, `binutils_impl_linux-64`, `kernel-headers_linux-64`,
  `sysroot_linux-64`, `ld_impl_linux-64`) — Conda resolves these automatically
- **Remove low-level system libraries** (`libgcc-ng`, `libstdcxx-ng`, `zlib`,
  `libffi`, etc.) — these are transitive dependencies, pinning them causes conflicts
- **Remove the `prefix:` line** — it points to your colleague's home directory
- **Remove pip packages that duplicate conda packages** (`numpy`, `pandas`, etc.
  if they're already conda dependencies)

### Some Summary

| What | Where | Rebuild needed? |
|------|-------|-----------------|
| R, Python, R packages, Bioconda tools | `micromamba install ...` | No |
| Dynamic linker for Conda binaries | `programs.nix-ld.enable = true` | Once |
| `/usr/bin/which` for R | `system.activationScripts` | Once |
| micromamba shell hook | `.bashrc` or `configuration.nix` | Once |
| Missing system `.so` files (rare) | `programs.nix-ld.libraries` | Only if needed |

### Common errors and fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `cannot create directories: Read-only file system [/nix/store/...]` | `MAMBA_ROOT_PREFIX` not set | `export MAMBA_ROOT_PREFIX="$HOME/.mamba"` |
| `package 'utils' was not found` | `/usr/bin/which` missing | Add the symlink activation script |
| `Kernel does not exist` (Jupyter) | R kernel crashing on startup | Fix the R issue first, then `IRkernel::installspec()` |
| `activate is not available` | Shell hook not loaded | Open new terminal or `source ~/.bashrc` |
| `ResolvePackageNotFound` | Build hashes in `.yaml` | Remove build hashes from environment file |

## Acknowledgments

This guide was co-created with [Claude Opus 4.6] through an interactive debugging session on NixOS.
