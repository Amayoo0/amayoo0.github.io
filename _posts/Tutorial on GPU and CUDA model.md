---
title: Tutorial on GPU and CUDA model
categories: CUDA GPU
excerpt: | 
   A tutorial on GPU parallel programming using the CUDA model.

feature_text: |
  ## Tutorial on GPU and CUDA model
  
feature_image: "https://picsum.photos/2560/600?image=733"
image: "https://picsum.photos/2560/600?image=733"
---


## Architecture

The Graphic Processing Unit (GPU) is designed to work with parallelism mindset. CPU is designed to complete operations sequentially (this is what we call _thread_) and can execute tens of thread, while GPU can execute thousands of them.

### Memory hierarchy

Green block is called _Grid_ and is composed by Thread Blocks. Thread Block is a group of thread with shared memory. This group of thread is also called _warp_ and has 32 thread.
Thread blocks in a 'Thread Block Cluster' can perform read, write, and atomics operations on each other's shared memory. Finally, all threads have access to the same global memory.

## Development Environment Setup
Connect to the UMA VPN and establish a *Remote Desktop Connection* to your server:

``` bash
docker start keysuser3_dev_gemini
docker exec -it keysuser3_dev_gemini bash
cd my_share/KeysightTrack
pip3 install -r requirements.txt
source set_env.sh
```

## Building and Running the Code

Before building and running the code, be sure to have properly setted up the environment as described in the [Development Environment Setup](#development-environment-setup) section.

To build the code you need first to **create the build directory**. This is done in the repository root directory with:

```sh
meson setup build -Dtarget=<target>
```

Where `<target>` identifies the binary you want to compile, between:

- `tutorial` - For the tutorial code under `src/tutorial`.
- `modem_rx` - For the modem code under `src/modem_rx`. This is the **default value**.

> **Note**: If no `<target>` is provided, `modem_rx` is used by default.

The next step is to **build the code** which is achieved by:

```shell
meson compile -C build
```

The last step is to **test the code** which is achieved by:

```shell
meson test -C build
```

> **Note:** Alternatively, you can use VSCode tasks to build and run the code as described in the [VSCode tasks](https://github.com/pmonaster/KeysightTrack/blob/main/README.md#vscode-tasks) section.

## Exercise 1: HelloWorld
./src/tutorial/test/main.cpp
``` C++
#include "helloWorld.cuh"
int main(int argc, char **argv)
{
    int numBlocks = 1;
    int numThreads = 1;
    hw::helloWorld(numBlocks,numThreads);
}
```

./src/tutorial/src/helloWorld.cu
``` C++
#include <cuda.h>
#include <cuda_fp16.h>
#include <stdio.h>
#include "helloWorld.cuh"
#include <chrono>
#include <thread>

//
// version 1: new beginners version not utilizing any parallelism
//
__global__ void kernel()
{
    printf("[GPU] hello from GPU block %d, thread %d\n", blockIdx.x, threadIdx.x);
}

__host__ void hw::helloWorld(int gridDim, int blockDim)
{
    kernel<<<gridDim,blockDim>>>(); 

    // dummy delay to avoid aborting before print is done
    using namespace std::this_thread; // sleep_for, sleep_until
    using namespace std::chrono; // nanoseconds, system_clock, seconds
    std::this_thread::sleep_for(nanoseconds(100000000));
}
```

Output: $ ./build/src/tutorial/tutorial
``` bash
[GPU] hello from GPU block 0, thread 0
```
## Exercise 2: ScalarProduct Sequentially


./src/tutorial/test/main.cpp
``` C++
#include <iostream>
#include <fstream>
#include "cudaScalarProduct1.cuh"
//#include "cudaScalarProduct2.cuh"
//#include "cudaScalarProduct3.cuh"

#define DIM (1024 * 1024)
float x[DIM];  //reserva memoria para los vectores de memoria
float y[DIM];

using namespace std;
using namespace cudaSP1;
//using namespace cudaSP2;
//using namespace cudaSP3;

int main(int argc, char **argv)
{
    // fetch vector x from file
    ifstream fid;
    fid.open(argv[1]);
    if (!fid)
    {
        cout << "[ERROR] " << argv[1] << " not found" << endl;
        exit(0);
    }
    cout << "[HST] reading file " << argv[1] << " into vector x" << endl;
    for (int row = 0; row < DIM; row++) fid >> x[row];
    fid.close();


    // fetch vector y from file
    fid.open(argv[2]);
    if (!fid)
    {
        cout << "[ERROR] " << argv[2] << " not found" << endl;
        exit(0);
    }
    cout << "[HST] reading file " << argv[2] << " into vector y" << endl;
    for (int row = 0; row < DIM; row++) fid >> y[row];
    fid.close();

    //
    // computation of scalar vector product
    //
    cout << "[HST] computation of scalar product x'y" << endl;
    float ms;
    float z = scalarProduct(x, y, DIM, &ms); //arguments: vectors x, y, vector's dimension and ms by reference to calculate the processing delay.
    printf("[HST] x'y =  %7.1f\n", z);
    printf("[HST] CUDA x'y execution time %.1f us\n", ms * 1e3);

}
```

./src/tutorial/src/helloWorld.cu
``` C++
#include <cuda.h>
#include <cuda_fp16.h>
#include <stdio.h>
#include "cudaScalarProduct1.cuh"
#define NREP 100

//
// version 1: new beginners version not utilizing any parallelism
//
__global__ void cudaScalarProduct1(float *x, float *y, float *z, int dim)
{
    float sum = 0.0;
    for (int n = 0; n < dim; n++)
    {
        sum += x[n] * y[n];
    }
    *z = sum;
}

__host__ int getBlockSize(int dim)
{
    return 1;
}

__host__ int getGridSize(int dim)
{
    return 1;
}

__host__ float cudaSP1::scalarProduct(float *x, float *y, int dim, float *ms)
{
    printf("[HST] using cuda scalarProduct version 1\n");
    // alloc memories in GPU
    float *devX, *devY;
    cudaMalloc(&devX, dim * sizeof(float));
    cudaMalloc(&devY, dim * sizeof(float));
    float *devZ;
    cudaMalloc(&devZ, 1 * sizeof(float));


    // copy input data (from host) to memory in GPU (to device)
    cudaMemcpy(devX, x, dim * sizeof(float), cudaMemcpyHostToDevice);
    cudaMemcpy(devY, y, dim * sizeof(float), cudaMemcpyHostToDevice);

    int gridDim;
    gridDim = getGridSize(dim);
    int blockDim;
    blockDim = getBlockSize(dim);
    int devDim = dim;

    void *Args[] = {(void *)&devX, (void *)&devY, (void *)&devZ, (void *)&devDim};
    void *Func = reinterpret_cast<void *>(cudaScalarProduct1);

	// start event to calculate processing delay
    cudaEvent_t eStart;
    cudaEvent_t eStop;
    cudaEventCreate(&eStart);
    cudaEventCreate(&eStop);
    cudaEventRecord(eStart);

    for (int n=0; n<NREP; n++)
    {
        cudaError_t cudaError = cudaLaunchKernel(Func, gridDim, blockDim, Args);
        if (cudaError != cudaSuccess) printf("[HST] cudaError = %d\n", (int)cudaError);
    }

	// end event of calculating delay
    cudaEventRecord(eStop);
    cudaEventSynchronize(eStop);
    cudaEventElapsedTime(ms, eStart, eStop);
    cudaEventDestroy(eStart);
    cudaEventDestroy(eStop);
    *ms = *ms / NREP;

	// copy output data form device (GPU) to host (CPU)
    float hstZ;
    cudaMemcpy(&hstZ, devZ, 1 * sizeof(float), cudaMemcpyDeviceToHost);

    cudaFree(devZ);
    cudaFree(devY);
    cudaFree(devX);
    return hstZ;
}
```



Output: ./build/src/tutorial/tutorial ./src/tutorial/data/tv_x.csv ./src/tutorial/data/tv_y.csv 
``` bash
[HST] reading file ./src/tutorial/data/tv_x.csv into vector x
[HST] reading file ./src/tutorial/data/tv_y.csv into vector y
[HST] computation of scalar product x'y
[HST] using cuda scalarProduct version 1
[HST] x'y =   1332.3
[HST] CUDA x'y execution time 23463.5 us
```


## Exercise 2: ScalarProduct Thread Parallelism

In this exercise we are going to use thread parallelism for computing scalar vector product.

First exercise resolve this operation without parallelism and compute the result value with only one thread. What we are going to do now is to distribute the computational effort among many thread.  

$$ \sum_{n=0}^{1024^2 - 1} x[n] \cdot y[n]  = \sum_{id=0}^{1024 - 1}\sum_{n=0}^{1024 - 1} x[id+1024\cdot n] \cdot y[id + 1024 \cdot n]$$

./src/tutorial/test/main.cpp
``` C++
#include <iostream>
#include <fstream>
//#include "cudaScalarProduct1.cuh"
#include "cudaScalarProduct2.cuh"
//#include "cudaScalarProduct3.cuh"

#define DIM (1024 * 1024)
float x[DIM];  //reserva memoria para los vectores de memoria
float y[DIM];

using namespace std;
//using namespace cudaSP1;
using namespace cudaSP2;
//using namespace cudaSP3;

int main(int argc, char **argv)
{
    // fetch vector x from file
    ifstream fid;
    fid.open(argv[1]);
    if (!fid)
    {
        cout << "[ERROR] " << argv[1] << " not found" << endl;
        exit(0);
    }
    cout << "[HST] reading file " << argv[1] << " into vector x" << endl;
    for (int row = 0; row < DIM; row++) fid >> x[row];
    fid.close();


    // fetch vector y from file
    fid.open(argv[2]);
    if (!fid)
    {
        cout << "[ERROR] " << argv[2] << " not found" << endl;
        exit(0);
    }
    cout << "[HST] reading file " << argv[2] << " into vector y" << endl;
    for (int row = 0; row < DIM; row++) fid >> y[row];
    fid.close();

    //
    // computation of scalar vector product
    //
    cout << "[HST] computation of scalar product x'y" << endl;
    float ms;
    float z = scalarProduct(x, y, DIM, &ms); //arguments: vectors x, y, vector's dimension and ms by reference to calculate the processing delay.
    printf("[HST] x'y =  %7.1f\n", z);
    printf("[HST] CUDA x'y execution time %.1f us\n", ms * 1e3);

}
```

./src/tutorial/src/cudaScalarProduct2.cu
``` C++
#include <cuda.h>
#include <cuda_fp16.h>
#include <stdio.h>
#include "cudaScalarProduct2.cuh"
#define NREP 100

//
// version 2: version utilizing thread parallelism but not grid parallelism
//
__global__ void cudaScalarProduct2(float *x, float *y, float *z, int dim)
{
    __shared__ float sum[32];

    int n = threadIdx.x;
    int tidx = threadIdx.x % 32;
    int widx = threadIdx.x / 32;

    float value = 0.0;
    while (n < dim)
    {
        value += x[n] * y[n];
        n += blockDim.x;
    }
	// __shfl_down_sync perfom a tree-reduction
    for (int i = 16; i >= 1; i /= 2) value += __shfl_down_sync(0xffffffff, value, i, 32);
    if (tidx == 0) sum[widx] = value;
    __syncthreads();

    if (widx == 0)
    {
        value = sum[tidx];
        for (int i = 16; i >= 1; i /= 2) value += __shfl_down_sync(0xffffffff, value, i, 32);
        if (tidx == 0) *z = value;
    }
}


__host__ int getBlockSize2(int dim)
{
    int n = dim % 1024;
    return n == 0 ? 1024 : n;
}

__host__ int getGridSize2(int dim)
{
    return 1;
}

__host__ float cudaSP2::scalarProduct(float *x, float *y, int dim, float *ms)
{
    printf("[HST] using cuda scalarProduct version 2\n");
    // alloc memories in GPU
    float *devX, *devY;
    cudaMalloc(&devX, dim * sizeof(float));
    cudaMalloc(&devY, dim * sizeof(float));
    float *devZ;
    cudaMalloc(&devZ, 1 * sizeof(float));


    // copy input data to memory in GPU
    cudaMemcpy(devX, x, dim * sizeof(float), cudaMemcpyHostToDevice);
    cudaMemcpy(devY, y, dim * sizeof(float), cudaMemcpyHostToDevice);

    int gridDim; 
    gridDim = getGridSize2(dim);
    int blockDim;
    blockDim = getBlockSize2(dim);
    int devDim = dim;

    void *Args[] = {(void *)&devX, (void *)&devY, (void *)&devZ, (void *)&devDim};
    void *Func = reinterpret_cast<void *>(cudaScalarProduct2);

    cudaEvent_t eStart;
    cudaEvent_t eStop;
    cudaEventCreate(&eStart);
    cudaEventCreate(&eStop);
    cudaEventRecord(eStart);

    for (int n=0; n<NREP; n++)
    {
        cudaError_t cudaError = cudaLaunchKernel(Func, gridDim, blockDim, Args);
        if (cudaError != cudaSuccess) printf("[HST] cudaError = %d\n", (int)cudaError);
    }

    cudaEventRecord(eStop);
    cudaEventSynchronize(eStop);
    cudaEventElapsedTime(ms, eStart, eStop);
    cudaEventDestroy(eStart);
    cudaEventDestroy(eStop);
    *ms = *ms / NREP;

    float hstZ;
    cudaMemcpy(&hstZ, devZ, 1 * sizeof(float), cudaMemcpyDeviceToHost);

    cudaFree(devZ);
    cudaFree(devY);
    cudaFree(devX);
    return hstZ;
}
```




Output: ./build/src/tutorial/tutorial ./src/tutorial/data/tv_x.csv ./src/tutorial/data/tv_y.csv 
```sh
[HST] reading file ./src/tutorial/data/tv_x.csv into vector x
[HST] reading file ./src/tutorial/data/tv_y.csv into vector y
[HST] computation of scalar product x'y
[HST] using cuda scalarProduct version 1
[HST] x'y =   1332.3
[HST] CUDA x'y execution time 333.4 us
```

## Homework: cudaVectorProjection 

Implementing a kernel for computing the vector projection formula using any number of GPU blocks and threads. 
$$z=\frac{x^T y}{y^T y} y$$
Use all we have learnt in this tutorial:
- Create your git branch and save there.
- Remember to add new files that you create for compilation, e.g. cudaVectorProjection.cu in the meson build file.
- Compute first scalar products $x^T y$, $y^T y$ using the kernels for scalar products then create a kernel that scales y with the scalar value ($\frac{x^T y}{y^T y}$).
- Keep a similar coding style/structure (\*.cuh and \*cu file,  name spacing, variable naming, function naming…)
- In main.cpp add a final check that checks the error $(z-z_{ref})^T (z-z_{ref})<1e^{-6}$ (the reference $z_{ref}$  is in the tv_z.csv file)


./src/tutorial/test/main.cpp
``` C++
#include <iostream>
#include <fstream>
// includes for vectorProjection1
#include "cudaScalarProduct2.cuh"
#include "cudaVectorProjection1.cuh"
// includes for vectorProjection2
#include "cudaScalarProduct3.cuh"
#include "cudaVectorProjection2.cuh"
#define DIM (1024 * 1024)

float x[DIM];
float y[DIM];
float z[DIM];
float z_ref[DIM];

using namespace std;
using namespace cudaVP2;

int main(int argc, char **argv)
{
    // fetch vector x from file
    ifstream fid;
    fid.open(argv[1]);
    if (!fid)
    {
        cout << "[ERROR] " << argv[1] << " not found" << endl;
        exit(0);
    }
    cout << "[HST] reading file " << argv[1] << " into vector x" << endl;
    for (int row = 0; row < DIM; row++) fid >> x[row];
    fid.close();

    // fetch vector y from file
    fid.open(argv[2]);
    if (!fid)
    {
        cout << "[ERROR] " << argv[2] << " not found" << endl;
        exit(0);
    }
    cout << "[HST] reading file " << argv[2] << " into vector y" << endl;
    for (int row = 0; row < DIM; row++) fid >> y[row];
    fid.close();

    // fetch vector z_ref from file
    fid.open(argv[3]);
    if (!fid)
    {
        cout << "[ERROR] " << argv[3] << " not found" << endl;
        exit(0);
    }
    cout << "[HST] reading file " << argv[3] << " into vector z_ref" << endl;
    for (int row = 0; row < DIM; row++) fid >> z_ref[row];
    fid.close();

   
    //
    // computation of vector projection
    //
    float ms;
    //cudaVP1::vectorProjection(x, y, z, DIM, &ms);
    cudaVP2::vectorProjection(x, y, z, DIM, &ms);
    

    // printing results
    printf("[HST] CUDA x'y execution time %.1f us\n", ms * 1e3);
    printf("[HST] Compare last 10 numbers of Z\n--- z_ref[i],  z[i] --- \n");
    for (int i = DIM-10; i < DIM; i++) printf("%8.4e, %8.4e\n", z_ref[i],z[i]);   

    // computation of error
    float error = 0.0;
    for (int i = 0; i < DIM; i++) error += (z[i]-z_ref[i])*(z[i]-z_ref[i]);

    if (error >= 1e-6)
    {
        printf("[ERROR] Error value: %2.1f\n", error);
		exit(0);
    }else{
        printf("[HST] Error value: %2.1f\n", error);
    } 
 }
```

./src/tutorial/src/cudaVectorProjection2.cu
``` C++
#include <cuda.h>
#include <cuda_fp16.h>
#include <stdio.h>
#include "cudaScalarProduct3.cuh"
#include "cudaVectorProjection2.cuh"
#define NREP 100

using namespace cudaVP2;

//
// version 2: version utilizing thread parallelism but not grid parallelism
//
__global__ void cudaVP2::cudaVectorProjection(float *xy, float *yy, float *y, float *z, int dim)
{
    int n = blockDim.x * blockIdx.x + threadIdx.x;
    while (n < dim)
    {
        z[n] = yy[0] == 0.0 ? 0.0 : (xy[0] / yy[0]) * y[n];
        n += gridDim.x * blockDim.x;
    }
}


__host__ void cudaVP2::vectorProjection(float *x, float *y, float *z, int dim, float *ms)
{
    printf("[HST] using cuda vectorProjection version 2\n");
    // alloc memories in GPU
    float *devX, *devY, *devZ;
    cudaMalloc(&devX, dim * sizeof(float));
    cudaMalloc(&devY, dim * sizeof(float));
    cudaMalloc(&devZ, dim * sizeof(float));

    float *devXY, *devYY;
    cudaMalloc(&devXY, 1 * sizeof(float));
    cudaMalloc(&devYY, 1 * sizeof(float));


    // copy input data to memory in GPU
    cudaMemcpy(devX, x, dim * sizeof(float), cudaMemcpyHostToDevice);
    cudaMemcpy(devY, y, dim * sizeof(float), cudaMemcpyHostToDevice);

    int gridDim = cudaSP3::getGridSize(dim);
    int blockDim = cudaSP3::getBlockSize(dim);
    int devDim = dim;

    void *Args1[] = {(void *)&devX, (void *)&devY, (void *)&devXY, (void *)&devDim};
    void *Func1 = reinterpret_cast<void *>(cudaSP3::cudaScalarProduct);

    void *Args2[] = {(void *)&devY, (void *)&devY, (void *)&devYY, (void *)&devDim};
    void *Func2 = reinterpret_cast<void *>(cudaSP3::cudaScalarProduct);

    void *Args3[] = {(void *)&devXY, (void *)&devYY, (void *)&devY, (void *)&devZ, (void *)&devDim};
    void *Func3 = reinterpret_cast<void *>(cudaVP2::cudaVectorProjection);



    cudaEvent_t eStart;
    cudaEvent_t eStop;
    cudaEventCreate(&eStart);
    cudaEventCreate(&eStop);
    cudaEventRecord(eStart);

    for (int n=0; n<NREP; n++)
    {
	// Inicialice devXY and devYY to 0.0
	float aux = 0.0;
	cudaMemcpy(devXY, &aux, 1 * sizeof(float), cudaMemcpyHostToDevice);	
	cudaMemcpy(devYY, &aux, 1 * sizeof(float), cudaMemcpyHostToDevice);	
	   
        cudaError_t cudaError = cudaLaunchKernel(Func1, gridDim, blockDim, Args1);
        if (cudaError != cudaSuccess) printf("[HST] cudaError1 = %d\n", (int)cudaError);
	
        cudaError = cudaLaunchKernel(Func2, gridDim, blockDim, Args2);
        if (cudaError != cudaSuccess) printf("[HST] cudaError2 = %d\n", (int)cudaError);
        
	cudaError = cudaLaunchKernel(Func3, gridDim, blockDim, Args3);
        if (cudaError != cudaSuccess) printf("[HST] cudaError3 = %d\n", (int)cudaError);
    }

    cudaEventRecord(eStop);
    cudaEventSynchronize(eStop);
    cudaEventElapsedTime(ms, eStart, eStop);
    cudaEventDestroy(eStart);
    cudaEventDestroy(eStop);
    *ms = *ms / NREP;

    cudaMemcpy(z, devZ, dim * sizeof(float), cudaMemcpyDeviceToHost);

    cudaFree(devZ);
    cudaFree(devY);
    cudaFree(devX);
    cudaFree(devXY);
    cudaFree(devXY);
}
```


./src/tutorial/inc/cudaVectorProjection2.cuh
``` C++
#pragma once
namespace cudaVP2
{
	/**
	 * @brief vectorProjection
	 *
	 * @param xy pointer to first scalar product
	 * @param yy pointer to second scalar product
	 * @param y  pointer to the vector
	 * @param z  pointer to vector projection (x'y/y'y)*y
	 * @param dim   length of input vector
	 * @param us pointer to execution time in us
	 *
	 */
	void vectorProjection(float *x, float *y, float *z, int dim, float *us);
	
	// others kernel related..
	#ifdef __cuda_cuda_h__
	__global__ void cudaVectorProjection(float *xy, float *yy, float *y, float *z, int dim);
	#endif
}
```

./src/tutorial/meson.build
``` json
message('## [INFO] processing ' + meson.current_source_dir() + '/meson.build ##')

prj_src = files(
    'src/helloWorld.cu',
    'src/cudaScalarProduct1.cu',
    'src/cudaScalarProduct2.cu',
    'src/cudaScalarProduct3.cu',
    'src/cudaVectorProjection1.cu',
    'src/cudaVectorProjection2.cu',
    'test/scalarProduct.cpp',
    'test/main.cpp'
)

prj_inc = include_directories('inc')

# Configure executable
test_exe = executable(
    'tutorial',
    prj_src,
    dependencies: [
        libcuda_dep,
        libboost_dep,
    ],
    cuda_args: build_args_gpu,
    cpp_args: build_args_cpp,
    link_args: link_args_gpu,
    gnu_symbol_visibility: 'default',
    include_directories: prj_inc,
)

suite_name = 'tutorial'
tv_dir = root_dir + '/src/tutorial/data/'
test('tutorial test', test_exe, args: [tv_dir + 'tv_x.csv', tv_dir + 'tv_y.csv', tv_dir + 'tv_a.csv'], suite: suite_name, is_parallel: false, timeout: 10)
```

Output: ./build/src/tutorial/tutorial ./src/tut
orial/data/tv_x.csv ./src/tutorial/data/tv_y.csv ./src/tutorial/data/tv_z.csv 
``` bash
[HST] reading file ./src/tutorial/data/tv_x.csv into vector x
[HST] reading file ./src/tutorial/data/tv_y.csv into vector y
[HST] reading file ./src/tutorial/data/tv_z.csv into vector z_ref
[HST] using cuda vectorProjection version 1
[HST] CUDA x'y execution time 1632.1 us
[HST] Compare last 10 numbers of Z
--- z_ref[i],  z[i] --- 
1.5140e-03, 1.5144e-03
-1.4440e-03, -1.4440e-03
1.2800e-03, 1.2795e-03
-1.0730e-03, -1.0729e-03
1.8800e-04, 1.8764e-04
7.4300e-04, 7.4331e-04
1.3700e-04, 1.3691e-04
1.6330e-03, 1.6331e-03
-1.6660e-03, -1.6663e-03
-1.9710e-03, -1.9713e-03

[HST] Error value: 0.0
```
