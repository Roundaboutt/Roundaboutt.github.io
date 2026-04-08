## 



## naive实现

```cpp
const int WMMA_M = 16;
const int WMMA_N = 16;
const int WMMA_K = 16;

using namespace nvcuda;
__global__ void gemm_wmma_naive(half* A, half* B, float* C, int M, int N, int K, float alpha, float beta)
{
    int lda = K;
    int ldb = N;
    int ldc = N;

    int warpM = blockIdx.y;
    int warpN = blockIdx.x;

    wmma::fragment<wmma::matrix_a, WMMA_M, WMMA_N, WMMA_K, half, wmma::row_major> a_frag;
    wmma::fragment<wmma::matrix_b, WMMA_M, WMMA_N, WMMA_K, half, wmma::row_major> b_frag;
    wmma::fragment<wmma::accumulator, WMMA_M, WMMA_N, WMMA_K, float> acc_frag;
    wmma::fragment<wmma::accumulator, WMMA_M, WMMA_N, WMMA_K, float> c_frag;
    wmma::fill_fragment(acc_frag, 0.f);

    for (int k = 0; k < K; k += WMMA_K)
    {
        int a_row = warpM * WMMA_M;
        int a_col = k;
        int b_row = k;
        int b_col = warpN * WMMA_N;

        if (a_row < M && a_col < K && b_row < K && b_col < N)
        {
            wmma::load_matrix_sync(a_frag, A + a_row * lda + a_col, lda);
            wmma::load_matrix_sync(b_frag, B + b_row * ldb + b_col, ldb);

            wmma::mma_sync(acc_frag, a_frag, b_frag, acc_frag);
        }
    }

    int c_row = warpM * WMMA_M;
    int c_col = warpN * WMMA_N;
    if (c_row < M && c_col < N)
    {
        wmma::load_matrix_sync(c_frag, C + c_row * ldc + c_col, ldc, wmma::mem_row_major);
        for (int i = 0; i < c_frag.num_elements; ++i)
        {
            c_frag.x[i] = alpha * acc_frag.x[i] + beta * c_frag.x[i];
        }
        wmma::store_matrix_sync(C + c_row * ldc + c_col, c_frag, ldc, wmma::mem_row_major);
    }
}
```























