# Linkers, Loaders, and Sharing

## Executable and Linkable Format (ELF)

Compilers produce, and linkers and loaders operate on object files. They have specific formats such as:

- assembler output (a.out)
- Common Object File (COFF)
- Mach-0
- ELF
- executables (e.g., a.out)
- core: virtual address space and register state of a process (debugging info)
- relocatable file: can be linked together with others to produce a shared library or executable
- shared object file: position independent code, used by the dymanic linker to create a process image

## Linkers and Loaders

Revisiting the compile chain:

1. compiler performs preprocessing (cpp(1))
2. includes and pull in macros and headers (cc(1))
3. actual compilation into assembly (as(1))
4. linking to create an executable (ld(1))

The linker takes multiple object files, resolves symbols to e.g., addresses in libraries, and produces an executable

The loader copies a program into main memory, possibly invoking the dynamic linker or run-time link editor to find the right libraries, resolve addresses of symbols, and relocate them

On some systems, the run-time link-editor is itself an executable, allowing direct invokation and passing another executable file as an argument

## Shared Libraries

A shared library:

- contains a set of callable C functions (i.e., implemtation of function prototypes defined in .h header files)
- code is position-independent (i.e., code can be executed anywhere in memory)
- libraries may be static or dynamic
- dynamically shared libraries can be loaded/unloaded at execution time or at will

How do they work?

- at link time, the linker resolves undefined symbols
- contents of object files and static libraries are pulled into the executable at link time
- contents of dynamic libraries are used to resolve symbols at link time, but loaded at execution time by the dynamic linker
- contents of dynamic libraries may be loaded at any time via explicit calls to the dynamic linking loader interface functions

### Statically Linked Shared Libraries

Let you build statically linked executables containing all the code needed to run the program

- created using ar(1)
- usually end in .a
- effectively a single file containing other (object) files
- linking statically pulls in all the code from the archives into the executable

### Dynamically Linked Shared Libraries

Let you define which code should be pulled in at execution time. Behavior can be influenced by changing the dynamic library without requiring the executable to be re-compiled or re-linked

- requires object files to be compiled into Position Independent Code (PIC)
- usually end in .so
- frequently have multiple levels of symlinks providing backwards compatibility / ABI definitions
- symbols are resolved at link time, but require the libraries to be found at execution time
- system- and user-specific configuration may influence resolution
