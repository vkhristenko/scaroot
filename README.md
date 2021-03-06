# ScaROOT
Call ROOT or arbitrary C++ code from Scala.

## Motivation

ROOT is a popular framework for high energy physics data. Most "big data" frameworks, such as Hadoop and Spark, are implemented in Java or Scala. ScaROOT allows you to call ROOT functions from Scala so that ROOT can be used to perform calculations in a big data workflow.

This is for accessing ROOT library functions. See [root2avro](https://github.com/diana-hep/root2avro) for a high-throughput data feed.

ScaROOT serves the same purpose as PyROOT, which provides access to ROOT functions in Python, though the interface differs because Python is more dynamic than the Java Virtual Machine (JVM).

## Examples

ScaROOT works by linking Scala traits (abstract interfaces) to C++ code that may contain ROOT calls.

```scala
import org.dianahep.scaroot.RootClass

// Scala trait (abstract interface):
trait ChiSqProb {
  def apply(chi2: Double, ndof: Int): Double
}

// C++ class definition that satisfies the interface:
val chiSqProbClass = RootClass[ChiSqProb]("""
class ChiSqProb {
public:
  double apply(double chi2, int ndof) {
    return ROOT::Math::chisquared_cdf(chi2, ndof);
  }
};
""")

// Create an instance:
val chiSqProb = chiSqProbClass.newInstance

// And use it:
chiSqProb.apply(53.8, 50)
0.6689797343068249
```

Or better yet, call the instance like a function. (In Scala, the `apply` method is the equivalent of C++'s `operator()`.)

```scala
chiSqProb(53.8, 50)
0.6689797343068249
```

The class instance can maintain state and you can mix Scala definitions with C++ definitions.

```scala
trait Histogram {
  private var name: String = ""
  private var bins: Int = 0
  private var low: Double = 0
  private var high: Double = 0
  def init(name: String, bins: Int, low: Double, high: Double) {
    this.name = name
    this.bins = bins
    this.low = low
    this.high = high
    initroot(name, bins, low, high)
  }
  def initroot(name: String, bins: Int, low: Double, high: Double)
  def fill(x: Double)
  def get(bin: Int): Double
  def getall: Array[Double] = 1 to bins map {i => get(i)} toArray
}

val histogramClass = RootClass[Histogram]("""
class Histogram {
private:
  TH1D *hist;
public:
  void initroot(const char *name, int bins, double low, double high) {
    hist = new TH1D(name, "", bins, low, high);
  }
  void fill(double x) {
    if (hist != nullptr) hist->Fill(x);
  }
  double get(int bin) {
    if (hist != nullptr)
      return hist->GetBinContent(bin);
    else
      return 0.0;
  }
};
""")

val hist1 = histogramClass.newInstance
hist1.init("myhist", 10, 0, 1)

0 until 10000 foreach {i => hist1.fill(scala.util.Random.nextDouble())}

hist1.getall
Array(950.0, 996.0, 960.0, 1001.0, 1010.0, 982.0, 1067.0, 956.0, 1049.0, 1029.0)
```

## How it works

`RootClass` is a parameterized type (equivalent of a C++ template); when you specify a concrete class, such as `RootClass[Histogram]`, it invokes a compile-time macro that creates a specialized class with all the external bindings built-in for speed. In Scala, "compile-time" may be when you press enter on the Scala prompt or the Spark prompt, or it may be when you build a deployable JAR.

These bindings connect to ROOT through [Java Native Access (JNA)](https://github.com/java-native-access/jna), which directly connects Java/Scala code and natively compiled libraries in the same process. Data are copied to and from ROOT without serialization or interprocess communication. (The data must be copied because (a) the Java garbage collector might otherwise move it and (b) it may need to be converted to little-endian.)

C++ code is compiled and linked at runtime using ROOT's `TInterpreter` interface to CINT or Cling (LLVM). Method calls are made using cached `TMethod` pointers, not string or hashmap lookups.

This _might_ be the fastest way possible to call runtime-generated C++ code from the JVM.

## Capabilities

   * The C++ code is not fixed until it is used to create an object. You can even generate it in Scala (see [string interpolation](http://docs.scala-lang.org/overviews/core/string-interpolation.html)).
   * `RootInstance` objects (created via `newInstance` above) can be inspected with Scala match patterns.
   * `RootClass` objects are serializable, so they can be submitted in a Spark job.

## Limitations

   * `RootInstance` objects are not serializable, since they might carry C++ data. They must be generated from a `RootClass`.
   * Scala traits cannot have constructors, so the C++ class must have a zero-argument constructor (implicitly or explicitly).
   * Class methods declared in Scala and defined in C++ can only have primitives for arguments and return values: `Boolean` (`bool` in C++), `Byte` (`char` in C++), `Short`, `Int`, `Long`, `Float`, `Double`, `String` (`char*` in C++), or an opaque `com.sun.jna.Pointer` to C++ data.
   * The JVM has no equivalent of unsigned primitives, so all arguments must be signed.
   * An installation of ROOT must be accessible on the library path (e.g. `LD_LIBRARY_PATH`) and that version must be compatible with the version compiled into ScaROOT.
   * Since this project makes used of Scala macros, it is bound to a Scala release (2.10 and not 2.11).

## How to install

[Install ROOT](http://root.cern.ch/) if necessary. Clone this repository and compile it with Maven:

```
cd scaroot
mvn install
```

In your project, include ScaROOT as a dependency. Its Maven coordinates are

```xml
<dependency>
  <groupId>org.dianahep</groupId>
  <artifactId>scaroot</artifactId>
  <version>0.1</version>
  <classifier>scala_2.10-root_6.06</classifier>
</dependency>
```

Note that the classifier is derived from the Scala version (hard-coded in `pom.xml`) and the ROOT version on your system (determined by running `root-config --version`).

## Roadmap

   1. ScaROOT needs to fail gracefully from errors. Currently it segmentation faults.
   2. Must catch C++ exceptions and propagate to Java exceptions.
   3. Ensure that all possible variants of primitive types (e.g. `int`, `Int_t`, `Int32_t`, etc.) are correctly mapped to Scala types.
   4. Test in Spark.
   5. Test performance, including an apples-to-apples comparison with PyROOT.
   6. A library of common classes (e.g. `TH1D`) should be wrapped.
   7. Expand the build process to include more architectures, versions of ROOT, and versions of Scala.

## Code overview

All of the Scala code is in `src/main/scala/org/dianahep/scaroot.scala`.

   * `RootClass` instances represent C++ classes for ROOT to compile. In Scala, they are instances created through the `RootClass.newClass[INSTANCE]` macro (or just `RootClass[INSTANCE]`, which is an alias).
   * `RootInstance` instances represent C++ objects, already compiled, instantiated, and ready to use.
   * `Param` tracks method parameter names and types. It has a subclass for each primitive type.
   * `Ret` tracks method return types. It has a subclass for each primitive type.
   * `Method` organizes `Param` and `Ret` descriptions in a way that can be used in Scala match patterns.

The C++ code and `Makefile` are in the `src/main/cpp` directory. Most of these functions are just pass-throughs to the related ROOT classes with C-style function names (note the `extern "C"`) for JNA. There's a hard-coded `execute` function for each number of parameters up to 22 (Scala's limit).

The C++ component compiles to a dynamic library (`.so` file) that acts as an interface between the JVM and ROOT. This `.so` file is bundled inside the ScaROOT JAR to simplify deployment, but note that the resulting JAR is not platform-independent.

Why not connect the JVM directly to ROOT with [BridJ](https://github.com/nativelibs4java/BridJ) instead of JNA? (Unlike JNA, BridJ is C++ aware and potentially wouldn't need an intermediate `.so`.) Because constructors and virtual functions are indexed in a dynamic library by number, and the only way to determine the correct number is to parse the header files. Directly binding to C++ functions with BridJ is essentially like linking by hand, and ultimately unsafe. For safety, BridJ would require intermediate linking code just like the JNA, and so I decided to go with the older, more widely used product.

The [JNAerator plugin](https://github.com/nativelibs4java/JNAerator/tree/master/maven-jnaerator-plugin) creates Java source code from the C++ header file, which is compiled into the Scala project.

## More examples

See the `src/test/scala` directory for unit tests.
