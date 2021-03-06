---
title: Implementing block LU decomposition using MPI and BLACS
date: 2018-03-23T18:42:53+09:00
---

Recently I was tasked with implemented a block LU decomposition in parallel using
a block cyclic process distribution using BLACS and MPI. This decomposition would
then be extended to hierarchical matrices and would eventually work with dense matrices
instead of hierarchical. Thus we cannot use already implemented distributed LU factorization
methods like scalapack for this purpose.

In this post I would like to document my learnings about desinging the parallel algorithm
and installing the various libraries that are required for this purpose. Hopefully, the reader
will find something useful in this post too. This post will cover only LU factorization of dense
matrices. Hierarchical matrices will be covered in another post.

I have written about using the scalapack C++ interface for a simple block LU decomposition 
in [this](URL) post. 

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-generate-toc again -->
**Table of Contents**

- [Installing libraries](#installing-libraries)
- [Designing the algorithm](#designing-the-algorithm)
    - [Asynchronous block LU](#asynchronous-block-lu)
    - [Synchronous block LU](#synchronous-block-lu)
    - [Resources](#resources)
- [Implementation with MPI](#implementation-with-mpi)
    - [Block cyclic data distribution](#block-cyclic-data-distribution)
        - [Block cyclic nomenclature](#block-cyclic-nomenclature)
    - [ScaLAPACK protips](#scalapack-protips)
        - [Use of M and N in routines](#use-of-m-and-n-in-routines)
    - [BLACS protips](#blacs-protips)
        - [BLACS topologies](#blacs-topologies)
        - [BLACS general APIs](#blacs-general-apis)
    - [Asynchronous block LU](#asynchronous-block-lu)
    - [Synchronous block LU](#synchronous-block-lu)
- [Resources](#resources)
    - [BLACS](#blacs)

<!-- markdown-toc end -->

# Installing libraries

For this computation, we use MPICH and [BLACS](). While MPICH is easily installable on most
GNU/Linux distributions, the same cannot be said for BLACS.

I first tried downloading [BLACS sources]() and compiling the library, however it gave too
many compilation errors and was taking a long time to debug. Therefore, I resorted to using
the [ScaLAPACK installer](), which is a Python script that downloads the sources of BLACS,
LAPACK and ScaLAPACK, compiles all these libraries on your system and produces a single 
shared object file `libscalapack.a` which you can use for linking with your program. 
Since BLACS is included in the ScaLAPACK distribution, you can use the scalapack binary
directly for linking.

Just download the ScaLAPACK installer from the website and follow the instructions in the README for quick and easy installation.

# Designing the algorithm

## Asynchronous block LU

One problem that I faced when designing the algorithm is that when writing a CBLACS
program, you are basically writing the same code that is being run on multiple processes, 
however the data that is stored in variables is not the same for each process.

So it becomes important to write the program in such a way that maximum data is shared
between the processes but there is minimmum communication of things like the block
that is currently under process.

If it is a diagonal block, it simply factorizes the block into L & U parts and broadcasts
it to rows and columns.

If it is a row or column block, it listens for the broadcast from the diagonal block and mutliplies the contents that it receives with the data it posseses. It then broadcasts the multiplied matrix block accross the lower right block so that the block can be reduced.

It can be expressed with this line of code:
``` cpp
p2p_recv(recv_block, blocksize, rows[index] % N, rows[index] % N);
```

The source row and source col arguments (last two) are computed by keeping in mind
that we can compute the diagonal block of a particular block if we know the absolute
row number of the block. 

If is a block in the right lower block of the matrix (the A^ block), it waits for 
the broadcast from the row and column elements, multiplies the received data with
the stored data and over writes the stored data.

The computation and communication is mostly asynchronous. This means that there
needs to be some kind of a trigger to launch the computation or communication
tasks in a given process. 

A major problem is synchronization of successive diagonal matrix blocks. The
computation must proceed from the top left corner of the matrix until the lower
right corner. For this to work properly it is important that the diagonal blocks
do not compute and send their data unless the diagonal block to the upper left 
of the block has finished computing.

## Synchronous block LU

The main thing to take care of in synchronous block LU is that of the indexing of the data array
and the subsequent generation of the matrix. To demonstrate, here is what the matrix structure of
the synchronous block LU looks like:

<!-- insert that hand drawn image of sync block LU here -->

We can know the actual row and col number of the global matrix through the process ID and the
block number. The following lines of code are useful for this purpose:
``` cpp
// bcounter_i is a counter identifying the block row within a process
// bcounter_j is a counter identifying the block col within a process
// num_blocks_per_process is the number of blocks in a process
// myrow is the BLACS process row number
// mycol is the BLACS process col number
// block_size_per_process_r is the row size of each block within the process
// block_size_per_process_c is the col size of each block within the process

row_i = bcounter_i*num_blocks_per_process + myrow*block_size_per_process_r + i;
col_j = bcounter_j*num_blocks_per_process + mycol*block_size_per_process_c + j;
```

We can get the index number of the data array in the following manner:
``` cpp
int index = (bcounter_i*block_size_per_process_r + bcounter_j)*
    num_blocks_per_process +  i*process_block_size + j;
```

Before creating a full-fledged version of this code, I first made a simple code
that would calculate the LU decomposition in the case where there is only one
matrix block per process.

## Resources

Some resources that I found during this phase are as follows:
* [Designing and building parallel programs.](http://www.mcs.anl.gov/~itf/dbpp/)
* [Introduction to Parallel Computing.](http://www-users.cs.umn.edu/~karypis/parbook/)
* [Designing parallel programs course.](https://computing.llnl.gov/tutorials/parallel_comp/#Designing)
* [Lecture on parallel Gaussian from Berkeley](http://people.eecs.berkeley.edu/~demmel/cs267/lecture12/lecture12.html).
* [Parallelizing LU factorization.](https://cseweb.ucsd.edu/classes/sp07/cse262/Projects/260_fa06/Ricketts_SR.pdf) 

# Implementation with MPI

Each process should hold only the part of the matrix that it is working upon.

## Block cyclic data distribution

The block cyclic distribution is a central idea in the case of PBLAS and BLACS.
It is important to store the matrix in this configuration since it is the most 
efficient in terms of load balancing for most applications.

If you're reading a matrix from an external file it can get cumbersome to read 
into in a block cyclic manner manually. You do this with little effort using MPI IO.
Refer [this blog post](URL ) that describes this in detail along with C code.

For this code we generate the data on a per process basis.

### Block cyclic nomenclature

Its somewhat confusing how exactly the blocks are named. So here's the nomenclature
I'm using when talking about certain kinds of blocks:
* Process blocks :: blocks inside a process.
* Matrix blocks :: blocks of the global matrix.
* Matrix sub-blocks :: Each matrix block is divided into sub-blocks that are scattered
over processes. Each of these sub-blocks corresponds to a single process block.

## MPI communication protips

### Communicating lower triangular matrices with `MPI_Type_indexed`

For communicating arrays that are not contiguos in memory it is useful to use the
`MPI_Type_indexed` function for sending/receiving a non-contiguos array stored in memory.

If using `indexed` for sending, one must keep in mind that the array will be sent and
received in the exact same form that it is sent. So for example, if you have a 4x4 matrix
stored in an array of length 16 and you wish to send the lower triangle of this array,
you will need to reserve an array of length 16 at the receiving process too. The receiving
array will be populated at the positions that are indicated by the displacement and length
arrays that you commit when making the data type.

Here's a sample code for sending the lower triangle part of a 4x4 matrix stored as a 1D array:
``` cpp
// Sample program for demoing sending the lower triangle of a square
// matrix using types made by the MPI_Type_indexed function.

#include "mpi.h"
#include <iostream>
using namespace std;

int main()
{
  MPI_Init(NULL, NULL);
  int mpi_rank, mpi_size;
  MPI_Comm_size(MPI_COMM_WORLD, &mpi_size);
  MPI_Comm_rank(MPI_COMM_WORLD, &mpi_rank);

  double A[16], G[16];
  int displs[4] = {0, 4, 8, 12};
  int lens[4] = {1, 2, 3, 4};
  MPI_Datatype tril;
  MPI_Status status;

  for (int i = 0; i < 16; i++)
    A[i] = i+1;

  MPI_Type_indexed(4, lens, displs, MPI_DOUBLE, &tril);
  MPI_Type_commit(&tril);

  if (mpi_rank == 0) {
    MPI_Send(A, 1, tril, 1, 0, MPI_COMM_WORLD);
  }

  if (mpi_rank == 1) {
    MPI_Recv(G, 1, tril, 0, 0, MPI_COMM_WORLD, &status);

    for (int i = 0; i < 4; i++) {
      for (int j = 0; j < 4; j++) {
        if (j <= i)
          cout << G[i*4 + j] << " ";
        else
          cout << " ";
      }
      cout << endl;
    }
  }
  MPI_Type_free(&tril);
  MPI_Finalize();
}
```

### Communicating lower triangular matrices using MPI_Pack.



Link:

* https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=3&cad=rja&uact=8&ved=2ahUKEwj37KWzyNfcAhWCQN4KHcXACzUQFjACegQICBAC&url=http%3A%2F%2Fwww.mathnet.or.kr%2Fmathnet%2Fpaper_file%2Fiowa%2FGlenn%2FMPI.Derived.Types.Yanmei.April14.2005.doc&usg=AOvVaw3Qbf0Sbs6SWcAuddAZrFqG

## ScaLAPACK protips

### Use of M and N in routines

ScaLAPACK operates on a block cyclic data distribution. Most of the routines accept
two parameters: `M` and `N` that are described as the number of rows and cols of the
distributed submatrix sub(A). Its easy to get confused by thinking of these variables
as the dimensions of the _global_ matrix. However, since scalapack relies on a block
cyclic data distribution, the 'world' for all processes at _one_ time is basically one
matrix block which is spread over all the processes. Therefore, when calling scalapack
routines care must be taken to specify the dimensions of the matrix block in `M` and `N`
and not those of the global matrix.

If you see other code that does not rely on multiple sub-matrix blocks inside processes, 
they will usually pass the dimensions of the global matrix to the routine, which is correct
for that case since there is only one sub-matrix block per process.

## BLACS protips

### BLACS topologies

### BLACS general APIs

Similar to MPI, BLACS contains some routines for sending and receiving data in 
a point-to-point manner. They are as below:
* `gesd2d`: This routine is for point-to-point sending of data from one process to another. This routine is non-blocking by default (unlike `MPI_Send` which is blocking). It's prototype for the C interface is as follows:
``` cpp
void Cdgesd2d(
    int CBLACS_CONTEXT, // CBLACS context
    int M, // row size of matrix block
    int N, // col size of matrix block
    double* A, // pointer to matrix block
    int LDA, // leading dim of A (col size for C programs)
    int RDEST, // row number of destination process
    int CDEST // col number of destination process
);
```
* `trsd2d`: This routine is used for point-to-point sending of trapezoidal matrices.
* `gerv2d`: This routine is used for point-to-point receiving of general rectangular
matrices. This routine will block until the message is received. Its prototype looks like so:
``` cpp
void Cdgerv2d(
    int CBLACS_CONTEXT, // CBLACS conntext
    int M, // row size of matrix block
    int N, // col size of matrix block
    double *A, // pointer to matrix data.
    int LDA, // leading dim of A (col size for C)
    int RSRC, // process row co-ordinate of the sending process.
    int CSRC // process col co-ordinate of the sending process.
);
```

For broadcast receive, there is the `gebr2d` routine. This routine is particularly 
useful since it can broadcast over all processes, or a specific row or column. 
This can be helpful over using MPI directly since it allows us to easily broadcast
over rows or columns without having to define separate communicators.

The prototype of this routine is as follows:
``` cpp
// Cd stands for 'C double'
// ge is 'general rectangular matrix'
// br is 'broadcast receive'
void Cdgebr2d(
    int CBLACS_CONTEXT, // CBLACS context
    char* SCOPE, // scope of the broadcast. Can be "Row", "Column" or "All"
    char* TOP, // indicates communication pattern to use for broadcast.
    int M, // number of rows of matrix.
    int N, // number of columns of matrix.
    double* A, // pointer to matrix data.
    int LDA, // leading dim of matrix (col size for C)
    int RSRC, // process row co-ordinate of the process who called broadcast/send.
    int CSRC // process column co-ordinate of the process who called broadcast/send.
);
```

For broadcast send, there is the `gebs2d` routine. This is helpful for receiving broadcasts.
The prototype of this function is as follows:
``` cpp
Cdgebs2d(
    int CBLACS_CONTEXT, // CBLACS context.
    char* SCOPE, // scope of broadcast. can be "All", "Row" or "Column".
    char* TOP, // network topology to be used.
    int M, // num of rows of the matrix.
    int N, // num of cols of the matrix.
    double *A, // pointer to the matrix data.
    int LDA // leading dimension of A.
);
```
The `TOP` argument specifies the communication pattern to use. Leave it as a blank space
(`" "`) to use the default.

## Asynchronous block LU

## Synchronous block LU

In the asynchronous LU, it is assumed that the block size is equal to the processor size,
i.e each block of the matrix is limited to only a single processor. For synchronous LU 
decomposition, we take blocks which are spread out over multiple processors. To illustrate, 
see the below figure:

Four of the above colors represent a single block and each color represents a process. This
means that each block is spread out over 4 processes. This ensures that the processes are
always kept busy no matter the operation.

It should be remembered that scalapack expects the data to be in column-major format.
Therefore, it must be stored that way.

# Resources

## BLACS

* [Intel MKL BLACS resources](https://software.intel.com/en-us/mkl-developer-reference-c-blacs-routines).
* [Blog post detailing use of BLACS for scatter operations.](https://andyspiros.wordpress.com/2011/07/08/an-example-of-blacs-with-c/)
* [Netlib BLACS reference](http://www.netlib.org/blacs/BLACS/QRef.html#BS).
* [BLACS array-based communication](http://www.netlib.org/blacs/BLACS/Array.html).
* [BLACS user manual](http://www.netlib.org/lapack/lawnspdf/lawn94.pdf). 
* [BLACS communication topologies](http://www.netlib.org/blacs/BLACS/Top.html).
* [Using PBLAS for matrix multiplication.](https://scicomp.stackexchange.com/questions/1688/how-do-i-use-scalapack-pblas-for-matrix-vector-multiplication) 
* [PBLAS rountines overview from Intel.](https://software.intel.com/en-us/mkl-developer-reference-c-pblas-routines-overview)
* [ScaLAPACK pdgemm matrix multiplication example.](http://www.nersc.gov/users/software/programming-libraries/math-libraries/libsci/libsci-example/) 
* [Presentation about Scalapack/PBLAS/BLACS with good details on usage.](http://www.training.prace-ri.eu/uploads/tx_pracetmo/scalable_linear_algebra.pdf) 
* [Block cyclic data distribution (netlib).](http://www.netlib.org/utk/papers/scalapack/node8.html)
* [BLACS Topology.](http://www.netlib.org/blacs/BLACS/Top.html) 
