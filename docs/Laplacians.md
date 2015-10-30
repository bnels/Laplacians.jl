[TOC]

# Laplacians

Laplacians.jl is a package for analyzing graphs in Julia.  Our emphsis is on things related to Laplacian linear sytems, such as solving equations, sparsifying, computing eigenvectors, partitioning, and other graph optimization tasks.

The graphs are represented as sparse matrices.  The particular class in Julia is called a SparseMatrixCSC.  The reasons for this are:

  * They are fast, and
  * We want to do linear algebra with them, so matrices help.

You can probably learn more about the CSC (Compressed Sparse Column) format by googling it.

So far, speed tests of code that we've written for connected components, shorest paths, and minimum spanning trees have been as fast or faster than the previous routines we could call from Matlab.



## To install Laplacians
you will need a number of packages.
You install these like

~~~julia
Pkg.add("PyCall")
Pkg.add("PyPlot")
Pkg.add("DataStructures")
~~~
I also recommend the Optim package.

I think you need to install matplotlib in python before PyPlot.
Look at this page for more information: https://github.com/stevengj/PyPlot.jl

I'm not sure if there are any others.  If you find that there are, please list them above.

## To use Laplacians

Examples of how to do many things in yinsGraph may be found in the IJulia notebooks.  These have the extensions .ipynb.  When they look nice, I think it makes sense to convert them to .html.

Right now, the notebooks worth looking at are:

* [yinsGraph](yinsGraph.md) - usage, demo, and speed tests (Laplacians was previously called yinsGraph)
* [Solvers](solvers.md) - code for solving equations.  How to use direct methods, conjugate gradient, and a preconditioned augmented spanning tree solver.


(I suggest that you open the html in your browser)


## About this Documentation

This documentation is still very rough.
It is generated by a combination of Markdown and semi-automatic generation.  The steps to generate and improve it are:

* Edit Markdown files in the `docs` directory.  For example, you could use MacDown to do this.
* If you want to add a new page to the documention, create one.  Edit the file mkdocs.yml so show where it should appear.
* Run `mkdocs build` in the root directory to regenerate the documenttion from the Markdown.
* Add Julia 0.4 docstrings to every function that you want to appear in the API.  You do not need to give the function template--that appears automatically.
* I (Dan) just created a script called `buildAPI.jl` that builds a nice API containing everything.  To use it, run

~~~julia
include("docs/buildAPI.jl")
~~~

* As it might also be useful to have file-specific APIs, I think that is the next thing we should generate.  For now, they are being generated
by code run like

~~~julia
include("docs/build.jl")
~~~

* This is a modification of code that I found in a package called LightGraph.jl.  I have very little idea of how it works, so it might be buggy.  We should find a better way later.  The Lexicon package looked like a good possibility, but it lacks a lot of features that I would like.  In particular, it can only organize APIs by module, not by file.

## Some Documentation that hasn't yet made it elsewhere.

### Fundamental Graph Algorithms:

*  `components` computes connected components, returns as a vector
*  `vecToComps` turns into an array with a list of vertices in each component
*  `shortestPaths(mat, start)`  returns an array of distances,
    and pointers to the node closest (parent array)
*  `kruskal(mat; kind=:min)`  to get a max tree, use `kind = :max`
    returns it as a sparse matrix.

### Solving Linear equations:

We have implemented Conjugate Gradient (cg) and the Preconditioned Conjugate Gradient (pcg).  These implementations use BLAS when they can, and a slower routine for data types like BigFloat.

To learn more, read [solvers.md](solvers.md).


## To develop Laplacians

Just go for it.
Don't worry about writing fast code at first.
Just get it to work.
We can speed it up later.
The yinsGraph.ipynb notebook contains some examples of speed tests.
Within some of the files, I am keeping old, unoptimized versions of code around for comparison (and for satisfaction).  I will give them the name "XSlow"

I think that each file should contain a manifest up top listing the functions and types that it provides.  They should be divided up into those that are for internal use only, and those that should be exported.  Old code that didn't work well, but which you want to keep for reference should go at the end.

### Using sparse matrices as graphs
The routines `deg`, `nbri` and `weighti` will let you treat a sparse matrix like a graph.

`deg(graph, u)` is the degree of node u.
`nbri(graph, u, i)` is the ith neighbor of node u.
`weighti(graph, u, i)` is the weight of the edge to the ith neighbor of node u.

Note that we start indexing from 1.

For example, to iterate over the neighbors of node v,
  and play with the attached nodes, you could write code like:

~~~julia
  for i in 1:deg(mat, v)
     nbr = nbri(mat, v, i)
     wt = weighti(mat, v, i)
     foo(v, nbr, wt)
  end
~~~

But, this turns out to be much slower than working with the structure directly, like

~~~julia
  for ind in mat.colptr[v]:(mat.colptr[v+1]-1)
      nbr = mat.rowval[ind]
      wt = mat.nzval[ind]
      foo(v, nbr, wt)
  end
~~~

* [ ] Maybe we can make a macro to replace those functions.  It could be faster and more readable.

### Parametric Types

A sparse matrix has two types associated with it: the types of its indices (some sort of integer) and the types of its values (some sort of number).  Most of the code has been written so that once these types are fixed, the type of everything else in the function has been too.  This is accomplished by putting curly braces after a function name, with the names of the types that we want to use in the braces.  For example,

~~~julia
shortestPaths{Tv,Ti}(mat::SparseMatrixCSC{Tv,Ti}, start::Ti)
~~~

`Tv`, sometimes written `Tval` denotes the types of the values, and `Ti` or `Tind` denotes the types of the indices.  This function will only be called if the node from which we compute the shortest paths, `start` is of type `Ti`.  Inside the code, whenever we write something like `pArray = zeros(Ti,n)`, it creates an array of zeros of type Ti.  Using these parameteric types is *much* faster than leaving the types unfixed.

### Data structures:

* `IntHeap` a heap that stores small integers (like indices of nodes in a graph) and that makes deletion fast.  Was much faster than using Julia's more general heap.

### Interface issue:
There are many different sorts of things that our code could be passing around.  For example, kruskal returns a graph as a sparse matrix.  But, we could use a format that is more specialized for trees, like the RootedTree type.  At some point, when we optimize code, we will need to figure out the right interfaces between routines.  For example, some routines symmetrize at the end.  This is slow, and should be skipped if not necessary.  It also doubles storage.

### Writing tests:
I haven't written any yet.  I'll admit that I'm using the notebooks as tests.  If I can run all the cells, then it's all good.

## Integration with other packages.

There are other graph packages that we might want to sometimes use.

* [Graphs.jl](http://github.com/JuliaLang/Graphs.jl) : I found this one to be too slow and awkward to be useful.
* [LightGraphs.jl](http://github.com/JuliaGraphs/LightGraphs.jl) : this looks more promising.  We will have to check it out.