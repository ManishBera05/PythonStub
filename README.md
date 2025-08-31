# **Pythonmaker - GSoC 2025**

**Author:** Manish Bera
**Mentoring Organization:** The Document Foundation
**Project:** Python Code Auto-completion ([GSoC Project Link](https://summerofcode.withgoogle.com/programs/2025/projects/TVa3MD9k))

## Table of Contents

1.  [Project Overview](#project-overview)
2.  [The Problem Solved](#the-problem-solved)
3.  [About `pythonmaker`](#about-pythonmaker)
4.  [How to Use `pythonmaker`](#how-to-use-pythonmaker)
    -   [Prerequisites](#prerequisites)
    -   [`unoidl-write`: Compiling IDL to RDB](#unoidl-write-compiling-idl-to-rdb)
    -   [`pythonmaker`: Generating Stubs from RDB](#pythonmaker-generating-stubs-from-rdb)
5.  [Example: From IDL to Type-Safe Python](#example-from-idl-to-type-safe-python)
6.  [Automated Stub Generation in the Build System](#automated-stub-generation-in-the-build-system)
7.  [Unit Testing Strategy](#unit-testing-strategy)
8.  [Project Deliverables (Gerrit Patches)](#project-deliverables-gerrit-patches)
9.  [Future Work](#future-work)

## 1. Project Overview

This project introduces `pythonmaker`, a new command-line utility for the LibreOffice codebase. Its primary function is to parse UNO's Interface Definition Language (IDL) and generate Python type stubs (`.pyi` files).

The successful implementation of this tool modernizes the Python scripting experience for LibreOffice by enabling full IDE support, including intelligent auto-completion, static type checking, and real-time error detection.

## 2. The Problem Solved

Python is a powerful language for scripting and extending LibreOffice. However, developers have historically faced significant challenges:

-   **No Auto-completion:** LibreOffice's UNO objects are created dynamically at runtime, making it impossible for IDEs to statically analyze the API and provide code completion.
-   **No Static Type Checking:** This leads to "runtime-guess-and-check" development, where type-related errors (e.g., passing a `string` instead of an `integer`) are only discovered when the script is run.
-   **Lack of Discoverability:** Without IDE support, developers must constantly refer to the online API documentation, slowing down the development process.

`pythonmaker` solves these problems by providing the metadata that development tools need in a standard format (`.pyi` files).

## 3. About `pythonmaker`

`pythonmaker` is a C++ tool built within the `codemaker` module, following the architectural patterns of existing tools like `javamaker` and `cppumaker`.

**Key Features:**
-   **Comprehensive Type Coverage:** Generates stubs for all major UNO IDL constructs: enums, constants, typedefs, structs (plain and generic), exceptions, interfaces, services, and singletons.
-   **Pythonic & Robust Output:** The generated stubs are `mypy --strict` compliant, handle name collisions with Python keywords, manage complex cross-module imports, and correctly represent concepts like abstract interfaces and inheritance.
-   **Full Integration:** The tool is integrated into the LibreOffice `gbuild` system, allowing for automated generation of stubs for the entire public UNO API during the build process.

## 4. How to Use `pythonmaker`

The generation process is a two-step flow: `IDL -> RDB -> PYI`.

### Prerequisites

-   A compiled `unoidl-write` executable.
-   A compiled `pythonmaker` executable.
-   Both executables must be run from the `instdir/program/` directory due to DLL dependencies.

### `unoidl-write`: Compiling IDL to RDB

First, the human-readable `.idl` file must be compiled into a binary Registry Database (`.rdb`) format.

**Command Syntax:**
```bash
./unoidl-write -I <path_to_standard_IDLs> -C <path_to_dependency_RDBs> -o <output.rdb> <input.idl>
```

### `pythonmaker`: Generating Stubs from RDB

Next, `pythonmaker` consumes one or more `.rdb` files to generate the `.pyi` stubs.

**`pythonmaker --help` Output:**
```
pythonmaker.exe Version 1.0 

pythonmaker.exe - UNO Type Library to Python Stub Generator

About:
  This tool generates Python stub files (.pyi) from UNO type libraries (.rdb).
  These stubs enable static type checking and provide rich
  auto-completion for the LibreOffice API in modern code editors.

Usage:
  pythonmaker.exe -O <out_dir> [options] <input.rdb>...

Required Arguments:
  -O, --output-dir <path>
      The root directory for the generated Python stub package. The necessary
      module directories (e.g., 'com/sun/star/') will be created inside.

Filtering and Dependency Options:
  -T, --types <type_list>
      A semicolon-separated list of types to generate. Wildcards are supported.
      If omitted, all types from the primary input RDB files are generated.
  -X, --extra-types <file.rdb>
      Load an extra RDB for dependency resolution (e.g., for base classes)
      without generating stubs for its types. Can be specified multiple times.
  -nD
      No dependent types. Only generates stubs for the types explicitly matched
      by the -T filter.

Generation Control:
  -G              Generate a file only if it does not already exist.
  -Gc             Generate a file only if its content would change.

Other Options:
  -v, --verbose   Enable verbose logging of processed types and created files.
  -h, --help      Display this help message and exit.
  --version       Display version information and exit.
  @<cmdfile>      Read arguments from a file (one argument per line).

Examples:

  1. Generate all stubs from 'acme_api.rdb' into a 'stubs' directory:
     pythonmaker.exe -O ./stubs ./acme_api.rdb

  2. Generate stubs for only the 'org.company.widgets' module:
     pythonmaker.exe -O ./stubs -T "org.company.widgets.*" ./acme_api.rdb

  3. Generate a specific interface, using 'core_types.rdb' for dependencies
     (like com.sun.star.uno.XInterface), without generating 'core_types.rdb' itself:
     pythonmaker.exe -O ./stubs -T org.company.widgets.XButton -X ./core_types.rdb ./acme_api.rdb
```

## 5. Example: From IDL to Type-Safe Python

**Step 1: Create a custom IDL file (`person.idl`)**
```idl
module ExampleModule {
    struct Person {
        string name;
        long age;
        float height;
    };
};
```

**Step 2: Generate Python Stubs (assuming RDB is compiled)**
```bash
./pythonmaker -O ./stubs ./person.rdb
```

**Resulting Stub File (`stubs/ExampleModule/Person.pyi`):**
```python
# coding: utf-8
# Auto-generated by pythonmaker - DO NOT EDIT.
from __future__ import annotations

class Person(object):
    def __init__(self, name: str | None = ..., age: int | None = ..., height: float | None = ...) -> None: ...
    name: str
    age: int
    height: float
```

### The Impact in a Code Editor

The generated stub files immediately enable a modern development experience.

| Before `pythonmaker`                                                                                                      | After `pythonmaker`                                                                                                       |
| ------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| ![Before using pythonmaker stubs](https://github.com/ManishBera05/PythonStub/blob/main/before.png?raw=true)                  | ![After using pythonmaker stubs](https://github.com/ManishBera05/PythonStub/blob/main/after.png?raw=true)                   |
| - No auto-completion; developers must guess or look up method names. <br/>- No parameter information or type hints. <br/>- Type errors (like passing a `string` to a method expecting an `int`) are only found at runtime. | - **Intelligent Auto-completion:** All valid methods and properties are suggested. <br/>- **Static Type Checking:** The IDE flags type errors in real-time. <br/>- **API Discoverability:** Parameter info and type hints are shown on hover, boosting productivity. |

## 6. Automated Stub Generation in the Build System

As part of this project, `pythonmaker` has been fully integrated into the LibreOffice `gbuild` system. A new `CustomTarget` (`CustomTarget_python_stubs`) automatically executes `pythonmaker` during the build process. This target uses the official `offapi.rdb` and `udkapi.rdb` to generate a complete set of stubs for the entire public UNO API, placing them in the build's `workdir`.

## 7. Unit Testing Strategy

A robust, automated test suite has been developed to ensure `pythonmaker`'s correctness and prevent regressions. The strategy is based on "Golden File" testing:

1.  **Source IDL (`pythontypes.idl`):** A comprehensive IDL file located in `codemaker/tests/pythonmaker/idl/` contains test cases for every supported UNO construct and various edge cases.
2.  **Golden Files (`expected_pyi_stubs/`):** A directory of manually verified, `mypy`--compliant `.pyi` files that represent the perfect, expected output for the source IDL.
3.  **Test Runner (`test_pythonmaker.py`):** A Python script that automates the test. During a `make check` run, this script compiles the test IDL, executes `pythonmaker`, and performs a recursive `diff` between the generated stubs and the "golden" files. The test fails if any difference is found.

## 8. Project Deliverables (Gerrit Patches)

The work for this project is divided into three logical patches submitted to the LibreOffice Gerrit instance.

| Patch Description                        | Link                                                                                       |
| ---------------------------------------- | ------------------------------------------------------------------------------------------ |
| **Main Implementation of `pythonmaker`** | [gerrit.libreoffice.org/c/core/+/185498](https://gerrit.libreoffice.org/c/core/+/185498) |
| **Automated Unit Test Suite**            | [gerrit.libreoffice.org/c/core/+/188456](https://gerrit.libreoffice.org/c/core/+/188456) |
| **Build System Integration**             | [gerrit.libreoffice.org/c/core/+/189522](https://gerrit.libreoffice.org/c/core/+/189522) |

## 9. Future Work

While `pythonmaker` is now a complete and functional tool, future work could further enhance the developer experience:

-   **Add Documentation Support for Codemaker Tools (Bug 168155):** The `codemaker` tools currently do not process documentation comments (docstrings) from the IDL files. A significant future enhancement would be to parse these Doxygen-style comments and embed them as docstrings in the generated stubs. This would provide inline documentation directly within the IDE, greatly improving API discoverability. The implementation should be able to process simple tags like `<b>`, `<em>` and also create links for `@see` tags.

-   **CI for Stub Generation:** Create a CI job that automatically re-runs `pythonmaker` when the core UNO IDL files are changed, ensuring the stubs are always up-to-date.

-   **Extend Testing Pattern:** The flexible Python-based test runner developed for this project could be adapted to create similar "golden file" tests for `javamaker`, `cppumaker`, and other build tools.
