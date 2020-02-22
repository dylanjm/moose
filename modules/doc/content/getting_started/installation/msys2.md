# MSYS2

Install MSYS2 following the instructions on the [MSYS2 homepage](https://www.msys2.org/).
Choose the 64bit version, keep the suggested install location (or you will have to adapt below).
Once the two `pacman` runs are completed make sure you open the `MINGW64` terminal
and use it for all further steps.

## prerequisites

### MS MPI and toolchain

Git, compilers, and MPI are installed using packages.

```
pacman -S base-devel binutils git mingw-w64-x86_64-toolchain
pacman -S mingw-w64-x86_64-msmpi
```

Next download and install `msmpisetup.exe` (not `msmpisdk.msi`!) from Microsoft.
Stick with the default installation path (else you will have to modify the `export` below).
Pick version that matches the msys2 msmpi sdk package version. As of the time of
writing this, it is MS MPI 10. Add the MS MPI runtime tools to your path (add this to `~/.bashrc`)

```
export PATH=$PATH:/c/Program\ Files/Microsoft\ MPI/Bin/
```

### Metis, Parmetis, Blas

We will use MSYS2 packages for those PETSc dependencies as downloading and compiling
from the PETSc config script does not work (the MSYS2 packages includes some minor patches to
make everything work).

```
pacman -S mingw-w64-x86_64-metis mingw-w64-x86_64-parmetis mingw-w64-x86_64-openblas
```

### MOOSE repository

Create a `projects` directory and change into it

```
mkdir ~/projects
cd ~/projects
```

Clone moose and initialize its submodules

```
git clone https://github.com/idaholab/moose
cd moose
git submodule update --init --recursive
```

Add the `MOOSE_DIR` environment variable to your `~/.bashrc` config and reload the
bash configuration (make sure you are the `moose` directory when you do this!)

```
echo "export MOOSE_DIR='`pwd`'" >> ~/.bashrc
. ~/.bashrc
```

# PETSc

Next we build PETSc. We have prepared a special configuration script for PETSc
to make sure we use the preinstalled prerequisites and leave off optional features
that may cause the build to fail.

```
cd $MOOSE_DIR
./scripts/petsc_msys2_config.py
```

Follow the instructions printed at the end of the configure output on how to run make.

Once PETSc is built do **not** run the `make install` command. We'll leave the built
PETSc where it is and add two exports for the `PETSC_DIR` and `PETSC_ARCH` to your `~/.bashrc`.

**IMPORTANT:** make sure you prepend `C:/msys64` to the `PETSC_DIR`. The lines in your
`~/.bashrc` should look something like this:

```
export PETSC_DIR=C:/msys64/home/`whoami`/projects/moose/petsc
export PETSC_ARCH=msys2
```

### PETSc `make getlinklibs`

`make getlinklibs` is used by libMesh to gather a set of libraries and library
directories that PETSc config already discovered for us. Unfortunately it is a bit buggy.

Edit the file  `$PETSC_DIR/$PETSC_ARCH/lib/petsc/conf/petscvariables`
and remove the strings `/C:msys64mingw64lib` and `-LC:msys64mingw64lib` (the first one is attached
to the end of a path that you will need to keep!). And make sure the paths in
`PETSC_WITH_EXTERNAL_LIB` all use forward slashes.

## libMesh

Fix missing symlinks

```
cd $MOOSE_DIR/libmesh
./contrib/bin/fix_windows_symlinks.sh
```

We will build the `opt` version only here. You can instead provide `METHODS='opt dbg'`
(note the `S`), but you may need to take extra measures building TMPI if you want to also
build the `oprof` method (i.e. passing the `--with-methods="..."` argument to TIMPI
configure).

Start the configure and build process with

```
cd $MOOSE_DIR
METHOD=opt ./scripts/update_and_rebuild_libmesh.sh --disable-netcdf
```

This will take a few minutes and you should finally error out with make complaining
about a missing `metaphysicl` directory.

Something is not quite right in the libMesh configure scripts, resulting in the
build directories for the two checked out submodules for MetaPhysicL and TIMPI
not getting created and configured.

We will do this manually by running

```
mkdir ~/projects/moose/libmesh/build/contrib/metaphysicl
cd ~/projects/moose/libmesh/build/contrib/metaphysicl
../../../contrib/metaphysicl/configure --enable-shared=no
make
```

Now configure and build TIMPI. We disable the automatic option fetching from PETSc
as it does not quite work on MSYS2 (some paths contain backslashes and result in
compile errors). We also need to disable shared libraries to suppress a libtool error
demanding the use of `-no-undefined`.

```
mkdir ~/projects/moose/libmesh/build/contrib/timpi
cd ~/projects/moose/libmesh/build/contrib/timpi
../../../contrib/timpi/configure --disable-petsc --enable-shared=no
make
```

Now back out to the main libmesh build directory and continue the build process

```
cd ../..
make -j 2
```

libtool may complain `error: Continuing, but uninstalled executables may not work.`,
which can be ignored. However if make is complaining about not being able to
find `C:msys64mingw64lib` (note the missing directory slashes) you need to go into the
`Makefile` and edit out every occurrence of this path (there should be about four).

Once compilation is finished install libMesh using

```
make install
```

## MOOSE

Now it is finally time to build MOOSE. We'll start with the framework and the
tests. MSYS2 does not support symlinks, so we need to disable `MOOSE_HEADER_SYMLINKS`.

```
cd $MOOSE_DIR/test
MOOSE_HEADER_SYMLINKS=false MOOSE_UNITY=false make
```
