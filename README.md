# SOWFA - OpenFAST installation notes

> [!WARNING] 
> This document contains my brief notes about how to compile and install _the old and deprecated_ NREL/SOWFA coupled with OpenFAST. **This software is not currently developed nor maintained and heavily depends on really old third party software.** 
> 
> For a more recent version of SOWFA please go to [https://github.com/NREL/SOWFA-6](https://github.com/NREL/SOWFA-6) 


## Requirements 


To be able to run SOWFA, you have compile its three main components: [**OpenFAST**](#openfast-compilation), [**OpenFOAM 2.4.x**](#openfoam-24x-compilation), and [**SOWFA**](#sowfa-compilation) itself.

But before doing it, you have to ensure that the machine that will compile all the software has the proper tools and extra software to do so. 
First of all you will need a couple of compilers. For the OpenFOAM parts, the GNU Compiler Collection is more than enough. But for the Fortran parts, like OpenFAST, the Intel Compiler suite is highly recommended. 

Also you will need a bunch of extra tools and libraries:
 
 * flex
 * bison
 * make 
 * cmake
 * git
 * An MPI library like OpenMPI, MPICH or IntelMPI, and its development package
 * A recent version of Boost Libraries and its development package
 * Zlib and its development  package
 * libreadline and its development package
 * libgmp and its development package
 * libmpfr and its development package
 * libcurses and its development package
 * libxml2 and its development package
 * libCGAL and its development package
 * HDF5 and its development package
 * Python and its development package
 * Yaml-cpp 
 * A BLAS/LAPACK library. Intel MKL is recommended here but libOpenBLAS can be used too

Normally all those packages can be directly downloaded from the repositories of your Linux distribution, so there is no need to download and compile them.
 
It's impossible to cover all the combinations of OS, package versions and environments, so this document assumes that the machine used to compile the code, runs an Ubuntu 16.04 LTS or Ubuntu 18.04, with OpenMPI as MPI library. If you use another OS, or another environment (like Environment Modules) you must adapt the commands to your specific requirements.

As we said, on an Ubuntu 16.04/18.04 machine all those packages can be installed simply typing:

```bash
sudo apt install build-essential flex bison gfortran git cmake python python-dev  \
    zlib1g-dev libreadline-dev libncurses-dev libyaml-cpp-dev libgmp-dev libmpfr-dev \
    libboost-system-dev libboost-thread-dev libopenmpi-dev openmpi-bin \
    libhdf5-dev libxml2-dev libcgal-dev libptscotch-dev libscotch-dev   
```


## OpenFAST compilation 

OpenFAST code is basically Fortran (2003 standard) with some parts in C++ (2011 standard) code, so a recent C++ and Fortran compiler is needed. After the tests we did, we can say that the best performance is achieved by using a recent version of the Intel Composer Compiler Suite. 

First download the OpenFAST code (version v2.3.0):

```bash
git clone https://github.com/OpenFAST/OpenFAST.git -b v2.3.0
```

Go to the OpenFAST directory, and create a *build* folder:

```bash
cd OpenFAST
mkdir build 
cd build

```


Then you have to declare the location of the libraries and headers of the different libraries needed by OpenFAST: libhdf5, yaml-cpp. 

For the HDF5 library, as we indicated in [**Requirements**](#requirements) section of this document, we used the *serial* version that comes with Ubuntu 16.04/18.04. Once installed, the libraries and development headers can be located in `/usr/lib/x86_64-linux-gnu/hdf5/serial`. 

The Yaml-cpp library was also installed through the package manager, and the root location of the libraries and headers is, therefore, `/usr/`.

We declare the location of all those paths through these two environment variables:

```bash
export HDF5_ROOT="/usr/lib/x86_64-linux-gnu/hdf5/serial"
export YAML_ROOT="/usr/"
```

Then we will launch the `cmake` configuration tool with a bunch of flags, depending on our preferences for the compilers, the Blas/Lapack libraries and the installation directory.

The first thing we have to do is choosing the compiler. For this, `cmake` has three flags:  `CMAKE_C_COMPILER`, `CMAKE_CXX_COMPILER` and `CMAKE_Fortran_COMPILER`. As we said, OpenFAST is mainly Fortran code so, the Fortran Intel Compiler is highly recommended. If you don't have a license for the Intel Compiler suite you can always use gcc/g++/gfortran.

Then you have to install the BLAS/LAPACK libraries to be used by OpenFAST. Our recommended option is using the Intel MKL libraries that can be downloaded for free from [Intel](https://software.intel.com/en-us/articles/installing-intel-free-libs-and-python-apt-repo). Once you have configured the apt repository from intel, just install the latest version of MKL available. For example:

```
sudo apt-get install intel-mkl-2019.3-062
```  

or, if you prefer to use the OpenBLAS implementation, just type:

```
sudo apt-get install libopenblas-dev
``` 

By default the OpenFAST `cmake` script will try to find and use the Intel MKL library. Otherwise it will use the OpenBLAS library. 


Finally we can launch the `cmake` command. As you can see in the example below, with the first 3 arguments we declare the compilers for C, C++ and Fortran that we want to use. If you use gcc/gfortran those lines are not needed, but, if you use a different compiler you should change those three lines accordingly. The C++ parts of OpenFAST need a 2011 C++ standard compliant compiler. For GCC, this means at least GCC 4.8.1.
The `DCMAKE_INSTALL_PREFIX` argument controls the destination path of the OpenFAST compilation. You have to put there the path to folder where you want to install OpenFAST. The flag `FPE_TRAP_ENABLED` must be ON, otherwise, SOWFA simulations will crash with a floating point exception during OpenFAST initialization (thanks to [@mchurchf](https://github.com/mchurchf), [@ewquon ](https://github.com/ewquon) and [@hjohlas](https://github.com/hjohlas) for their help with solving this)

The two final arguments are needed in order to couple OpenFAST with OpenFOAM. 


```bash
cmake \
    -DCMAKE_C_COMPILER=icc \
    -DCMAKE_CXX_COMPILER=icpc \
    -DCMAKE_Fortran_COMPILER=ifort \
    -DCMAKE_INSTALL_PREFIX="/path/to/wherever/you/want/to/install/OpenFAST" \
    -DFPE_TRAP_ENABLED=ON \
    -DBUILD_OPENFAST_CPP_API:BOOL=ON \
    -DBUILD_SHARED_LIBS:BOOL=ON \
    ../
```


Once the project is successfully configured, run the usual `make` and `make install`

```bash
make 
make install 
```


## OpenFOAM 2.4.x compilation 

For the moment, SOWFA is an OpenFOAM 2.4.x only application, so we must have available an installation of this version of OpenFOAM. 

First create the OpenFOAM installation folder, in our case it will be located at `${HOME}/OpenFOAM`:

```bash
# Create the OpenFOAM install folder
cd ${HOME} 
mkdir -p OpenFOAM
cd OpenFOAM
``` 


Now, *inside* the OpenFOAM installation folder, download OpenFOAM 2.4.x code, and the ThirdParty package. The official OpenFOAM 2.4.x code is quite old, and if it is compiled with a gcc version 6 or newer, you will get weird segmentation faults. A corrected version of this OpenFOAM code, that can be properly compiled with a recent version of gcc can be downloaded from:

```bash
# Download the OpenFOAM source code
git clone https://github.com/pablo-benito/OpenFOAM-2.4.x
git clone https://github.com/pablo-benito/ThirdParty-2.4.x.git
```

By default, OpenFOAM assumes that the installation path is going to be `${HOME}/OpenFOAM/`. If your installation folder is in a different location, you must edit the file  `OpenFOAM-2.4.x/etc/bashrc` and change the variable `foamInstall` so that it points to the required location.

The next step is optional, but recommended: since we prefer to use CGAL 4.7 that comes with Ubuntu instead of compiling the older version that comes in the ThirdParty package, we are going to configure OpenFOAM accordingly by running:

```bash 
sed -i -e 's/^\(cgal_version=\).*/\1cgal-system/' OpenFOAM-2.4.x/etc/config/CGAL.sh
```


Now, before we can build OpenFOAM, we need to do a few fixes. OpenFOAM 2.4.x do not properly detect modern versions of Flex, so we have to edit some files:

```bash
source ${HOME}/OpenFOAM/OpenFOAM-2.4.x/etc/bashrc
cd ${WM_PROJECT_DIR}
find src applications -name "*.L" -type f | xargs \
    sed -i -e 's=\(YY\_FLEX\_SUBMINOR\_VERSION\)=YY_FLEX_MINOR_VERSION < 6 \&\& \1='
```


In order to reduce the OpenFOAM compilation time, we can configure the `wmake` utility to use more than one core. In our case, we set it up to 4 cores with:

```bash
export WM_NCOMPPROCS=4
```

Finally we run the compilation:

```bash
cd ${WM_PROJECT_DIR}
./Allwmake
```


## SOWFA compilation 

Before doing anything we must load our OpenFOAM environment by:

```bash
source ${HOME}/OpenFOAM/OpenFOAM-2.4.x/etc/bashrc
```

Then, we download the SOWFA code:

```bash
# Download SOWFA code into ${HOME}
cd ${HOME}
git clone https://github.com/NREL/SOWFA.git
cd SOWFA
```

At the time of writing this document, there is a couple of changes that we must do in the OpenFAST SOWFA code. First of all, we must add the include paths and lib path to the HDF5 library in certain files. Also, starting with OpenFAST version 1.0, there is an extra library, `versioninfolib`, that must be included as a dependency in some SOWFA binaries.

To solve all those issues we must  edit certain `options` files. The first one is located at:

```
applications/solvers/incompressible/windEnergy/pisoFoamTurbine.ALMAdvancedOpenFAST/Make/options 
``` 

Inside that file, add `-I$(HDF5_DIR)/include` at the end of the `EXE_INC` variable, so that it looks like this:

```bash
EXE_INC = \
    -I$(LIB_SRC)/turbulenceModels/incompressible/turbulenceModel \
    -I$(LIB_SRC)/transportModels \
    -I$(LIB_SRC)/transportModels/incompressible/singlePhaseTransportModel \
    -I$(LIB_SRC)/finiteVolume/lnInclude \
    -I$(LIB_SRC)/meshTools/lnInclude \
    -I$(LIB_SRC)/fvOptions/lnInclude \
    -I$(LIB_SRC)/sampling/lnInclude \
    -I$(SOWFA_DIR)/src/turbineModels/turbineModelsOpenFAST/lnInclude \
      $(PFLAGS) \
      $(PINC) \
    -I$(OPENFAST_DIR)/include \
    -I$(HDF5_DIR)/include # <--- Add this line. Add a backslash in the previous line if needed

``` 

Also, inside that file, in the `EXE_LIBS` variable, add the dependency to the new OpenFAST library, `versioninfolib`, 

```bash

    -lsctypeslib \
    -lservodynlib \
    -lsubdynlib \
    -lversioninfolib \  # <--- Add this line 
     $(BLASLIB) \
    -L$(HDF5_DIR)/lib/ \
    -lhdf5_hl \
    -lhdf5 
```

Then open the next `options` file:
```bash
applications/solvers/incompressible/windEnergy/windPlantSolver.ALMAdvancedOpenFAST/Make/options
```

and, again, add the dependency to the  `versioninfolib` library, inside the `EXE_LIBS` variable:

```bash
    -lsubdynlib \
    -lversioninfolib \ # <-- Add this line 
     $(BLASLIB) \
    -L$(HDF5_DIR)/lib/ \
    -lhdf5_hl \

```

The last file you need to edit, is:

```bash
src/turbineModels/turbineModelsOpenFAST/Make/options
``` 

Here you  have to add the paths to the HDF5 libraries, both in the EXE_INC and in the LIB_LIBS variable:

```bash 
 # Inside the EXE_INC variable 
    -I$(LIB_SRC)/transportModels \
     $(PFLAGS) \
     $(PINC) \
    -I$(OPENFAST_DIR)/include \ # <--  Add this backslash 
    -I$(HDF5_DIR)/include  # >-- and add this line
```

and, in the same file, but in the LIB_LIBS variable:
```bash 
 # Inside the LIB_LIBS variable 
    -lsctypeslib \
    -lservodynlib \
    -lsubdynlib \  # <-- Add this backslash
    -L$(HDF5_DIR)/lib \ # and add the next three lines 
    -lhdf5 \
    -lhdf5_hl
```

Once finished editing the `options` files, we must declare certain paths with environment variables:

First, we export the location of the OpenFAST installation directory. You must change the path to wherever you installed OpenFAST:

```bash
export OPENFAST_DIR="/path/to/wherever/you/installed/OpenFAST"
``` 

Also, we export the location of the HDF5 installation directory. You should change the path to wherever you have installed the HDF5 library. It must be a directory that contains an `include` directory with the headers, and a `lib` directory with the libraries. On Ubuntu 16.04 and Ubuntu 18.04 this should be:

```bash
export HDF5_DIR="/usr/lib/x86_64-linux-gnu/hdf5/serial"
``` 

Finally, we must declare the path to wherever you downloaded SOWFA inside the `SOWFA_DIR` environment variable, in our example it was `${HOME}/SOWFA`:
```bash 
export SOWFA_DIR="${HOME}/SOWFA"
```

Then, we go to the SOWFA source directory and launch the SOWFA compilation with:
```bash
cd ${SOWFA_DIR}
./Allwmake
``` 

The resulting binaries will be at `${SOWFA_DIR}/applications/bin/${WM_OPTIONS}`:
```bash
ls ${SOWFA_DIR}/applications/bin/${WM_OPTIONS}

 ABLSolver                            ABLTerrainSolver 
 pisoFoamTurbine.ADM                  pisoFoamTurbine.ALM 
 pisoFoamTurbine.ALMAdvanced          pisoFoamTurbine.ALMAdvancedOpenFAST 
 setFieldsABL                         turbineTestHarness.ALM 
 turbineTestHarness.ALMAdvanced       windPlantSolver.ADM 
 windPlantSolver.ALM                  windPlantSolver.ALMAdvanced
 windPlantSolver.ALMAdvancedOpenFAST 
``` 

and the associated libraries will be inside `${SOWFA_DIR}/lib/${WM_OPTIONS}/`:

```bash
ls ${SOWFA_DIR}/lib/${WM_OPTIONS}/

 libSOWFATurbineModelsOpenFAST.so  libSOWFATurbineModelsStandard.so
 libSOWFAfiniteVolume.so           libSOWFAincompressibleLESModels.so
 libSOWFAsampling.so               libSOWFArutilityFunctionObjects.so
 libSOWFAfileFormats.so
```


## Runtime configuration 

As we have seen, SOWFA binaries depends on a wide group of libraries: The OpenFOAM core libraries, the OpenFAST libraries, the Blas/Lapack libraries, the HDF5 libraries and its own group of libraries. In order to be able to run the SOWFA utilities, we must be sure that *all* needed libraries are visible to the binaries at runtime. 

First we must load the *OpenFOAM 2.4.x environment*: 

```bash
source [INSTALL_PREFIX_FOR_OPENFOAM_2.4.x]/etc/bashrc
```

This ensures that the OpenFOAM core libraries are visible to our binaries. 

For the other groups of libraries, we need to append their locations to the `${LD_LIBRARY_PATH}` variable. 
We will start with the OpenFAST libraries. If the root installation path of OpenFAST is pointed by the variable `${OPENFAST_DIR}`, then the libraries are located in the `${OPENFAST_DIR}/lib` folder.  

```bash
export LD_LIBRARY_PATH=${OPENFAST_DIR}/lib:${LD_LIBRARY_PATH}
```

If the location of the *Blas/Lapack libraries* is not a *default* one, you must also add it to your `${LD_LIBRARY_PATH}` variable. 
For example, our Intel MKL libraries are located inside the folder `/opt/intel2018/mkl/lib/intel64/`, so we did:

```bash
export LD_LIBRARY_PATH=/opt/intel2018/mkl/lib/intel64/:${LD_LIBRARY_PATH}
```

Also, you must declare the location of the SOWFA libraries, so the SOWFA binaries can find them at runtime. As usual, you must append that location to the LD_LIBRARY_PATH variable:
 
```bash
export LD_LIBRARY_PATH=${SOWFA_DIR}/lib/${WM_OPTIONS}/:${LD_LIBRARY_PATH}
```

## Citation

Did you like my notes? Please cite! 
[![DOI](https://zenodo.org/badge/185154334.svg)](https://zenodo.org/badge/latestdoi/185154334)


## References

 * [SOWFA github](https://github.com/NREL/SOWFA.git)
 * [OpenFAST](http://openfast.readthedocs.io)
 * [OpenFAST installation docs](http://openfast.readthedocs.io/en/master/source/install/index.html)
 * [OpenFOAM installation](https://openfoamwiki.net/index.php/Installation/Linux/OpenFOAM-2.4.0/Ubuntu#Ubuntu_16.04)
