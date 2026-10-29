# FAISS 源码编译：Ubuntu 24.04 环境搭建

**概述:** 在搜索会使用到向量检索，而且使用的C++开发环境，Faiss就是个很不错的选择。下面主要是讲述如何在Intel MKL平台加速向量计算的编译和案例

## **下载Intel MKL 组件，用于加速向量计算**

详细参考这个地址:https://www.intel.com/content/www/us/en/developer/tools/oneapi/onemkl-download.html?operatingsystem=linux&linux-install=apt，安装完成后会默认在系统的目录下：/opt/intel/oneapi
安装运行时：sudo apt install intel-oneapi-mkl
安装开发和编译：sudo apt install intel-oneapi-mkl

## **下载faiss源码**

直接克隆源码：git clone https://github.com/facebookresearch/faiss.git

## **编译源码**

这里使用CPU，avx2，intelMKL 库，详细过程
使intelMKL 环境生效，执行：source /opt/intel/oneapi/setvars.sh
设置编译参数：
mkdir build && cd build
cmake .. \
-DBLA_VENDOR=Intel10_64lp \
-DFAISS_ENABLE_GPU=OFF \
-DFAISS_ENABLE_PYTHON=OFF \
-DFAISS_OPT_LEVEL=avx2 \
-DCMAKE_BUILD_TYPE=Release

make -j$(nproc)

## **测试代码**

```cpp
#include <faiss/IndexFlat.h>
#include <faiss/index_io.h>
#include <memory>

// Create a main function
int main() {
    // Define the dimension of the vectors
    int d = 64;

    // Define the number of vectors to index
    int nb = 100000;

    // Generate some random data to index
    float *xb = new float[d * nb];
    for(int i = 0; i < nb; i++) {
        for(int j = 0; j < d; j++) {
            xb[d * i + j] = drand48();
        }
    }

    // Create an index
    faiss::IndexFlatL2 index(d);

    // Index the data
    index.add(nb, xb);

    // Save the index to disk
    faiss::write_index(&index, "example.index");

    // Load the index from disk
    std::unique_ptr<faiss::Index> index2 = faiss::read_index_up("example.index");

    // Search the index
    int k = 5;
    faiss::idx_t *I = new faiss::idx_t[k * nb];
    float *D = new float[k * nb];
    index2->search(nb, xb, k, D, I);

    // Print the results
    for(int i = 0; i < nb; i++) {
        printf("Query %d:\n", i);
        for(int j = 0; j < k; j++) {
            printf("  %lld (distance=%g)\n", static_cast<long long>(I[k * i + j]), D[k * i + j]);
        }
    }

    // Clean up
    delete[] xb;
    delete[] I;
    delete[] D;

    return 0;
}
```

## **编译命令**

g++ test_simple.cpp -o test_simple \
-I./ \
-I/opt/intel/oneapi/mkl/latest/include \
-L./build/faiss \
-L/opt/intel/oneapi/mkl/latest/lib \
-lfaiss_avx2 \
-lmkl_rt \
-lgomp \
-lpthread -lm -O2

## **验证**

执行：./test_simple
