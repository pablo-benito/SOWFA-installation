# SOWFA - OpenFAST installation notes

This document contains a brief notes about how to compile and install NREL/SOWFA coupled with OpenFAST. 



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
 
It's impossible to cover all the combinations of OS, package versions and environments, so this document assumes that the machine used to compile the code, runs an Ubuntu 16.04.3 LTS, with OpenMPI as MPI library. If you use another OS, or another environment (like Environment Modules) you must adapt the commands to your specific requirements.

As we said, on a Ubuntu 16.04.3 LTS machine all those packages can be installed simply typing:

```bash
$ sudo apt install build-essential flex bison gfortran git cmake python python-dev  \
    zlib1g-dev libreadline-dev libncurses-dev libyaml-cpp-dev libgmp-dev libmpfr-dev \
    libboost-system-dev libboost-thread-dev libopenmpi-dev openmpi-bin \
    libhdf5-dev libxml2-dev    
```


## OpenFAST compilation 

OpenFAST code is basically Fortran (2003 standard) with some parts in C++ (2011 standard) code, so a recent C++ and Fortran compiler is needed. After the tests we did, we can say that the best performance is achieved by using a recent version of the Intel Composer Compiler Suite. 

First download the OpenFAST code:

```bash
$ git clone https://github.com/OpenFAST/OpenFAST.git
```

Go to the OpenFAST directory, and create a *build* folder:

```bash
$ cd OpenFAST
$ mkdir build 
$ cd build

```


Then you have to declare the location of the libraries and headers of the different libraries needed by OpenFAST: libhdf5, yaml-cpp. 

For the HDF5 library, as we indicated in [**Requirements**](#requirements) section of this document, we used the *serial* version that comes with Ubuntu 16.04.3 LTS. Once installed, the libraries and development headers can be located in `/usr/lib/x86_64-linux-gnu/hdf5/serial`. 

The Yaml-cpp library was also installed through the package manager, and the root location of the libraries and headers is, therefore, `/usr/`.

We declare the location of all those paths through these two environment variables:

```bash
$ export HDF5_ROOT="/usr/lib/x86_64-linux-gnu/hdf5/serial"
$ export YAML_ROOT="/usr/"
```

Then we will launch the `cmake` configuration tool with a bunch of flags, depending on our preferences for the compilers, the Blas/Lapack libraries and the installation directory.

The first thing we have to do is choosing the compiler. For this, `cmake` has three flags:  `CMAKE_C_COMPILER`, `CMAKE_CXX_COMPILER` and `CMAKE_Fortran_COMPILER`. As we said, OpenFAST is mainly Fortran code so, the Fortran Intel Compiler is highly recommended. If you don't have a license for the Intel Compiler suite you can always use gcc/g++/gfortran.

Then you have to install the BLAS/LAPACK libraries to be used by OpenFAST. Our recommended option is using the Intel MKL libraries that can be downloaded for free from [Intel](https://software.intel.com/en-us/articles/installing-intel-free-libs-and-python-apt-repo). Once you have configured the apt repository from intel, just install the latest version of MKL available. For example:

```
$ sudo apt-get install intel-mkl-2019.3-062
```  

or, if you prefer to use the OpenBLAS implementation, just type:

```
$ sudo apt-get install openblas-dev
``` 

By default the OpenFAST `cmake` script will try to find and use the Intel MKL library. Otherwise it will use the OpenBLAS library. 


Finally we can launch the `cmake` command. As you can see in the example below, with the first 3 arguments we declare the compilers for C, C++ and Fortran that we want to use. If you use gcc/gfortran those lines are not needed, but, if you use a different compiler you should change those three lines accordingly. The C++ parts of OpenFAST need a 2011 C++ standard compliant compiler. For GCC, this means at least GCC 4.8.1.
The `DCMAKE_INSTALL_PREFIX` argument controls the destination path of the OpenFAST compilation. You have to put there the path to folder where you want to install OpenFAST. The flag `FPE_TRAP_ENABLED` must be ON, otherwise, SOWFA simulations will crash with a floating point exception during OpenFAST initialization (thanks to [@mchurchf](https://github.com/mchurchf), [@ewquon ](https://github.com/ewquon) and [@hjohlas](https://github.com/hjohlas) for their help with solving this)

The two final arguments are needed in order to couple OpenFAST with OpenFOAM. 


```bash
$ cmake \
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
$ make 
$ make install 
```


## OpenFOAM 2.4.x compilation 

For the moment, SOWFA is an OpenFOAM 2.4.x only application, so we must have available an installation of this version of OpenFOAM. 

First create the OpenFOAM installation folder, in our case it will be located at `${HOME}/OpenFOAM`:

```bash
# Create the OpenFOAM install folder
$ cd ${HOME} 
$ mkdir -p OpenFOAM
$ cd OpenFOAM
``` 


Now, *inside* the OpenFOAM installation folder, download OpenFOAM 2.4.x code, and the ThirdParty package. The official OpenFOAM 2.4.x code is quite old, and if it is compiled with a gcc version 6 or newer, you will get weird segmentation faults. A corrected version of this OpenFOAM code, that can be properly compiled with a recent version of gcc can be downloaded from:

```bash
# Download the OpenFOAM source code
$ git clone https://github.com/pablo-benito/OpenFOAM-2.4.x
$ git clone https://github.com/pablo-benito/ThirdParty-2.4.x.git
```

By default, OpenFOAM assumes that the installation path is going to be `${HOME}/OpenFOAM/`. If your installation folder is in a different location, you must edit the file  `OpenFOAM-2.4.x/etc/bashrc` and change the variable `foamInstall` so that it points to the required location.

The next step is optional, but recommended: since we prefer to use CGAL 4.7 that comes with Ubuntu instead of compiling the older version that comes in the ThirdParty package, we are going to configure OpenFOAM accordingly by running:

```bash 
$ sed -i -e 's/^\(cgal_version=\).*/\1cgal-system/' OpenFOAM-2.4.x/etc/config/CGAL.sh
```


Now, before we can build OpenFOAM, we need to do a few fixes. OpenFOAM 2.4.x do not properly detect modern versions of Flex, so we have to edit some files:

```bash
$ source ${HOME}/OpenFOAM/OpenFOAM-2.4.x/etc/bashrc
$ cd ${WM_PROJECT_DIR}
$ find src applications -name "*.L" -type f | xargs \
    sed -i -e 's=\(YY\_FLEX\_SUBMINOR\_VERSION\)=YY_FLEX_MINOR_VERSION < 6 \&\& \1='
```


In order to reduce the OpenFOAM compilation time, we can configure the `wmake` utility to use more than one core. In our case, we set it up to 4 cores with:

```bash
$ export WM_NCOMPPROCS=4
```

Finally we run the compilation:

```bash
$ cd ${WM_PROJECT_DIR}
$ ./Allwmake
```


## SOWFA compilation 

Before doing anything we must load our OpenFOAM environment by:

```bash
$ source ${HOME}/OpenFOAM/OpenFOAM-2.4.x/etc/bashrc
```

Then, we download the SOWFA code and switch to the OpenFAST branch:

```bash
# Download SOWFA code into ${HOME}
$ cd ${HOME}
$ git clone https://github.com/NREL/SOWFA.git
$ cd SOWFA
```

At the time of writing this document, there is a change that we must do in the OpenFAST SOWFA code: On Ubuntu 16.04, the HDF5 header files are not in a gcc compiler `include` default path, and a couple of applications will fail to build, because `wmake` will not be able to find them. The issue can be easily solved by editing the `options` file located at:

```
applications/solvers/incompressible/windEnergy/pisoFoamTurbine.ALMAdvancedOpenFAST/Make/
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
    -I$(HDF5_DIR)/include # Add this line. Add a backslash in the previous line if needed

``` 


Then, we export the location of the OpenFAST installation directory. You must change the path to wherever you installed OpenFAST:

```bash
$ export OPENFAST_DIR="/path/to/wherever/you/installed/OpenFAST"
``` 


Also, we export the location of the HDF5 installation directory. You should change the path to wherever you have installed the HDF5 library. On Ubuntu 16.04 this should be:

```bash
$ export HDF5_DIR="/usr/lib/x86_64-linux-gnu/hdf5/serial"
``` 


Finally, we go to the SOWFA root directory and launch the SOWFA compilation with:
```bash
$ cd ${HOME}/SOWFA
$ WM_PROJECT_USER_DIR=${PWD} ./Allwmake
``` 

The resulting binaries will be at `${FOAM_USER_APPBIN}`:
```bash
$ ls ${FOAM_USER_APPBIN}
```
```shell
 ABLSolver                            ABLTerrainSolver 
 pisoFoamTurbine.ADM                  pisoFoamTurbine.ALM 
 pisoFoamTurbine.ALMAdvanced          pisoFoamTurbine.ALMAdvancedOpenFAST 
 setFieldsABL                         turbineTestHarness.ALM 
 turbineTestHarness.ALMAdvanced       windPlantSolver.ADM 
 windPlantSolver.ALM                  windPlantSolver.ALMAdvanced
 windPlantSolver.ALMAdvancedOpenFAST 
``` 

and the associated libraries will be inside `${FOAM_USER_LIBBIN}`:

```bash
$ ls ${FOAM_USER_LIBBIN}
``` 
```
 libSOWFATurbineModelsOpenFAST.so  libSOWFATurbineModelsStandard.so
 libSOWFAfiniteVolume.so           libSOWFAincompressibleLESModels.so
 libSOWFAsampling.so               libSOWFArutilityFunctionObjects.so
 libSOWFAfileFormats.so
```


## Runtime configuration 

As we have seen, SOWFA binaries depends on a wide group of libraries: The OpenFOAM core libraries, the OpenFAST libraries, the Blas/Lapack libraries, the HDF5 libraries and its own group of libraries. In order to be able to run the SOWFA utilities, we must be sure that *all* needed libraries are visible to the binaries at runtime. 

First we must load the *OpenFOAM 2.4.x environment*: 

```bash
$ source [INSTALL_PREFIX_FOR_OPENFOAM_2.4.x]/etc/bashrc
```

This ensures that the OpenFOAM core libraries are visible to our binaries. 

For the other groups of libraries, we need to append their locations to the `${LD_LIBRARY_PATH}` variable. 
We will start with the OpenFAST libraries. If the root installation path of OpenFAST is pointed by the variable `${OPENFAST_DIR}`, then the libraries are located in the `${OPENFAST_DIR}/lib` folder.  

```bash
$ export LD_LIBRARY_PATH=${OPENFAST_DIR}/lib:${LD_LIBRARY_PATH}
```

If the location of the *Blas/Lapack libraries* is not a *default* one, you must also add it to your `${LD_LIBRARY_PATH}` variable. 
For example, our Intel MKL libraries are located inside the folder `/opt/intel2018/mkl/lib/intel64/`, so we did:

```bash
$ export LD_LIBRARY_PATH=/opt/intel2018/mkl/lib/intel64/:${LD_LIBRARY_PATH}
```


Also, if for any reason the path `${FOAM_USER_LIBBIN}` during runtime is different from the one that you used during compilation, the SOWFA applications will not be able to start. This can happen for several reasons: if the user that compiled the code is different from the one that runs the code each one will see a different path in the variable `${FOAM_USER_LIBBIN}`. Another possibility is that you need to move the libraries to a different location. In any case, you should declare the new location of the SOWFA libraries in the `${LD_LIBRARY_PATH}` variable:
 
```bash
$ export LD_LIBRARY_PATH=/location/of/SOWFA/libraries/:${LD_LIBRARY_PATH}
```
 

## References

 * [SOWFA github](https://github.com/NREL/SOWFA.git)
 * [OpenFAST](http://openfast.readthedocs.io)
 * [OpenFAST installation docs](http://openfast.readthedocs.io/en/master/source/install/index.html)
 * [OpenFOAM installation](https://openfoamwiki.net/index.php/Installation/Linux/OpenFOAM-2.4.0/Ubuntu#Ubuntu_16.04)





