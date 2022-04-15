# [Manual Swift: Understanding the Swift/Objective-C Build Pipeline](https://bignerdranch.com/blog/manual-swift-understanding-the-swift-objective-c-build-pipeline/)

How’s Xcode compile Swift and Obj-C together? What wouldja do if you didn’t have `xcodebuild`?

We can look at “together” two different ways:

- Obj-C using Swift
- Swift using Obj-C

Today, we’re going to look at just the Obj-C using Swift flavor,
with the aim of giving you an overall understanding of the process in broad
strokes. We’ll dig in another day to see how those broad strokes are executed
in practice.

## Review: How does Obj-C code manage to use other Obj-C code?

To start with, let’s walk through how Obj-C code manages to use Obj-C.
It turns out that Obj-C using Swift builds on top of this,
so most of the heavy lifting actually happens in this part of the process.
That also means, once you get this, you’re 90% of the way to understanding how Obj-C manages to use Swift!

### Compile Each, Link All

Overall, building an executable where Obj-C code uses other Obj-C code is a two-step process:

- **Compile:** Each file gets compiled into an object file:
  `A.m` -> `A.o`

![compiling, illustrated](https://4bj5ozxzwtu3i1od01ulxmj1-wpengine.netdna-ssl.com/assets/img/blog/2016/11/Compiling-an-Obj-C-File.png)

- **Link:** All the object files get combined into an executable file by the
  linker:
  `A.o`, `B.o`, … -> `MyApp`

![linking, illustrated](https://4bj5ozxzwtu3i1od01ulxmj1-wpengine.netdna-ssl.com/assets/img/blog/2016/11/Linking-an-Obj-C-Executable.png)

### Headers promise, compilers trust, linkers verify

The compilation and combination (linking) steps rely on information from
header files.

**Headers promise**

Header files make promises about APIs. A line like:

```
NSString *NSTemporaryDirectory();
```

says:

> Trust me! There’s gonna be this function, name of `NSTemporaryDirectory`, and
> if you call it without any arguments at all, it’ll return an `NSString *`.

An interface declaration like

```
@interface Something: NSObject
- (BOOL)makeItSo: (NSError **)outError;
@end
```

says:

> Take it from me: When you need it, there’ll be a class named `Something` that
> is a kind of `NSObject`. All the methods in this interface block? They look
> just like that. You can depend on me!

**Compilers trust**

The compiler takes these header files at their word,
and checks the implementation files it’s fed against those promises.

![compilers look at header files](https://4bj5ozxzwtu3i1od01ulxmj1-wpengine.netdna-ssl.com/assets/img/blog/2016/11/Compiling-Detail.png)

If it runs into:

```
NSTemporaryDirectory(updatedTemporaryDirectoryPath);
```

It’ll pitch a fit:

> What do you mean, passing an argument to this!
> And there’s gonna be a return value, you totally blew that off! Won’t
> somebody please think of the return values?!

(Compilers are very excitable. It’s how they cope with the extreme attention to detail required to do their job. They don’t mean anything by it, though.)

If everything checks out,
the compiler will keep quiet, finish its work,
and leave behind an object code translation of the implementation code.

(This translated object code embeds additional assumptions based on those
promises when it comes to how to pass function arguments—
on the stack? in a register? in a vector register?—
and where to read out the return value from.
This leads smack into describing what an ABI is, though,
so let’s set that aside for another time.)

The object code takes it completely on faith that a definition for a function promised by a header will actually
exist in the end! The compiler outputs code that says, “Hey, go call this
function, I have no idea where it is, some header file promised it’d be there
though, so I’m just going to do a trust fall here.”

The result is object code with a pile of undefined references.
And all around those undefined references are assumptions about what goes
where, and how many arguments, and what kind of arguments and return values.

Compilers put a lot of faith in those header files, yeah?

<aside>

How many undefined references? See for yourself!

Pick an object file `A.o` and run `nm -u A.o`.
Out will come a list of all undefined names referenced by the file.

`nm` is a utility that formats the table of names referenced by an object file, called a *symbol table*, in a human-readable way. (NaMes, get it? nm?)
It can also filter that list down, like here, where `-u` asks it to
only list undefined names.

</aside>

**Linkers verify**

Compilers are all, “Hey, this will *totally* be there, some header promised!”

Linkers are all, “Show me the money.” They sweep together all the object files
and resolve those undefined references. If something doesn’t check out,
they table-flip, dump an error, and refuse to go any further:

```
> cc trust-me.m
Undefined symbols for architecture x86_64:
  "_thisWillTotallyBeThere", referenced from:
      _main in trust-me-c9e7ba.o
ld: symbol(s) not found for architecture x86_64
clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

All the object files get fed into the linker, which wires up their
external names and checks that everything really is defined that should be.
If so, the linker spits out an executable file.

![linkers resolve references](https://4bj5ozxzwtu3i1od01ulxmj1-wpengine.netdna-ssl.com/assets/img/blog/2016/11/Linking-Detail.png)

*There’s more going on, natch. We didn’t touch on modules or dynamic linking
(frameworks! dylibs! `dlopen`! oh my!). If you can’t get enough, check out
[Advanced Mac OS X Programming](https://nerdranchighq.wpengine.com/we-write/advanced-mac-osx-programming/).*

## Tying It Up: Using Swift from Obj-C

Phew, that was quite the diversion, yeah?
Lucky thing we’re mostly done.

### Short Version: We Make Swift Look Like Obj-C

Obj-C uses Swift by having Swift dress itself up to look like Obj-C.
It’s amazing what you can do with costuming these days.

Obj-C compilation relies on headers and object files.
Swift doesn’t use headers, and you’ve probably never run into a Swift object file, either.
How does Xcode solve this?

By generating a header and object file per Swift file, of course!

Each Swift file gets compiled into an object file and, for use from Obj-C,
a header file: `A.swift` -> `A.o`, `A.h`

This gives us what we need to run the usual Obj-C build-and-link compilation
pipeline:

- Compile each .m file with the bridging headers in sight
- Link all the object files (Swift and Obj-C) together into an executable file

### Bridging Headers, Plural

Because that header file bridges between the world of Obj-C and that specific Swift file,
it gets called a *bridging header.*

Now, you’ve probably run into that, “You’re adding Obj-C! To a Swift project!
Do you want a bridging header?” prompt in Xcode before. That’s *also*
a bridging header, but it’s not *these* (many! one per Swift file!) bridging
headers. A bridge goes from one shore to another, and that
briding-header-singular is for crossing from Swift over to Obj-C–land, while
these bridging-headers-plural are for getting from Obj-C–land over to
Swiftlandia.

### An Example

Here’s a small .swift file:

```
// ===[ CallMeFromObjC.swift ]===
// You have to import Foundation to be able to @objc anything.
import Foundation

// And a class has to inherit from NSObject to be visible to Obj-C.
// (If you wanna write a root class, you'll have to do it in Obj-C.)
public class CallMeFromObjC: NSObject {
    // Publish the API you want to use from Obj-C.
    public var name: String

    public init(name: String) {
        self.name = name
    }

    public func speak() {
        print("(self)'s name is: (name)")
    }
}
```

Run it through the compiler pipeline using this gnarly invocation:

```
install -d build
xcrun -sdk macosx10.12 
  swift -frontend -c -primary-file CallMeFromObjC.swift 
  -sdk /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.12.sdk/ 
  -module-name Bridgette 
  -emit-module-path build/Bridgette.swiftmodule 
  -emit-objc-header-path build/CallMeFromObjC.h 
  -enable-testing -enable-objc-interop -parse-as-library 
  -o build/CallMeFromObjC.o
```

And here’s the heart of the bridging header it generates:

```
SWIFT_CLASS("_TtC9Bridgette14CallMeFromObjC")
@interface CallMeFromObjC : NSObject
@property (nonatomic, copy) NSString * _Nonnull name;
- (nonnull instancetype)initWithName:(NSString * _Nonnull)name OBJC_DESIGNATED_INITIALIZER;
- (void)speak;
- (nonnull instancetype)init SWIFT_UNAVAILABLE;
@end
```

(That conveniently omits a pile of definitions
at the top that you can view in exhaustive detail
[in this gist of the full bridging header](https://gist.github.com/e9c2797a32e0522a46c25707ab7653b1).
It’s interesting to see what warnings they’ve elected to suppress.)

### Compiling Swift: What about imports?

Compiling and linking Obj-C against Swift also means compiling Swift and
linking it with other Swift.
When the compiler is generating an object file for a Swift file,
it’s taking a lot on faith. Again.

In this case, other Swift files in the project promise external definitions to
the compiler. Yup—the Swift files themselves are effectively header files!

Importing names from other files in a module is implicit in the source code:
you can freely use a type `B` defined in `B.swift` while you’re writing code
in `A.swift`, and you just expect it to be available.
If this were Obj-C, it’d be roughly like every `.m` file in your project
automagically gained a series of `#import`s of every `.h` file in your project.

Swift code does not have `#import`s to name the specific files to factor
into compilation of a single `.swift` file, though.
So instead of listing all the files in the module in the `.swift` file,
that list moves to the compiler invocation:
When you compile a single Swift file, the compiler invocation
lists every other Swift file in that module.

Good thing Xcode is writing those compiler invocations for you, yeah?

## Conclusion

Obj-C compilation is map–reduce with headers providing all the nitty-gritty context needed to carry out that map step. Map to object files; reduce to executable file.

Using Swift from Obj-C works by making Swift look like Obj-C,
so the normal Obj-C build pipeline can work its magic.
To make Swift look like Obj-C,
each Swift file gets compiled into
a corresponding header and object file.

This was a high-level overview. To really understand what’s going on,
we’ll need to grovel through Xcode’s build logs to see how all this plays
out in gory detail. But that’s a job for another day.