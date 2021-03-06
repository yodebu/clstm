# -*- Python -*-
import os
import distutils.sysconfig

hgversion = os.popen("hg -q id").read().strip()

# A bunch of utility functions to make the rest of the SConstruct file a little simpler.

def die(msg):
    sys.stderr.write("ERROR "+msg+"\n")
    Exit(1)

def option(name,dflt):
    return (ARGUMENTS.get(name) or os.environ.get(name,dflt))

def findonpath(fname,path):
    for dir in path:
        if os.path.exists(os.path.join(dir,fname)):
            return dir
    raise die("%s: not found" % fname)

def protoc(target, source, env):
    os.system("protoc %s --cpp_out=." % source[0])
protoc_builder = Builder(action=protoc, src_suffix=".proto", suffix=".pb.cc")

# CLSTM requires C++11, and installes in /usr/local by default

prefix = option('prefix', "/usr/local")

env = Environment()
env.Append(CPPDEFINES={"HGVERSION" : '\\"'+hgversion+'\\"'})
env["BUILDERS"]["Protoc"] = protoc_builder

# With omp=1 support, Eigen and other parts of the code may use multi-threading.

if option("omp",0):
    env["CXX"]="g++ --std=c++11 -Wno-unused-result -fopenmp"
else:
    env["CXX"]="g++ --std=c++11 -Wno-unused-result"

# With debug=1, the code will be compiled suitable for debugging.

if option("debug",0):
    env.Append(CXXFLAGS="-g -fno-inline".split())
    env.Append(CCFLAGS="-g".split())
    env.Append(LINKFLAGS="-g".split())
else:
    env.Append(CXXFLAGS="-g -O3 -finline".split())
    env.Append(CCFLAGS="-g".split())

# Try to locate the Eigen include files (they are in different locations
# on different systems); you can specify an include path for Eigen with
# `eigen=/mypath/include`

if option("eigen","")=="":
    inc = findonpath("Eigen/Eigen","""
        /usr/include
        /usr/local/include/eigen3
        /usr/include/eigen3""".split())
else:
    inc = findonpath("Eigen/Eigen",[option("eigen")])
env.Append(CPPPATH=[inc])

# You can enable display debugging with `zmq=1`

if option("zmq",0):
    env.Append(LIBS=["zmqpp","zmq"])
else:
    env.Append(CPPDEFINES={'NODISPLAY' : 1})

env.Append(LIBS=["png","protobuf"])
env.Append(CPPDEFINES={'add_raw' : option("add_raw",'add')})

# We need to compile the protocol buffer definition as part of the build.

env.Protoc("clstm.proto")

# Build the CLSTM library.

libs = env["LIBS"]
libsrc = ["clstm.cc", "clstm_proto.cc", "clstm_prefab.cc",
          "extras.cc", "clstm.pb.cc"]
libclstm = env.StaticLibrary("clstm", source = libsrc)

programs = """clstmtext clstmfilter clstmfiltertrain clstmocr clstmocrtrain""".split()
for program in programs:
    env.Program(program,[program+".cc"],LIBS=[libclstm]+libs)

Alias('install-lib',
      Install(os.path.join(prefix,"lib"), libclstm))
Alias('install-include',
      Install(os.path.join(prefix,"include"), ["clstm.h"]))
Alias('install',
      ['install-lib', 'install-include'])

# If you have HDF5 installed, set hdf5lib=hdf5_serial (or something like that)
# and you will get a bunch of command line programs that can be trained from
# HDF5 data files. This code is messy and may get deprecated eventually.

if option("hdf5lib", "")!="":
    h5env = env.Clone()
    inc = findonpath("hdf5.h","""
        /usr/include
        /usr/local/include/hdf5/serial
        /usr/local/include/hdf5
        /usr/include/hdf5/serial
        /usr/include/hdf5""".split())
    h5env.Append(CPPPATH=[inc])
    h5env.Append(LIBS=["hdf5_cpp"])
    h5env.Append(LIBS=[option("hdf5lib", "hdf5_serial")])
    h5env.Prepend(LIBS=[libclstm])
    for program in "clstmctc clstmseq clstmconv".split():
        h5env.Program(program,[program+".cc"])


# You can construct the Python extension from scons using `pyswig=1`; however,
# the recommended way of compiling it is with "python setup.py build"

if option("pyswig", 0):
    swigenv = env.Clone( SWIGFLAGS=["-python","-c++"], SHLIBPREFIX="")
    swigenv.Append(CPPPATH=[distutils.sysconfig.get_python_inc()])
    swigenv.SharedLibrary("_clstm.so",
                          ["clstm.i", "clstm.cc", "extras.cc", "clstm.pb.cc"],
                          LIBS=libs)


destlib = distutils.sysconfig.get_config_var("DESTLIB")
Alias('pyinstall',
      Install(os.path.join(destlib, "site-packages"),
              ["_clstm.so", "clstm.py"]))
