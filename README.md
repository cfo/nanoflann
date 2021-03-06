![nanoflann](https://raw.githubusercontent.com/jlblancoc/nanoflann/master/doc/logo.png)

# nanoflann
[![Build Status](https://travis-ci.org/jlblancoc/nanoflann.svg?branch=master)](https://travis-ci.org/jlblancoc/nanoflann)


## 1. About 

*nanoflann* is a **C++ [header-only](http://en.wikipedia.org/wiki/Header-only) library** for building KD-Trees, mostly optimized for 2D or 3D point clouds. *nanoflann* does not require compiling or installing, just an `#include <nanoflann.hpp>` in your code.

This library is a *fork (and a subset)* of the [flann library](http://people.cs.ubc.ca/~mariusm/index.php/FLANN/FLANN) by Marius Muja and David G. Lowe, and born as a child project of [MRPT](http://www.mrpt.org/). Following the original license terms, *nanoflann* is distributed under the BSD license. Please, for bugs use the issues button or fork and open a pull request.


### 1.1. Obtaining the code

  * Clone this GIT repository.
  * List of [**stable releases**](https://github.com/jlblancoc/nanoflann/releases). Check out the [CHANGELOG](https://raw.githubusercontent.com/jlblancoc/nanoflann/master/CHANGELOG.txt)

Although nanoflann itself has not to be compiled, you can build some examples and tests with:

    sudo apt-get install build-essential cmake libgtest-dev libeigen3-dev
    mkdir build && cd build && cmake .. 
    make && make test
    

### 1.2. C++ API reference

  * Browse the [Doxygen documentation](http://nanoflann-reference.mrpt.org/svn/).

  * **Important note:** If L2 norms are used, notice that search radius and all passed and returned distances are actually *squared distances*.

### 1.3. Code examples

  * KD-tree look-up with `kdd_search()` and `radius_search()`: [pointcloud_kdd_radius.cpp](https://github.com/jlblancoc/nanoflann/blob/master/examples/pointcloud_kdd_radius.cpp)
  * KD-tree look-up on a point cloud dataset: [pointcloud_example.cpp](https://github.com/jlblancoc/nanoflann/blob/master/examples/pointcloud_example.cpp)
  * KD-tree look-up on a point cloud dataset with an external adaptor class: [pointcloud_adaptor_example.cpp](https://github.com/jlblancoc/nanoflann/blob/master/examples/pointcloud_adaptor_example.cpp)
  * KD-tree look-up directly on an `Eigen::Matrix<>`: [matrix_example.cpp](https://github.com/jlblancoc/nanoflann/blob/master/examples/matrix_example.cpp)
  * KD-tree look-up directly on `std::vector<std::vector<T> >` or `std::vector<Eigen::VectorXd>`: [vector_of_vectors_example.cpp](https://github.com/jlblancoc/nanoflann/blob/master/examples/vector_of_vectors_example.cpp)
  * Example with a `Makefile` for usage through `pkg-config` (for example, after doing a "make install" or after installing from Ubuntu repositories): [example_with_pkgconfig/](https://github.com/jlblancoc/nanoflann/blob/master/examples/example_with_pkgconfig/)
  * Example of how to build an index and save it to disk for posterior usage: [saveload_example.cpp](https://github.com/jlblancoc/nanoflann/blob/master/examples/saveload_example.cpp)


### 1.4. Why a fork?

  * **Execution time efficiency**: 
    * The power of the original `flann` library comes from the possibility of choosing between different ANN algorithms. The cost to pay is the declaration of pure virtual methods, which (in some circumstances) impose [run-time penalties](http://www.cs.cmu.edu/~gilpin/c%2B%2B/performance.html#virtualfunctions). In `nanoflann` all those virtual methods have been replaced by a combination of the [Curiously Recurring Template Pattern](http://en.wikipedia.org/wiki/Curiously_recurring_template_pattern) (CRTP) and inlined methods, which are much faster. 
    * For `radiusSearch()`, there is no need anymore to make a first call to determine the number of points within the radius, then call it again to get the data. By using STL containers for the output data, containers are automatically resized.
    * Users can (optionally) fix the problem dimensionality at compile-time via a template argument, thus allowing the compiler to fully unroll loops.
    * Also, `nanoflann` allows users to provide a precomputed bounding box of the data, if available, to avoid recomputing it again.
    * Indices of data points have been converted from `int` to `size_t`, which removes a limit when handling very large data sets.

  * **Memory efficiency**: Instead of making a copy of the entire dataset into a custom `flann`-like matrix before building a KD-tree index, `nanoflann` allows you to directly working on your class data via an **adaptor interface** which must be implemented in your class. 

Refer to the examples below or to the C++ API of [nanoflann::KDTreeSingleIndexAdaptor<>](http://nanoflann-reference.mrpt.org/svn/classnanoflann_1_1KDTreeSingleIndexAdaptor.html) for more info.


### 1.5. What can *nanoflann* do?

  * Building KD-trees with a single index (no randomized KD-trees).
  * Fast querying for neighbors on KD-trees. The two main entry points are:
    * [nanoflann::KDTreeSingleIndexAdaptor<>](http://nanoflann-reference.mrpt.org/svn/classnanoflann_1_1KDTreeSingleIndexAdaptor.html)`::knnSearch()`
    * [nanoflann::KDTreeSingleIndexAdaptor<>](http://nanoflann-reference.mrpt.org/svn/classnanoflann_1_1KDTreeSingleIndexAdaptor.html)`::radiusSearch()`
  * Define the dimensionality of the data at compile time, easing the generation of optimized code by the compiler.
  * Work with 2D and 3D point clouds or N-dimensional data sets.
  * Directly work with `Eigen::Matrix<>` classes (matrices and vectors-of-vectors).
  * Work with the distance metrics: 
    * `L1` (Manhattan) 
    * `L2` (Euclidean, favoring SSE2 optimization). 
    * `L2_Simple` (Euclidean, for low-dimensionality data sets like point clouds).
  * Save and load the built indices to disk.

### 1.6. What can't *nanoflann* do?

  * Using other distance metrics apart from L1 and L2.
  * Only the C++ interface exists: there is no support for C, MATLAB or Python.
  * There is no automatic algorithm configuration (as described in the original Muja & Lowe's paper).
  * The data in the source (data set) object is assumed *not to vary* during the execution of kd-tree queries. If this cannot be assured, the data set should be duplicated by the user and `nanoflann` applied to the copy. However, this is not the case in many practical applications.


------

## 2. Any help choosing the KD-tree parameters?

### 2.1. `KDTreeSingleIndexAdaptorParams::leaf_max_size`

A KD-tree is... well, a tree :-). And as such it has a root node, a set of intermediary nodes and finally, "leaf" nodes which are those without children. 

Points (or, properly, point indices) are only stored in leaf nodes. Each leaf contains a list of which points fall within its range.

While building the tree, nodes are recursively divided until the number of points inside is equal or below some threshold. **That is `leaf_max_size`**. While doing queries, the  "tree algorithm" ends by selecting leaf nodes, then performing linear search (one-by-one) for the closest point to the query within all those in the leaf.

So, `leaf_max_size` must be set as a **tradeoff**: 
  * Large values mean that the tree will be built faster (since the tree will be smaller), but each query will be slower (since the linear search in the leaf is to be done over more points).
  * Small values will build the tree much slower (there will be many tree nodes), but queries will be faster... up to some point, since the "tree-part" of the search (logarithmic complexity) still has a significant cost.

What number to select really depends on the application and even on the size of the processor cache memory, so ideally you should do some benchmarking for maximizing efficiency. 

But to help choosing a good value as a rule of thumb, I provide the following two benchmarks. Each graph represents the tree build (horizontal) and query (vertical) times for different `leaf_max_size` values between 1 and 10K (as 95% uncertainty ellipses, deformed due to the logarithmic scale).

  * A 100K point cloud, uniformly distributed (each point has (x,y,z) `float` coordinates):

![perf5_1e5pts_time_vs_maxleaf](https://raw.githubusercontent.com/jlblancoc/nanoflann/master/doc/perf5_1e5pts_time_vs_maxleaf.png)

  * A ~150K point cloud from a real dataset (`scan_071_points.dat` from the [Freiburg Campus 360 dataset](http://ais.informatik.uni-freiburg.de/projects/datasets/fr360/), each point has (x,y,z) `float` coordinates):

![perf5_1e5pts_time_vs_maxleaf_real_dataset](https://raw.githubusercontent.com/jlblancoc/nanoflann/master/doc/perf5_1e5pts_time_vs_maxleaf_real_dataset.png)

So, it seems that a `leaf_max_size` **between 10 and 50** would be optimum in applications where the cost of queries dominates (e.g. [ICP](http://en.wikipedia.org/wiki/Iterative_closest_point])). At present, its default value is 10.


### 2.2. `KDTreeSingleIndexAdaptorParams::checks` 

This parameter is really ignored in `nanoflann`, but was kept for backward compatibility with the original FLANN interface. Just ignore it.

-----

## 3. Performance

### 3.1. `nanoflann`: faster and less memory usage

Refer to the "Why a fork?" section above for the main optimization ideas behind `nanoflann`.

Notice that there are no explicit SSE2/SSE3 optimizations in `nanoflann`, but the intensive usage of `inline` and templates in practice turns into automatically SSE-optimized code generated by the compiler.


### 3.2. Benchmark: original `flann` vs `nanoflann`

The most time-consuming part of many point cloud algorithms (like ICP) is querying a KD-Tree for nearest neighbors. This operation is therefore the most time critical. 

`nanoflann` provides a ~50% time saving with respect to the original `flann` implementation (times in this chart are in microseconds for each query):

![perf3_query](https://raw.githubusercontent.com/jlblancoc/nanoflann/master/doc/perf3_query.small.png)

Although most of the gain comes from the queries (due to the large number of them in any typical operation with point clouds), there is also some time saved while building the KD-tree index, due to the templatized-code but also for the avoidance of duplicating the data in an auxiliary matrix (times in the next chart are in milliseconds):

![perf4_time_saved](https://raw.githubusercontent.com/jlblancoc/nanoflann/master/doc/perf4_time_saved.small.png)

These performance tests are only orientative. If you want to repeat them, read the instructions in [perf-tests](https://github.com/jlblancoc/nanoflann/tree/master/perf-tests)


----

## 4. Other KD-tree projects 

  * [FLANN](http://people.cs.ubc.ca/~mariusm/index.php/FLANN/FLANN) - Marius Muja and David G. Lowe (University of British Columbia).
  * [FASTANN](http://www.robots.ox.ac.uk/~vgg/software/fastann/) - James Philbin (VGG, University of Oxford).
  * [ANN](http://www.cs.umd.edu/~mount/ANN/) - David M. Mount and Sunil Arya (University of Maryland).
  * [libkdtree++](http://libkdtree.alioth.debian.org/) - Martin F. Krafft & others.

<br>

*Note: The project logo is due to [CedarSeed](http://www.iconarchive.com/show/patisserie-icons-by-cedarseed/Flan-icon.html)*
