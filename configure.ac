#                                               -*- Autoconf -*-
# Process this file with autoconf to produce a configure script.

AC_PREREQ([2.65])
AC_INIT([SHOC], [1.1.4], [shoc-help@elist.ornl.gov])
AC_CONFIG_AUX_DIR([build-aux])
AC_CONFIG_SRCDIR([src/common/Timer.h])
AC_CONFIG_HEADERS([config/config.h])
AC_CANONICAL_SYSTEM

# SHOC is a collection of C++ programs
AC_LANG([C++])


# Checks for fundamental programs.
AC_PROG_CXX
AC_PROG_CC
AC_PROG_RANLIB
AC_CHECK_TOOL([AR], [ar], [:])
AC_PATH_PROG([PERL], [perl])


# check whether special NVCXXFLAGS were specified
# if not, set default flags based on whehter we are using g++ and whether our
# C++ compiler supports -g.
# Logic based on the logic used in the AC_PROG_CXX macro.
ac_save_NVCXXFLAGS=${NVCXXFLAGS}
ac_test_NVCXXFLAGS=${NVCXXFLAGS+set}
if test "$ac_test_NVCXXFLAGS" = set
then
    NVCXXFLAGS=$ac_save_NVCXXFLAGS
elif test $ac_cv_prog_cxx_g = yes
then
    if test "$GXX" = yes; then
        NVCXXFLAGS="-g -O2"
    else
        NVCXXFLAGS="-g"
    fi
else
    if test "$GXX" = yes
    then
        NVCXXFLAGS="-O2"
    else
        NVCXXFLAGS=
    fi
fi




# on Linux, we expect that the clock_gettime function will be available,
# but it seems to require the rt library
AS_CASE([$host],
    [*-*-linux*], [LIBS="$LIBS -lrt"],
    [*], []
)


# Checks for header files.
AC_FUNC_ALLOCA
AC_CHECK_HEADERS([float.h stdlib.h string.h sys/time.h sys/timeb.h unistd.h])

# Checks for typedefs, structures, and compiler characteristics.
AC_HEADER_STDBOOL
AC_C_INLINE
AC_C_RESTRICT
AC_TYPE_SIZE_T

# Checks for library functions.
AC_FUNC_ERROR_AT_LINE
AC_CHECK_FUNCS([clock_gettime floor gethostname gettimeofday localtime_r memset pow sqrt strdup])


AS_IF([test "x$ac_cv_func_clock_gettime" = xyes],
    [ AC_MSG_CHECKING([whether clock_gettime() supports CLOCK_PROCESS_CPUTIME_ID timer])
      AC_LINK_IFELSE([AC_LANG_PROGRAM([],[])],
                    [AC_MSG_RESULT([yes])
                     AC_DEFINE([HAVE_CLOCK_PROCESS_CPUTIME_ID],[1],[Define to 1 if you have the CLOCK_PROCESS_CPUTIME_ID timer for the 'clock_gettime' function])
                    ],
                    [AC_MSG_RESULT([no])]
      )
    ]
)    



# OpenCL support
AC_ARG_WITH([opencl],
    [AS_HELP_STRING([--with-opencl],
        [build OpenCL versions of SHOC benchmark programs @<:@default=check@:>@])],
    [],
    [with_opencl=check])

# detect/verify we have a working OpenCL header and library
AS_IF([test "x$with_opencl" != xno],
    [
        # check for usable OpenCL header
        # note C++ binding uses opencl.h, not just cl.h
        AC_MSG_NOTICE([checking for usable OpenCL opencl.h header])
        AC_CHECK_HEADER([OpenCL/opencl.h],
            [OPENCL_HEADER='"OpenCL/opencl.h"'],
            [AC_CHECK_HEADER([CL/opencl.h],
                [OPENCL_HEADER='"CL/opencl.h"'],
                [if test "x$with_opencl" != xcheck; then
                    AC_MSG_FAILURE([OpenCL support was requested, but no usable OpenCL header was found])
                 fi
                ]
             )
            ]
        )

        AS_IF([test "x$OPENCL_HEADER" != x],
            [
            # we have a header, check for usable OpenCL library
            # we cannot use AC_CHECK_LIB because we want to check
            # using the OS X OpenCL framework, and AC_CHECK_LIB does not
            # know how to use the framework linker syntax.
            # (we also do not want the OpenCL libs to be added to the
            # LIBS variable, and AC_CHECK_LIB does that if it succeeds)
            AC_MSG_CHECKING([for usable OpenCL library])
            savedLIBS=$LIBS
            OPENCL_LIBS="-framework OpenCL"
            LIBS="$OPENCL_LIBS $savedLIBS"
            AC_LINK_IFELSE( [AC_LANG_PROGRAM([@%:@include $OPENCL_HEADER],[clGetPlatformIDs(0,0,0);])],
                [AC_MSG_RESULT($OPENCL_LIBS)],
                [OPENCL_LIBS="-lOpenCL"
                 LIBS="$OPENCL_LIBS $savedLIBS"
                 AC_LINK_IFELSE( [AC_LANG_PROGRAM([@%:@include $OPENCL_HEADER],[clGetPlatformIDs(0,0,0);])],
                    [AC_MSG_RESULT($OPENCL_LIBS)],
                    [AC_MSG_RESULT([no])
                     if test "x$with_opencl" != xcheck; then
                         AC_MSG_FAILURE([OpenCL support was requested, but no usable OpenCL library was fund])
                     fi
                    ]
                 )
                ]
            )
            LIBS=$savedLIBS
            ]
        )

        AS_IF([test "x$OPENCL_HEADER" != "x" -a "x$OPENCL_LIBS" != "x"],
            [with_opencl=yes
             SRC_SUBDIRS="$SRC_SUBDIRS opencl"
             MPI_SUBDIRS="$MPI_SUBDIRS opencl"
             AC_SUBST([OPENCL_LIBS])
             ],
            [AC_MSG_NOTICE([no usable OpenCL installation found])
             with_opencl=no
            ])
    ])


# CUDA support
AC_ARG_WITH([cuda],
    [AS_HELP_STRING([--with-cuda],
        [build CUDA versions of SHOC benchmark programs @<:@default=check@:>@])],
    [],
    [with_cuda=check])

USE_CUDA=no
AS_IF([test "x$with_cuda" != xno],
    [
        # check for the nvcc program
        AC_PATH_PROG([NVCC],[nvcc],[])

        AS_IF([test "x$NVCC" != x],
            [ savedCXX=$CXX
              CXX=$NVCC
              savedLIBS=$LIBS
              savedCPPFLAGS=$CPPFLAGS
              savedCXXFLAGS=$CXXFLAGS
              CXXFLAGS=$NVCXXFLAGS

              # find cuda include path based on location of nvcc command
              cudabindir=`AS_DIRNAME([$NVCC])`
              CUDA_INCDIR=`AS_DIRNAME([$cudabindir])`/include
              CPPFLAGS="$CPPFLAGS -I$CUDA_INCDIR"

              AC_CHECK_HEADERS([cuda.h cuda_runtime.h],[],
                [NVCC=""]
              )
              AC_CHECK_LIB([cublas], [cublasInit],[],
                [NVCC=""]
              )
              AC_CHECK_LIB([cufft], [cufftPlan1d], [],
                [NVCC=""]
              )

              CPPFLAGS=$savedCPPFLAGS
              LIBS=$savedLIBS
              CXX=$savedCXX
              CXXFLAGS=$savedCXXFLAGS
            ],
            [ if test "x$with_cuda" != xcheck; then
                AC_MSG_FAILURE([CUDA support was requested, but no usable CUDA installation found])
              fi
              NVCC=""
            ]
        )

        AS_IF([test "x$NVCC" != x],
            [with_cuda=yes
             USE_CUDA=yes
             SRC_SUBDIRS="$SRC_SUBDIRS cuda"
             MPI_SUBDIRS="$MPI_SUBDIRS cuda"
             AC_SUBST([CUDA_INCDIR])],
            [AC_MSG_NOTICE([no usable CUDA installation found])
             with_cuda=no
            ])

        AS_IF([test "x$with_cuda" = xyes],
            [AS_IF([test "x$CUDA_CPPFLAGS" = x],
                [dnl generate CUDA code for broad spectrum of devices
                 dnl Note: cc 13 for Tesla
                 dnl Note: cc 20 for Fermi
                 CUDA_CPPFLAGS=["-gencode=arch=compute_10,code=sm_10 \
 -gencode=arch=compute_11,code=sm_11 \
 -gencode=arch=compute_13,code=sm_13 \
 -gencode=arch=compute_20,code=sm_20 \
 -gencode=arch=compute_20,code=compute_20"]
                 echo "using CUDA_CPPFLAGS=$CUDA_CPPFLAGS"
                ]
             )
             AC_SUBST([CUDA_CPPFLAGS])
            ]
        )
    ]
)


# build stability tests? by default, enable if CUDA is enabled
AC_ARG_ENABLE([stability],
    [AS_HELP_STRING([--disable-stability],
        [build SHOC stability test; requires CUDA @<:@default=enabled@:>@])],
    [],
    [enable_stability=check]
)

AS_IF([test "x$enable_stability" = xyes],
    [AS_IF([test "x$with_cuda" != xno],
        [SRC_SUBDIRS="$SRC_SUBDIRS stability"],
        [ AC_MSG_FAILURE([CUDA-based stability test enabled but no usable CUDA installation found])
          with_stability=no
        ]
    )
    ],
    [AS_IF([test "x$enable_stability" != xno],
        [AS_IF([test "x$with_cuda" != xno],
            [SRC_SUBDIRS="$SRC_SUBDIRS stability"],
        [
          with_stability=no
        ]
        )]
    )]
    
)


# MPI
AC_ARG_WITH([mpi],
    [AS_HELP_STRING([--with-mpi],
        [build MPI-based parallel versions of SHOC benchmarks @<:@default=check@:>@])],
    [],
    [with_mpi=check]
)

USE_MPI=no
AS_IF([test "x$with_mpi" != xno],
    [ 
        AC_PATH_PROGS([MPICXX], [mpicxx mpic++ mpiCC])
        savedCXX=$CXX
        CXX=$MPICXX
        AS_IF([test "x$MPICXX" != "x"],
            [
                AC_MSG_CHECKING([whether we can compile an MPI program using $CXX])
                AC_COMPILE_IFELSE([AC_LANG_PROGRAM([@%:@include "mpi.h"],[MPI_Init(0,0);])],
                    [ AC_MSG_RESULT([yes])],
                    [ AC_MSG_RESULT([no])
                      MPICXX=""
                    ]
                )
            ]
        )

        AS_IF([test "x$MPICXX" != "x"],
            [
                AC_MSG_CHECKING([whether we can link an MPI program using $CXX])
                AC_LINK_IFELSE([AC_LANG_PROGRAM([@%:@include "mpi.h"],[MPI_Init(0,0);])],
                    [
                        AC_MSG_RESULT([yes])
                    ],
                    [
                        AC_MSG_RESULT([no])
                        MPICXX=""
                    ]
                )
            ]
        )
        CXX=$savedCXX

        AS_IF([test "x$MPICXX" != "x"],
            [ AS_IF([test "x$with_opencl" = xyes -o "x$with_cuda" = xyes],
                  [ with_mpi=yes
                    SRC_SUBDIRS="$SRC_SUBDIRS mpi"
                    USE_MPI=yes
                  ],
                  [ AC_MSG_FAILURE([MPI support was requested, but is only meaningful if OpenCL and/or CUDA support is available])
                    with_mpi=no
                  ]
              )
            ],
            [ AS_IF([test "x$with_mpi" != xcheck],
                    [ AC_MSG_FAILURE([--with-mpi specified, but no usable MPI installation found.])
                      with_mpi=no
                    ]
              )
            ]
        )
    ]
)

AC_SUBST([USE_MPI])
AC_SUBST([MPICXX])
AC_SUBST([USE_CUDA])
AC_SUBST([SRC_SUBDIRS])
AC_SUBST([MPI_SUBDIRS])
AC_SUBST([NVCXXFLAGS])

AC_CONFIG_FILES([Makefile
                 config/common.mk
                 config/targets.mk
                 src/Makefile
                 src/cuda/Makefile
                 src/cuda/level0/Makefile
                 src/cuda/level1/Makefile
                 src/cuda/level1/bfs/Makefile
                 src/cuda/level1/fft/Makefile
                 src/cuda/level1/md/Makefile
                 src/cuda/level1/reduction/Makefile
                 src/cuda/level1/scan/Makefile
                 src/cuda/level1/sgemm/Makefile
                 src/cuda/level1/sort/Makefile
                 src/cuda/level1/spmv/Makefile
                 src/cuda/level1/stencil2d/Makefile
                 src/cuda/level1/triad/Makefile
                 src/cuda/level2/Makefile
                 src/cuda/level2/qtclustering/Makefile
                 src/cuda/level2/s3d/Makefile
                 src/mpi/Makefile
                 src/mpi/common/Makefile
                 src/mpi/contention/Makefile
                 src/mpi/contention/cuda/Makefile
                 src/mpi/contention/opencl/Makefile
                 src/mpi/contention-mt/Makefile
                 src/mpi/contention-mt/cuda/Makefile
                 src/mpi/contention-mt/opencl/Makefile
                 src/opencl/Makefile
                 src/opencl/common/Makefile
                 src/opencl/level0/Makefile
                 src/opencl/level1/Makefile
                 src/opencl/level1/bfs/Makefile
                 src/opencl/level1/fft/Makefile
                 src/opencl/level1/md/Makefile
                 src/opencl/level1/reduction/Makefile
                 src/opencl/level1/scan/Makefile
                 src/opencl/level1/sgemm/Makefile
                 src/opencl/level1/sort/Makefile
                 src/opencl/level1/spmv/Makefile
                 src/opencl/level1/stencil2d/Makefile
                 src/opencl/level1/triad/Makefile
                 src/opencl/level2/Makefile
                 src/opencl/level2/s3d/Makefile
                 src/stability/Makefile
                 ])
AC_OUTPUT
