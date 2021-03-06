# clstm

[![Join the chat at https://gitter.im/tmbdev/clstm](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/tmbdev/clstm?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

A small C++ implementation of LSTM networks, focused on OCR.
The only essential external dependencies are Eigen and STL.

For I/O, having HDF5 is useful. For plotting, you need Python and the
ZMQ library.

To build a standalone C library, run

    scons
    sudo scons install

For debugging, you can compile with `debug=1`

To build the Python extension, run

    python setup.py build
    sudo python setup.py install

or

    scons
    sudo scons pyinstall

# Documentation / Examples

You can find some documentation and examples in the form of iPython notebooks in the `misc` directory
(these are version 3 notebooks and won't open in older versions).

You can view these notebooks online here:
http://nbviewer.ipython.org/github/tmbdev/clstm/tree/master/misc/

# C++ API

The `clstm` library operates on the Sequence type as its fundamental
data type, representing variable length sequences of fixed length vectors.
Internally, this is represented as an STL vector of Eigen dynamic vectors

    typedef stl::vector<Eigen::VectorXf> Sequence;

NB: This will be changed to an Eigen::Tensor

Networks are built from objects implementing the `INetwork` interface.
The `INetwork` interface contains:

    struct INetwork {
        Sequence inputs, d_inputs;      // input sequence, input deltas
        Sequence outputs, d_outputs;    // output sequence, output deltas
        void forward();                 // propagate inputs to outputs
        void backward();                // propagate d_outputs to d_inputs
        void update();                  // update weights from the last backward() step
        void setLearningRate(Float,Float); // set learning rates
        ...
    };

Network structures can be hierarchical and there are some network 
implementations whose purpose it is to combine other networks into more
complex structures.

    struct INetwork {
        ...
        vector<shared_ptr<INetwork>> sub;
        void add(shared_ptr<INetwork> net);
        ...
    };

The most important of these is the `Stacked` network, which simply
stacks the given set of networks on top of each other, using the ouput
from each network as the input to the next. 

There are a few utility functions for walking through the subnetworks,
states, and weights of a network, together with two hooks (`preSave`,
`postLoad`) to facilitate loading.

The implementations of the various networks are not exposed; instead of
`new Stacked()` use `make_Stacked()`.

In addition to these basic functions, there is also a small implementation
of CTC alignment.

The C++ code roughly follows the lstm.py implementation from the Python
version of OCRopus. Gradients have been verified for the core LSTM
implementation, although there may be still be bugs in other parts of
the code.

There is also a small multidimensional array class in `multidim.h`; that
isn't used in the core LSTM implementation, but it is used in debugging
and testing code, for plotting, and for HDF5 input/output. Unlike Eigen,
it uses standard C/C++ row major element order, as libraries like
HDF5 expect. (NB: This will be replaced with Eigen::Tensor.)

LSTM models are stored in protocol buffer format (`clstm.proto`), 
although adding new formats is easy. There is an older HDF5-based 
storage format.

# Layer and Network Instatiation

At its lowest level, layers are created by:

 - create an instance of the layer with `make_layer`
 - set any parameters (including `ninput` and `noutput`) as
   attributes
 - add any sublayers to the `sub` vector
 - call `initialize()`

The `layer(kind,ninput,noutput,args,sub)` function will perform 
these steps in sequence.

This can be used to construct network architectures in C++ pretty
easily. For example, the following creates a network that stacks
a softmax output layer on top of a standard LSTM layer:

    Network net = layer("Stacked", ninput, noutput, {}, {
        layer("LSTM", ninput, nhidden,{},{}),
        layer("SoftmaxLayer", nhidden, noutput,{},{})
    });

Note that you need to make sure that the number of input and
output units are correct.

The `make_net(kind,args)` function constructs
prefabricated networks (it also instantiates individual
layers, but does not have an option to add sublayers, since
that wouldn't make any sense).

# Python API

The `clstm.i` file implements a simple Python interface to clstm, plus
a wrapper that makes an INetwork mostly a replacement for the lstm.py
implementation from ocropy.

# Comand Line Drivers

There are several command line drivers:

  - clstmseq learns sequence-to-sequence mappings
  - clstmctc learns sequence-to-string mappings using CTC alignment
  - clstmtext learns string-to-string transformations

Note that most parameters are passed through the environment:

    lrate=3e-5 clstmctc uw3-dew.h5
    
See the notebooks in the `misc/` subdirectory for documentation on the parameters and examples of usage.

(You can find all parameters via `grep 'get.env' *.cc`.)

# TODO / UPCOMING

  - the HDF5 network save format will probably change
  - Lua and Torch bindings
  - more recurrent network types
  - replacement of mdarray with Eigen Tensors
  - 2D LSTM support
