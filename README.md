[README.md](https://github.com/user-attachments/files/33594074/README.md)
# Mininet Installation on Ubuntu 24.04

This document summarizes the changes and commands used to install
Mininet successfully on Ubuntu 24.04.3 LTS.

## Environment

-   **Operating system:** Ubuntu 24.04.3 LTS
-   **Mininet version:** 2.3.1b4
-   **Installation source:** Mininet Git repository
-   **Main issue addressed:** Python's PEP 668
    `externally-managed-environment` restriction

## 1. Obtain the Mininet source

If the repository has not already been cloned:

``` bash
sudo apt update
sudo apt install -y git
git clone https://github.com/mininet/mininet.git
cd mininet
```

If `~/mininet` already exists, do not clone it again. Use the existing
repository:

``` bash
cd ~/mininet
```

## 2. Changes in `util/install.sh`

Edit the installer:

``` bash
nano ~/mininet/util/install.sh
```

### 2.1 Install `pexpect` using the system package manager

In the dependency-installation section, use:

``` bash
$install ${PYPKG}-pexpect
```

This installs the appropriate `pexpect` package through the
distribution's package manager rather than trying to install it directly
into the system Python environment with `pip`.

### 2.2 Ubuntu `pep8` compatibility

The script uses this Ubuntu release condition to select `pycodestyle`
for Ubuntu 23.10 and newer:

``` bash
if [ "$DIST" = "Ubuntu" -a `expr $RELEASE '>=' 23.10` = "1" ]; then
    pep8=pycodestyle
fi
```

This was the condition present in the working installation script. The
release comparison is dependent on the way `expr` compares the supplied
values, so review it if adapting the script for other release-number
formats.

### 2.3 Keep the Mininet core installation commands

The installer calls the Makefile to install Mininet and its manual
pages:

``` bash
echo "Installing Mininet core"

pushd $MININET_DIR/mininet
sudo PYTHON=${PYTHON} make install-mnexec
sudo PYTHON=${PYTHON} make install-manpages
sudo PYTHON=${PYTHON} make install
popd
```

The pip commands causing the PEP 668 error are in the repository's
top-level `Makefile`, so the required pip change is made there.

## 3. Change the top-level `~/mininet/Makefile`

**Important:** The Makefile modified for this fix is
`~/mininet/Makefile`, not `~/mininet/mininet/Makefile`.

Open it:

``` bash
cd ~/mininet
nano Makefile
```

Find the `install` target. The original commands were:

``` make
install: install-mnexec install-manpages
#   This seems to work on all pip versions
    $(PYTHON) -m pip uninstall -y mininet || true
    $(PYTHON) -m pip install .
```

Change the two pip commands to:

``` make
install: install-mnexec install-manpages
#   This seems to work on all pip versions
    $(PYTHON) -m pip uninstall -y mininet --break-system-packages || true
    $(PYTHON) -m pip install . --break-system-packages
```

Keep the leading **tab characters** before the two command lines;
Makefiles require tabs for recipe commands.

The `--break-system-packages` option allows pip to modify the system
Python environment despite PEP 668. Use it deliberately: it bypasses
Ubuntu's protection and may cause conflicts with packages managed by
`apt`.

The `develop` target was not needed for this installation and was left
unchanged.

## 4. Run the installer

The installation was run using the `-n` option, which installs Mininet's
dependencies and core components rather than all optional components:

``` bash
cd ~/mininet
sudo ./util/install.sh -n
```

If the installer reports that a Git destination such as `openflow`
already exists, do not delete it automatically. Check the directory and
the installer's current state before deciding how to proceed.

## 5. Successful installation output

The installation reached these success messages:

``` text
Successfully built mininet
Successfully installed mininet-2.3.1b4
```

It also installed the executable and manual pages:

``` text
install -D mnexec /usr/bin/mnexec
install -D -t /usr/share/man/man1 mn.1 mnexec.1
```

The following warnings were observed and were not fatal in this run:

``` text
WARNING: Skipping mininet as it is not installed.
WARNING: Running pip as the 'root' user can result in broken permissions...
```

The first warning occurred because the Makefile attempted to uninstall
Mininet before installing it, but no existing pip-installed Mininet
package was present. The second is pip's general warning about running
as root.

## 6. Verify the installation

Check the installed version:

``` bash
mn --version
```

Then run Mininet's basic connectivity test:

``` bash
sudo mn --test pingall
```

A successful test should report that no packets were dropped (for
example, `Results: 0% dropped`). If either command fails, inspect the
error output before making further changes.

## Summary of changes

  -----------------------------------------------------------------------------
  File                    Change                        Purpose
  ----------------------- ----------------------------- -----------------------
  `util/install.sh`       Install `pexpect` through     Use the system package
                          `$install ${PYPKG}-pexpect`   manager

  `util/install.sh`       Select `pycodestyle` for      Compatibility with
                          Ubuntu 23.10+ using the       newer Ubuntu releases
                          existing release condition    

  `Makefile`              Add `--break-system-packages` Avoid the PEP 668
                          to the `pip uninstall`        refusal
                          command in the `install`      
                          target                        

  `Makefile`              Add `--break-system-packages` Permit the Mininet
                          to the `pip install .`        package installation
                          command in the `install`      
                          target                        
  -----------------------------------------------------------------------------

## Final result

Mininet **2.3.1b4** was successfully installed on Ubuntu 24.04.3 LTS
using the above changes. Run `mn --version` and `sudo mn --test pingall`
to confirm that the installation is still working on the current
machine.
