# `c2s2-toolchain`

C2S2's custom toolchain for chip development, built and managed using [EasyBuild](https://easybuild.io/)

<p align="center">
<img src="assets/toolchain.png" alt="C2S2's Logo in a blue gear" width="200"/>
</p>

## Installation

The only requirement is that EasyBuild is installed (4.9.0+) (as well as that the system has an environment module tool, the one preferred by EasyBuild being [Lmod](https://lmod.readthedocs.io/en/latest/)). There are a [variety of methods](https://tutorial.easybuild.io/2023-eb-eessi-uk-workshop/easybuild-installation/). On C2S2's server, we built Easybuild as a separate module, noting the [additional configurations](https://docs.easybuild.io/configuration/#modules_tool) needed to use `EnvironmentModulesC` as the modules tool:

```bash
# Define installation prefix, and install EasyBuild into it
export EB_TMPDIR=/tmp/$USER/eb_tmp
export EB_DIR=/classes/c2s2/easybuild
python3.8 -m pip install --ignore-installed --prefix $EB_TMPDIR easybuild

# Update environment to use this temporary EasyBuild installation
export PATH=$EB_TMPDIR/bin:$PATH
export PYTHONPATH=$EB_TMPDIR/lib/python3.11/site-packages/:$PYTHONPATH
export EB_PYTHON=python3.11

# Install EasyBuild module in C2S2's directory
eb --install-latest-eb-release --prefix $EB_DIR --modules-tool=EnvironmentModules --module-syntax=Tcl
```

**Note**: EasyBuild was installed using Anaconda's Python 3.8, sourced using `module load anaconda3`. If you are using a different version of Python, change the prompts accordingly. Here, we assume the tree:

```text
$EB_TMPDIR
 |-- bin
 |-- contrib
 |-- easybuild
 |-- etc
 `-- lib
      `-- python3.8
           `-- site-packages
                |-- easybuild
                ...
```

However, it's common for other trees to look different; below is another example installation tree, and the appropriate commands to run instead

```text
$EB_TMPDIR
 `-- local
      |-- bin
      |-- contrib
      |-- easybuild
      |-- etc
      `-- lib
           `-- python3.11
                `-- dist-packages
                     |-- easybuild
                     ...
```

In this case, you would replace the commands after the temporary installation with

```bash
export PATH=$EB_TMPDIR/local/bin:$PATH
export PYTHONPATH=$EB_TMPDIR/local/lib/python3.11/dist-packages/:$PYTHONPATH
export EB_PYTHON=python3.11
```

## Sourcing

Once EasyBuild is built as a module, we can inform the module tool of its location, then source the module

```bash
module use $EB_DIR/modules/all
module load EasyBuild
```

This will need to be done every time you log into a new shell; as an alternative, you may wish to add the following to a `.bashrc` (replacing `$EB_DIR` with the directory you selected above):

```bash
export MODULEPATH="$EB_DIR:$MODULEPATH"
module load EasyBuild
```

## Configuring

Configuring your EasyBuild system is also important to tailor it to your system. However, it can be overwhelming; EasyBuild is made to have very little hard-coded, and as such has around 275 configurations.

Configuration is most easily done through configuration files. However, EasyBuild only recognizes [specific locations](https://docs.easybuild.io/configuration/#configuration_file); these can be shown with

```bash
eb --show-default-configfiles
```

For C2S2, the system-level location was `/etc/xdg/easybuild.d/*.cfg`. This location was only writeable by `root`, so as to ensure easier customization, we included a symlink here to a configuration file in `/classes/c2s2/easybuild`

To create our configuration file, we started with the output of `eb --confighelp`. From here, we uncommented/included the following sections:

```ini
[config]
# ...
module-syntax=Tcl # Support Tcl modules - not needed if you use Lmod
modules-tool=EnvironmentModulesC # Specify the module tool used on C2S2's server
prefix=/classes/c2s2/easybuild # Have a global path for installations and building
# ...
[override]
#...
experimental=True # Allow for Easystack files
#...
```

You can verify that these changes took effect by running `eb --show-config` to show all of your current configurations

## Installing Packages

Before installing anything else, if you are on C2S2's server, the version of OpenSSL is so low that EasyBuild cannot install a wrapper. To remedy this, use the custom `easystack` for server-specific dependencies:

```bash
eb --easystack c2s2-dev.yaml --robot
```

From there, install the software you want based on the corresponding `easystack` file. For instance, if I wanted to build the software in `general`, I would run:

```bash
eb --easystack general.yaml --robot=./dependencies:./patches --include-easyblocks=./easyblocks/klayout.py,./easyblocks/riscvgnutoolchain.py,./easyblocks/dtc.py,./easyblocks/sky130.py
```

- `--robot` indicates that EasyBuild should automatically install dependencies as well (EasyBuild doesn't install dependencies by default), as well as indicates paths that EasyBuild uses to find dependency `easyconfig` files and any patches for packages. This includes custom dependencies that C2S2 has made, as well as those not included in a release of [`easybuild-easyconfigs`](https://github.com/easybuilders/easybuild-easyconfigs) yet
- `include-easyblocks=./easyblocks/klayout.py` indicates that we have some custom EasyBlock implemented in `klayout.py` (similar for `riscv-gnu-toolchain.py`)

Note that building these files involves a large amount of file space, despite each build being cleaned up after its (successful) completion. The most utilization is for the RISCV GNU Toolchain, about 8GB; if your `easybuild` directory doesn't have the space for this, another build directory can be specified with `--buildpath`

# Using Packages

Packages are installed in `$EB_DIR/software`, and are sourced using environment modules located in `$EB_DIR/modules`. Provided that you have used either the `module use` command or edited your `MODULEPATH` environment variable from above, they can be loaded using the appropriate module. For instance, to use a new version of Git, you would run

```bash
module load git
```

This will modify the necessary environment variables (`PATH`, etc.) to use Git from the shell, as well as sourcing any ohter modules for packages identified as runtime dependencies.
