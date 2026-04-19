# Compile-time Performance

<cite>
**Referenced Files in This Document**
- [CMakeLists.txt](file://CMakeLists.txt)
- [QuillUtils.cmake](file://cmake/QuillUtils.cmake)
- [LogMacros.h](file://include/quill/LogMacros.h)
- [Common.h](file://include/quill/core/Common.h)
- [MacroMetadata.h](file://include/quill/core/MacroMetadata.h)
- [Utility.h](file://include/quill/Utility.h)
- [HelperMacros.h](file://include/quill/HelperMacros.h)
- [compile_time_bench.cpp](file://benchmarks/compile_time/compile_time_bench.cpp)
- [CMakeLists.txt](file://benchmarks/compile_time/CMakeLists.txt)
- [CMakeLists.txt](file://benchmarks/compile_time/qwrapper/CMakeLists.txt)
- [qwrapper.cpp](file://benchmarks/compile_time/qwrapper/include/qwrapper/qwrapper.cpp)
- [quill_static.cpp](file://examples/recommended_usage/quill_static_lib/quill_static.cpp)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Project Structure](#project-structure)
3. [Core Components](#core-components)
4. [Architecture Overview](#architecture-overview)
5. [Detailed Component Analysis](#detailed-component-analysis)
6. [Dependency Analysis](#dependency-analysis)
7. [Performance Considerations](#performance-considerations)
8. [Troubleshooting Guide](#troubleshooting-guide)
9. [Conclusion](#conclusion)
10. [Appendices](#appendices)

## Introduction
This document focuses on compile-time performance optimization in Quill. It explains how Quill’s design leverages compile-time mechanisms to minimize template instantiation overhead, reduce header inclusion costs, and optimize macro expansions. It also covers build system integration via CMake, compile-time logging level filtering, and practical examples for measuring and improving compilation performance in large-scale projects.

## Project Structure
Quill exposes a header-only interface library with optional compile-time toggles. The top-level CMake configuration defines a primary interface target and forwards compile definitions to consumers. Benchmarks and examples demonstrate realistic usage and compile-time cost measurement.

```mermaid
graph TB
subgraph "Build System"
A["CMakeLists.txt<br/>Defines interface target and options"]
B["QuillUtils.cmake<br/>Common compile options"]
end
subgraph "Library Target"
T["Target: quill (INTERFACE)"]
D["Compile Definitions<br/>-DQUILL_* toggles"]
H["Public Headers<br/>include/quill/*"]
end
subgraph "Benchmarks"
C["compile_time_bench.cpp<br/>Heavy logging usage"]
W["qwrapper (STATIC)<br/>Minimal wrapper around Quill"]
end
subgraph "Examples"
S["quill_static.cpp<br/>Recommended usage pattern"]
end
A --> T
A --> D
A --> H
B --> T
C --> W
W --> T
S --> T
```

**Diagram sources**
- [CMakeLists.txt](file://CMakeLists.txt)
- [QuillUtils.cmake](file://cmake/QuillUtils.cmake)
- [compile_time_bench.cpp](file://benchmarks/compile_time/compile_time_bench.cpp)
- [CMakeLists.txt](file://benchmarks/compile_time/CMakeLists.txt)
- [CMakeLists.txt](file://benchmarks/compile_time/qwrapper/CMakeLists.txt)
- [quill_static.cpp](file://examples/recommended_usage/quill_static_lib/quill_static.cpp)

**Section sources**
- [CMakeLists.txt](file://CMakeLists.txt)
- [QuillUtils.cmake](file://cmake/QuillUtils.cmake)

## Core Components
- Compile-time logging level filtering: Macro selection is pruned at compile-time based on a configurable threshold, eliminating higher-level logs from code generation.
- Macro metadata capture: Compile-time metadata reduces runtime overhead by encoding source location, function name, and format strings as constexpr data.
- Header inclusion minimization: Consumers can disable expensive macro features (function name, file info) to reduce template expansion and macro expansion costs.
- Template specialization helpers: Type formatter and codec specializations are provided via macros to avoid repeated template instantiation boilerplate.

Key compile-time toggle options exposed by the build system:
- QUILL_COMPILE_ACTIVE_LOG_LEVEL: Prunes macro branches for disabled log levels.
- QUILL_DISABLE_FUNCTION_NAME / QUILL_DETAILED_FUNCTION_NAME: Controls function name capture in metadata.
- QUILL_DISABLE_FILE_INFO: Controls file and line capture.
- QUILL_NO_EXCEPTIONS / QUILL_NO_THREAD_NAME_SUPPORT / QUILL_X86ARCH: Adjusts platform and feature support.

**Section sources**
- [LogMacros.h](file://include/quill/LogMacros.h)
- [Common.h](file://include/quill/core/Common.h)
- [MacroMetadata.h](file://include/quill/core/MacroMetadata.h)
- [CMakeLists.txt](file://CMakeLists.txt)

## Architecture Overview
The compile-time optimization pipeline centers on:
- Macro-driven logging entry points that conditionally compile based on QUILL_COMPILE_ACTIVE_LOG_LEVEL.
- MacroMetadata capturing compile-time source location and format string information.
- Optional compile definitions controlling function name and file info capture.
- A lightweight wrapper library in benchmarks to isolate Quill’s compile-time cost from application code.

```mermaid
sequenceDiagram
participant App as "Application Code"
participant Wrapper as "qwrapper.cpp"
participant Macros as "LogMacros.h"
participant Meta as "MacroMetadata.h"
participant Common as "Common.h"
App->>Wrapper : Call setup_quill(...)
Wrapper->>Macros : LOG_* macros (compile-time filtered)
Macros->>Meta : QUILL_DEFINE_MACRO_METADATA (constexpr)
Macros->>Common : QUILL_FUNCTION_NAME / QUILL_FILE_INFO (conditional)
Note over Macros,Meta : Higher log levels pruned at compile-time
```

**Diagram sources**
- [compile_time_bench.cpp](file://benchmarks/compile_time/compile_time_bench.cpp)
- [qwrapper.cpp](file://benchmarks/compile_time/qwrapper/include/qwrapper/qwrapper.cpp)
- [LogMacros.h](file://include/quill/LogMacros.h)
- [MacroMetadata.h](file://include/quill/core/MacroMetadata.h)
- [Common.h](file://include/quill/core/Common.h)

## Detailed Component Analysis

### Compile-time Logging Level Filtering
Quill’s logging macros are gated by a compile-time constant that disables entire macro branches for higher log levels. This eliminates branches and reduces the number of constexpr metadata instances generated.

```mermaid
flowchart TD
Start(["Macro Invocation"]) --> CheckLevel["Compare with QUILL_COMPILE_ACTIVE_LOG_LEVEL"]
CheckLevel --> |Within threshold| Emit["Emit constexpr metadata<br/>and call logger"]
CheckLevel --> |Exceeds threshold| Prune["Pruned at compile-time<br/>(empty statement)"]
Emit --> End(["Return"])
Prune --> End
```

**Diagram sources**
- [LogMacros.h](file://include/quill/LogMacros.h)

**Section sources**
- [LogMacros.h](file://include/quill/LogMacros.h)

### Macro Metadata Capture
MacroMetadata captures source location, function name, and format string at compile time. It includes helpers to detect named arguments and compute offsets for file name extraction, minimizing runtime work.

```mermaid
classDiagram
class MacroMetadata {
+source_location() const
+caller_function() const
+message_format() const
+line() const
+full_path() const
+file_name() const
+short_source_location() const
+log_level() const
+tags() const
+has_named_args() const
+event() const
-_contains_named_args(fmt) static
}
```

**Diagram sources**
- [MacroMetadata.h](file://include/quill/core/MacroMetadata.h)

**Section sources**
- [MacroMetadata.h](file://include/quill/core/MacroMetadata.h)

### Header Inclusion Patterns and Macro Expansion Costs
Quill’s Common.h defines QUILL_FUNCTION_NAME and QUILL_FILE_INFO based on compile-time toggles. Disabling function name capture or file info reduces macro expansion and template instantiation overhead.

```mermaid
flowchart TD
A["Include Common.h"] --> B{"QUILL_DISABLE_FILE_INFO?"}
B --> |Yes| C["QUILL_FILE_INFO = \"\""]
B --> |No| D["QUILL_FILE_INFO = __FILE__:__LINE__"]
A --> E{"QUILL_DISABLE_FUNCTION_NAME?"}
E --> |Yes| F["QUILL_FUNCTION_NAME = \"\""]
E --> |No| G{"QUILL_DETAILED_FUNCTION_NAME?"}
G --> |Yes| H["QUILL_FUNCTION_NAME = compiler-specific detailed"]
G --> |No| I["QUILL_FUNCTION_NAME = __FUNCTION__"]
```

**Diagram sources**
- [Common.h](file://include/quill/core/Common.h)

**Section sources**
- [Common.h](file://include/quill/core/Common.h)

### Template Specialization Helpers
Helper macros define type formatters and codecs, avoiding repetitive template boilerplate and reducing per-type template instantiations.

```mermaid
flowchart TD
Start(["User Type T"]) --> Def["Use QUILL_LOGGABLE_DEFERRED_FORMAT(T)"]
Def --> Spec1["Explicit template specialization<br/>fmtquill::formatter<T>"]
Def --> Spec2["Explicit template specialization<br/>quill::Codec<T>"]
Spec1 --> End(["Reduced template instantiation cost"])
Spec2 --> End
```

**Diagram sources**
- [HelperMacros.h](file://include/quill/HelperMacros.h)

**Section sources**
- [HelperMacros.h](file://include/quill/HelperMacros.h)

### Benchmark Harness for Compile-time Cost Measurement
The compile-time benchmark demonstrates heavy logging usage and measures compilation overhead when integrating Quill into a wrapper library.

```mermaid
sequenceDiagram
participant Bench as "compile_time_bench.cpp"
participant Wrapper as "qwrapper.cpp"
participant Quill as "Quill Headers"
Bench->>Wrapper : Include qwrapper.h
Wrapper->>Quill : Include Backend/Frontend/Logger
Bench->>Quill : LOG_* invocations
Note over Bench,Quill : Heavy macro expansion and template instantiation
```

**Diagram sources**
- [compile_time_bench.cpp](file://benchmarks/compile_time/compile_time_bench.cpp)
- [qwrapper.cpp](file://benchmarks/compile_time/qwrapper/include/qwrapper/qwrapper.cpp)

**Section sources**
- [compile_time_bench.cpp](file://benchmarks/compile_time/compile_time_bench.cpp)
- [CMakeLists.txt](file://benchmarks/compile_time/CMakeLists.txt)
- [CMakeLists.txt](file://benchmarks/compile_time/qwrapper/CMakeLists.txt)

### Recommended Static Library Usage Pattern
The example shows a minimal initialization pattern that starts the backend and creates a logger once, reducing repeated template instantiation in hot paths.

**Section sources**
- [quill_static.cpp](file://examples/recommended_usage/quill_static_lib/quill_static.cpp)

## Dependency Analysis
Quill’s interface target forwards compile definitions and links against Threads. Consumers inherit these definitions, enabling compile-time toggles without duplicating configuration.

```mermaid
graph LR
App["Application Target"] --> Q["quill::quill (INTERFACE)"]
Q --> Flags["Compile Definitions<br/>-DQUILL_*"]
Q --> Threads["Threads::Threads"]
```

**Diagram sources**
- [CMakeLists.txt](file://CMakeLists.txt)

**Section sources**
- [CMakeLists.txt](file://CMakeLists.txt)

## Performance Considerations
- Prefer compile-time filtering: Set QUILL_COMPILE_ACTIVE_LOG_LEVEL to prune higher log levels at compile-time.
- Minimize macro overhead: Disable function name capture and file info when not needed via QUILL_DISABLE_FUNCTION_NAME and QUILL_DISABLE_FILE_INFO.
- Reduce template instantiation: Use QUILL_LOGGABLE_DEFERRED_FORMAT or QUILL_LOGGABLE_DIRECT_FORMAT to specialize formatters and codecs once per type.
- Build system flags: Leverage set_common_compile_options for consistent warnings and compiler-specific flags; disable exceptions when appropriate via QUILL_NO_EXCEPTIONS.
- Static vs dynamic: The interface target avoids linking overhead; for heavy integration, consider a small wrapper static library (as shown in benchmarks) to amortize header inclusion costs across translation units.

[No sources needed since this section provides general guidance]

## Troubleshooting Guide
- Unexpectedly high compile times:
  - Verify QUILL_COMPILE_ACTIVE_LOG_LEVEL is set appropriately.
  - Ensure QUILL_DISABLE_FUNCTION_NAME and QUILL_DISABLE_FILE_INFO are enabled if not needed.
  - Confirm QUILL_NO_EXCEPTIONS is set to avoid exception/RTTI overhead.
- Macro conflicts:
  - Enable QUILL_DISABLE_NON_PREFIXED_MACROS to keep only QUILL_LOG_* macros.
- Platform-specific issues:
  - On MinGW, ucrtbase linkage is handled automatically.
  - On older GCC, stdc++fs linkage may be required.

**Section sources**
- [CMakeLists.txt](file://CMakeLists.txt)
- [QuillUtils.cmake](file://cmake/QuillUtils.cmake)

## Conclusion
Quill’s compile-time performance model relies on compile-time pruning of logging branches, constexpr metadata capture, and minimal macro expansion through selective feature toggles. By tuning compile definitions and leveraging helper macros, teams can significantly reduce compilation overhead while maintaining flexible logging capabilities.

[No sources needed since this section summarizes without analyzing specific files]

## Appendices

### Practical Integration Checklist
- Set QUILL_COMPILE_ACTIVE_LOG_LEVEL to the minimum required level.
- Disable function name and file info if not used.
- Use QUILL_LOGGABLE_DEFERRED_FORMAT or QUILL_LOGGABLE_DIRECT_FORMAT for user-defined types.
- Configure QUILL_NO_EXCEPTIONS if exceptions are not needed.
- Measure compile times with the benchmark harness and iterate on toggles.

**Section sources**
- [LogMacros.h](file://include/quill/LogMacros.h)
- [HelperMacros.h](file://include/quill/HelperMacros.h)
- [compile_time_bench.cpp](file://benchmarks/compile_time/compile_time_bench.cpp)
- [CMakeLists.txt](file://benchmarks/compile_time/CMakeLists.txt)