[![Build Status](https://travis-ci.org/zpzim/SCAMP.svg?branch=master)](https://travis-ci.org/zpzim/SCAMP)
# SCAMP: SCAlable Matrix Profile
This is a GPU implementation of the SCAMP algorithm. SCAMP takes a time series as input and computes the matrix profile for a particular window size. You can read more at the [Matrix Profile Homepage](http://www.cs.ucr.edu/~eamonn/MatrixProfile.html)
This is a much improved framework over [GPU-STOMP](https://github.com/zpzim/STOMPSelfJoin) which has the following additional features:
 * Tiling for large inputs 
 * Computation in fp32, mixed fp32/fp64, or fp64 (mixed is recommended for most datasets, but if it doesn't work try double precision)
 * fp32 version should be compatible with GeForce cards
 * AB joins (you can produce the matrix profile from 2 different time series)
 * Distributable (we use AWS but other cloud platforms can work) with verified scalability to billions of datapoints
 * Some optimizations for architectures other than Volta

Note: for self-joins on small inputs (~2M or less) the features in this repository are probably overkill. You can use [GPU-STOMP](https://github.com/zpzim/STOMPSelfJoin) with little performance difference.

# Environment
This base project requires:
 * At least version 9.0 of the CUDA toolkit available [here](https://developer.nvidia.com/cuda-toolkit).
 * At least version 6.0 of clang (for clang-tidy and clang-format)
 * Currently builds under linux using gcc/clang and nvcc with cmake (3.8+ for cuda support), this version is not yet widely available in package managers (i.e. apt) so you will probably need to install it manually from [here](https://cmake.org/download/)
 * An NVIDIA GPU with CUDA support is also required. You can find a list of CUDA compatible GPUs [here](https://developer.nvidia.com/cuda-gpus)
 * Currently Supports Kepler-Volta, but Turing and beyond will likely work as well, just add the -gencode flag for your specific architecture in CMakeLists.txt
 * Should compile under windows, but untested. You will probably have to handle the parsing of command line arguments differently in windows.
 * We highly recommend using a Volta GPU (we get about a 3x improvement over Pascal)
# Usage
~~~~
git clone https://github.com/zpzim/SCAMP
cd SCAMP
cmake -D CUDA_TOOLKIT_ROOT_DIR=/path/to/cuda/install \
      -D CMAKE_CUDA_COMPILER=/path/to/nvcc \
      -D CMAKE_CXX_COMPILER=/path/to/clang/or/gcc .
make -j4
./SCAMP window_size input_A_file_path output_matrix_profile_path output_index_path
~~~~
~~~~
# Example parameters for cmake for a typical linux system with cuda and clang installed
cmake -D CUDA_TOOLKIT_ROOT_DIR=/usr/local/cuda
      -D CMAKE_CUDA_COMPILER=/usr/local/cuda/bin/nvcc
      -D CMAKE_CXX_COMPILER=clang++ .
~~~~
* Optional Arguments:
    * "-b [input B path]": allows a second input file which acts as the second time series for an AB join. An AB join compares every subsequence in input A with every subsequence in input B, the length of the matrix profile produced by this operation is always determined by input A, but the matrix profile index's values will reference subsequences in input B.
    * "-s [max tile size]": allows you to specify the max tile size used by the SCAMP tile scheme. By default this is set to 1M, but you can adjust this as desired. Note that a tile size smaller than ~1M will likely fail to saturate the compute resources of newer GPUs
    * "-d": forces SCAMP to compute the result in double precision. This is about 2x slower than single precision. This is uncessessary for many datasets, but it is necessary for datasets with values that cannot be represented in single precision. See test/SampleInput/earthquake_precision_test.txt for an example of a dataset that fails in single precision.
    * "-m": forces SCAMP to compute the result in mixed precision. This is slower than single precision, and faster than double precision. This precision should be satisfactory for most datasets.
    * "-g [device number to use]": allows you to specify which gpus to use on the machine, by default we try to use all of them. The device numbers must be valid cuda devices on your system. You can chain these to add more gpus. Example: -g 0 -g 1 will use gpu 0 and 1 on the system.
    * "-f [secondary output prefix]": only set this in a distributed environment. This forces AB joins to output a second matrix profile and index file which correspond to the reverse "BA" join. This is used to compute independant pieces of a large self-join that is distributed in separate jobs
    * "-r [global tile row]" : allows you to specify the tile row where this join starts, assuming that it is part of some larger join, this is required if you use -f
    * "-c [global tile col]" : allows you to specify the tile column where this join starts, assuming that it is part of some larger join, this is required if you use -f
* By default, if no devices are specified, SCAMP will run on all available devices
* By default, SCAMP will compute the result in single precision, this may not be what you want, and if you have trouble you should use the -m or -d options.
* cmake provides support for clang-tidy (when you build) and clang-format (using build target clang-format) to use these please make sure clang-tidy and clang-format are installed on your system



# AWS operation
* This framework can be used with [AWS Batch](https://aws.amazon.com/batch) to distribute the computation to a cluster of p2 or p3 instances
* Information forthcoming, but the scripts we used to scale out the algorithm are included in the aws/ directory

# TODOs (Contributors welcome):
* Cleanup codebase, improve integration testing
* Add an optimized CPU code path to this framework, we have optimized code [here](https://github.com/kavj/matrixProfile)
* Add documentation for and improve the general usability of the distributed portion of the framework, ease of use and portability would be great



