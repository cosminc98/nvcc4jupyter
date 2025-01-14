Usage
=====

Installation
------------

To use nvcc4jupyter, first install it using pip:

.. code-block:: console

    (venv) $ pip install nvcc4jupyter

Load the Extension
------------------

Now we need to load the IPython extension to be able to use its cell and line
magic commands:

.. code-block::

    %load_ext nvcc4jupyter

Hello World
-----------

We will use the :ref:`cuda <cuda_magic>` cell magic command to run a simple
hello world program.

.. code-block:: c++

    %%cuda
    #include <stdio.h>

    __global__ void hello(){
        printf("Hello from block: %u, thread: %u\n", blockIdx.x, threadIdx.x);
    }

    int main(){
        hello<<<2, 2>>>();
        cudaDeviceSynchronize();
    }

Groups
------

Now we will demonstrate a more complex scenario that uses source file groups.
If you want to split your code into multiple source files, either for code reuse
or just to have an easier to read project, you want to use groups. A group of
source files will be compiled together. Because of this, you can include headers
from the same group and use the code defined in other ".cu" files. There is also
a special group named "shared" whose files will be compiled together with all
other groups, which is a great feature for error handling code as we'll show now:

.. code-block:: c++

    %%cuda_group_save --group shared --name "error_handling.h"
    // error checking macro
    #define cudaCheckErrors(msg) \
        do { \
            cudaError_t __err = cudaGetLastError(); \
            if (__err != cudaSuccess) { \
                fprintf(stderr, "Fatal error: %s (%s at %s:%d)\n", \
                    msg, cudaGetErrorString(__err), \
                    __FILE__, __LINE__); \
                fprintf(stderr, "*** FAILED - ABORTING\n"); \
                exit(1); \
            } \
        } while (0)

Now we can use that error handling macro in this vector addition program but
also in other programs that we define in other Jupyter cells:

.. code-block:: c++

    %%cuda
    #include <stdio.h>
    #include "error_handling.h"

    const int DSIZE = 4096;
    const int block_size = 256;

    // vector add kernel: C = A + B
    __global__ void vadd(const float *A, const float *B, float *C, int ds){
        int idx = threadIdx.x + blockIdx.x * blockDim.x;
        if (idx < ds) {
            C[idx] = A[idx] + B[idx];
        }
    }

    int main(){
        float *h_A, *h_B, *h_C, *d_A, *d_B, *d_C;

        // allocate space for vectors in host memory
        h_A = new float[DSIZE];
        h_B = new float[DSIZE];
        h_C = new float[DSIZE];

        // initialize vectors in host memory to random values (except for the
        // result vector whose values do not matter as they will be overwritten)
        for (int i = 0; i < DSIZE; i++) {
            h_A[i] = rand()/(float)RAND_MAX;
            h_B[i] = rand()/(float)RAND_MAX;
        }

        // allocate space for vectors in device memory
        cudaMalloc(&d_A, DSIZE*sizeof(float));
        cudaMalloc(&d_B, DSIZE*sizeof(float));
        cudaMalloc(&d_C, DSIZE*sizeof(float));
        cudaCheckErrors("cudaMalloc failure"); // error checking

        // copy vectors A and B from host to device:
        cudaMemcpy(d_A, h_A, DSIZE*sizeof(float), cudaMemcpyHostToDevice);
        cudaMemcpy(d_B, h_B, DSIZE*sizeof(float), cudaMemcpyHostToDevice);
        cudaCheckErrors("cudaMemcpy H2D failure");

        // launch the vector adding kernel
        vadd<<<(DSIZE+block_size-1)/block_size, block_size>>>(d_A, d_B, d_C, DSIZE);
        cudaCheckErrors("kernel launch failure");

        // wait for the kernel to finish execution
        cudaDeviceSynchronize();
        cudaCheckErrors("kernel execution failure");

        cudaMemcpy(h_C, d_C, DSIZE*sizeof(float), cudaMemcpyDeviceToHost);
        cudaCheckErrors("cudaMemcpy D2H failure");

        printf("A[0] = %f\n", h_A[0]);
        printf("B[0] = %f\n", h_B[0]);
        printf("C[0] = %f\n", h_C[0]);
        return 0;
    }

Above we use the :ref:`cuda <cuda_magic>` magic command which saves the code
in the cell to an anonymous source file group, compiles, and executes that
code. This only allows us to have one source file (besides the ones in the
"shared" group). In order to have multiple source files we need to use the
:ref:`cuda_group_save <cuda_group_save_magic>` and
:ref:`cuda_group_run <cuda_group_run_magic>` magics.

First, we save the vector addition function to its own file:


.. code-block:: c++

    %%cuda_group_save --name "vector_add.cu" --group "vector_add"
    // vector add kernel: C = A + B
    __global__ void vadd(const float *A, const float *B, float *C, int ds){
        int idx = threadIdx.x + blockIdx.x * blockDim.x;
        if (idx < ds) {
            C[idx] = A[idx] + B[idx];
        }
    }

Now we create a header file so the main cuda file knows the signature of "vadd":

.. code-block:: c++

    %%cuda_group_save --name "vector_add.h" --group "vector_add"
    __global__ void vadd(const float *A, const float *B, float *C, int ds);

To tie it all together, we save the main cuda file, which includes our vector
addition code:

.. code-block:: c++

    %%cuda_group_save --name "main.cu" --group "vector_add"
    #include <stdio.h>
    #include "error_handling.h"
    #include "vector_add.h"

    const int DSIZE = 4096;
    const int block_size = 256;

    int main(){
        float *h_A, *h_B, *h_C, *d_A, *d_B, *d_C;

        // allocate space for vectors in host memory
        h_A = new float[DSIZE];
        h_B = new float[DSIZE];
        h_C = new float[DSIZE];

        // initialize vectors in host memory to random values (except for the
        // result vector whose values do not matter as they will be overwritten)
        for (int i = 0; i < DSIZE; i++) {
            h_A[i] = rand()/(float)RAND_MAX;
            h_B[i] = rand()/(float)RAND_MAX;
        }

        // allocate space for vectors in device memory
        cudaMalloc(&d_A, DSIZE*sizeof(float));
        cudaMalloc(&d_B, DSIZE*sizeof(float));
        cudaMalloc(&d_C, DSIZE*sizeof(float));
        cudaCheckErrors("cudaMalloc failure"); // error checking

        // copy vectors A and B from host to device:
        cudaMemcpy(d_A, h_A, DSIZE*sizeof(float), cudaMemcpyHostToDevice);
        cudaMemcpy(d_B, h_B, DSIZE*sizeof(float), cudaMemcpyHostToDevice);
        cudaCheckErrors("cudaMemcpy H2D failure");

        // launch the vector adding kernel
        vadd<<<(DSIZE+block_size-1)/block_size, block_size>>>(d_A, d_B, d_C, DSIZE);
        cudaCheckErrors("kernel launch failure");

        // wait for the kernel to finish execution
        cudaDeviceSynchronize();
        cudaCheckErrors("kernel execution failure");

        cudaMemcpy(h_C, d_C, DSIZE*sizeof(float), cudaMemcpyDeviceToHost);
        cudaCheckErrors("cudaMemcpy D2H failure");

        printf("A[0] = %f\n", h_A[0]);
        printf("B[0] = %f\n", h_B[0]);
        printf("C[0] = %f\n", h_C[0]);
        return 0;
    }

Now we can compile all the source files in the group and execute the main
function with the following command:

.. code-block:: c++

    %cuda_group_run --group "vector_add"

Profiling
---------

Another important feature of nvcc4jupyter is its integration with the NVIDIA
Nsight Compute profiler, which you need to make sure is installed and its
executable can be found in a directory in your PATH environment variable.

In order to use it and provide the profiler with custom arguments, simply run:

.. code-block:: c++

    %cuda_group_run --group "vector_add" --profile --profiler-args "--section SpeedOfLight"

Running the cell above will compile and execute the vector addition code in the
"vector_add" group and profile it, keeping only the metrics from the
"SpeedOfLight" section. The output will contain something similar to:

.. code-block::

    Section: GPU Speed Of Light Throughput
    ----------------------- ------------- ------------
    Metric Name               Metric Unit Metric Value
    ----------------------- ------------- ------------
    DRAM Frequency          cycle/nsecond         4.65
    SM Frequency            cycle/usecond       544.31
    Elapsed Cycles                  cycle        2,145
    Memory Throughput                   %         3.19
    DRAM Throughput                     %         3.19
    Duration                      usecond         3.94
    L1/TEX Cache Throughput             %         6.67
    L2 Cache Throughput                 %         1.98
    SM Active Cycles                cycle       383.65
    Compute (SM) Throughput             %         1.19
    ----------------------- ------------- ------------
