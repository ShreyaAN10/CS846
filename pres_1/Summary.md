# A Model for Detecting Faults in Build Specifications

## Summary

Parallelism and incrementality are necessary for faster build processes. However, enabling parallelism and incrementality while maintaining error-free build processes is challenging. Authors, therefore, propose `BuildFS`, a build process reasoning framework. It takes build specifications in the form of `build scripts` and actual/expected behaviour of build operations i.e `low-level file system operations`, and formally define 3 different types of errors prone to parallelization and incrementalization when file system operations violate the specifications of a build process.

> Execute single build -> Analyze execution -> Translate into BuildFS -> Look for violations (uncover faults)

#### Significant results
- effectiveness: good at uncovering faults (detected issues in 324 out of 612 Make and Gradle projects; 247 issues found in 47 open-source projects were later fixed)
- efficiency: faster than SOTA analysis tooks for Make projects (avg speedup of 74x)
- applicability: evaluated on 612 Make and Gradle projects; first to handle JVM-oriented build projects (Gradle); also applicable to other build systems such as Ninja, Bazel or Scala’s sbt. (Proof?)

## Background

- Build scripts involve source code compilation, application testing, construction of software artifacts (libraries and executables).
- Typically, a build is a sequence of tasks that work on input files to produce output files that will potentially be used by other tasks.

> CI = Continuous Integration
CI has forced build systems to be efficient and fast, while being reliable. To support this, modern build systems provide features like parallelism, caching, incrementality, lazy retrieval of project dependencies
> - Parallel builds reduce build times by processing independent build operations on multiple CPU cores(Linux Kernel or LLVM systems consisting of thousands of source files can be built in a few minutes).
  - Incrementality saves time and resources by executing only those build operations affected by a specific change in the codebase.
>  Make
  - Oldest build system in use today
  - Make provies domain-specific language to write definitions/rules (called Make rules) of how to build certain targets
  > `
    1 source.o: source.c
    2 gcc -c $^
    `
    source.c to source.o using GCC compiler
  - Make builds every target incrementally, meaning that it generates targets only when they are missing or when their dependent files are more recent than the target (using timestampes)
  - Tools like `cmake` and `GNU Autotools` can automatically generate make files
> Gradle
  - More than half (55%) of the Java projects on Github use Gradle
  - Gradle is also the preferred build tool for Kotlin and Android programs
  - Gradle offers parallelism and build cache, making it 2x faster than Maven
  - Gradle utilizes a task-based programming model; logic is assembled in a set of tasks
  - A task, in Gradle, describes a piece of work needed to be done as part of a build
  - Gradle files need to be enumerated to support incrementality; a task is executed only if its input/output files     have been changed (checksum-based checking against the latest build)
  `
  1 task extractZip {
  2 inputs.file "/file.zip"
  3 outputs.dir "/extractedZip"
  4 from zipTree("/file.zip")
  5 into "/extractedZip"}
  `
  Contents of `file.zip` extracted into directory `'extractedZip` 
 
- To avoid failures and race conditions, developers must specify all dependencies in their build scripts, so that the underlying build system does not process dependent tasks in the wrong sequence or in parallel (e.g., linking before compilation is erroneous). Similarly, for correct incremental builds, developers need to enumerate all source files that a build task relies on. This ensures that after an update to a source file, all the necessary tasks are re-executed to generate the new build artifacts reflecting this change. Build scripts are susceptible to faults because declaring all task dependencies is a challenging and error-prone task even with best practises and helpful tools.

Existing approaches to detecting faults in build specifications/definitions have 2 major drawbacks:
1. they are tailored for Make-based build systems (low external validity) -> not many techniques that apply to systems (other than Make) like JVM-based, Gradle that are also extensively used
2. they are especially slow (therefore, not efficient). For instance, `mkcheck` for about 100 Make files take hours and sometimes days

#### BuildFS
- "effective and efficient dynamic method for detecting faults in parallel and incremental builds"
- handles build execution of _any arbitrary_ build system as a sequence of tasks
  - each task takes some input files, performs file operations, returns output files
- inorder to detect flaws, takes into consideration
  1. build specification in the form of `build scripts`
  2. actual files accesses that take place during the build process i.e `low-level file system operations`
- formally defines 3 types of flaws:
  1. *Missing inputs (concerning incrementality)*: A build definition manifests a missing input issue, when a developer fails to define all input files of a particular build task. This leads to faulty incremental builds because whenever there is an update to any of the missing input files, the dependent build task is not executed by the build system. Consequently, the build system produces stale targets and outputs.
  `
  1 CXXFLAGS=-MD
  2 OBJS=CMetricsCalculator.o QualityMetrics.o
  3 qmcalc: $(OBJS) qmcalc.o
  4 gcc -g qmcalc.o $(OBJS) -o $@
  5 -include $(OBJS:.o=.d)
  `
  At line 5, while linking dependencies, the `qmcalc.o` file is missed thereby rendering the executable `qmcalc` stale
  2. *Missing outputs (concerning incrementality)*: The cause of the problem here is that a developer does not properly enumerate the output files of a task. As with missing inputs, this issue makes incremental builds skip the execution of some build tasks even if their outputs have changed. Note that missing outputs _do not appear in Make builds_, because Make considers only the timestamp of input files to decide if a target rule must be re-executed. However, in case of Gradle, missing outputs not only affect the correctness of the build process but also its efficiency. Gradle maintains build cache so as to not build previously built sources whose dependencies have not changed. Missing outputs leads to not maintaining the right state of the build cache which can slower the build process.
  3. *Ordering violations (concerning parallelism)*: Parallelism allows for independent tasks to run non-deterministically, implying that the build system is free to process independent processes in any order to achieve high performance. However, in the case of dependent processes, this might lead to race conditions if no constraints regarding the ordering of the dependent processes is specified. An ordering violation occurs when a developer does not specify ordering constraints between two dependent tasks.
  `
  1 apply plugin: "java"
  2 apply plugin: "com.github.johnrengelman.shadow"
  3 shadowJar { classifier = "" }
  `
  Above, is a Gradle snippet that generated the fat JAR of a Java application. To do so, the program uses two plugins: `java` and `shadowJar`. Since these two plugins are independent in their functioning (on a process-level), Gradle is free to schedule them in any order. However, in the example above, the name of the fat JAR generated by the task
  shadowJar conflicts with the name of the naive JAR produced by the task jar. This
  erroneous ordering results in incorrect output, i.e., the task jar overrides the contents of the JAR
  file produced by the task shadowJar. A fix to this problem is to create a fat JAR with a different
  name (e.g, changing its classifier at line 3 to "-all").

  > Fat JARs (also known as Uber JARs) are a type of Java archive file (JAR) that includes all the necessary dependencies and libraries required to run an application. Instead of having a JAR file with just the application's classes, a Fat JAR contains the compiled classes of the application along with all the dependent classes from external libraries. This creates a self-contained executable JAR file that can be easily distributed and executed without worrying about external dependencies.

  In order to detect such faults, the authors argue that we need an arbitrary(agnostic to the underlying build system) model with precise reasoning about build processes. 
  - *Applicability:* Existing models are not build system agnostic and therefore make specific assumptions of the build system they are designed for. 
  - *Precision:* Existing make-based solutions that aim to detect faults in incremental build processes have _low precision_ i.e they model build execution as a set of processes. However, a single process might have multiple build tasks (modern build systems like Gradle, Maven, Scala).
  ![Overconstrained Graph](/Critique_1/Overconstrained_Graphs.png)
  The authors use an example of two Gradle tasks, A and B, where A reads A.in and creates A.out, and B reads B.in and produces B.out. When applying an approach that works at the process level, the dependency graph (Figure 3a) is generated. In this approach, the main Gradle process runs both tasks, treating A.in and B.in as its inputs and A.out and B.out as its outputs. This essentially combines the two tasks into a single conceptual task. However, this results in an overconstrained graph, leading to inaccuracies. For instance, if there's a change in A.in, the analysis wrongly assumes that both A.out and B.out must be updated, even if A.in only affects A.out. Overconstrained graphs can produce numerous false positives and negatives.
  - *Efficiency:* Existing solutions are not efficient. To validate inferred file dependencies against build script specifications, previous approaches (e.g., mkcheck) initiate incremental builds by recompiling each source file. The verification process becomes notably slow due to the significant resources required.

  `BuildFS` tackles these challenges in the ways below,
  - Applicability: `BuildFS` first monitors the execution of a build (written in an arbitrary build system), and models it in `BuildFS`. Then, based on the notion of correctness for build executions (Section 3.4), our approach verifies whether the examined build is incorrect, and then reports all violations (if any).
  - Precision: `BuildFS` treats every build as a sequence of tasks rather than system processes. Every task corresponds to the execution of a build operation. For example, a task in buildfs stands for the execution of a target rule in Make and Ninja, a goal in Java Maven, or a Gradle task in Gradle. This tackles low precision introduced by prior work, because it enables us to relate every build task to its correct input and output files regardless of the internal behavior of the build tool (e.g., whether it spawns a separate process or not). For example, unlike the overconstrained dependency graph of Figure 3a, buildfs allows us to infer the precise graph shown in Figure 3b. buildfs separates file accesses based on which task they belong to.
  - Efficiency: `BuildFS` equips each task with a specification comprising (1) the files it is expected to consume, (2) the files it should produce, and (3) its dependencies on other tasks. Task dependencies ensure execution order. In addition to the specification, each task includes a definition detailing low-level file system operations performed during execution, such as file reads/writes and OS structure changes. This combination of high-level specifications with actual behavior makes `BuildFS` very efficient.

### BuildFS: Language Syntax, Semantics and Task Graph

#### Syntax
`
- A build execution (`b`) is a sequence of tasks `(t<sub>1</sub>, t<sub>2</sub>, ...)`, where each task is uniquely named (τ ∈ TaskName) and comprises a specification and a definition.
- The specification declares the input/output files and dependencies of a task. For instance, `task A (/file/in): /file/out after ⊥ = s` means task A consumes `/file/in`, produces `/file/out`, has no dependencies (`after ⊥`), and is defined by statements in `s`.
- Symbols `⊥` and `⊤` denote the absence of a value and a wildcard symbol representing any value, respectively. A declared output of `⊤` implies the task is valid to produce any file.
- A definition consists of one or more statements, including `sysOp in z = o`, which executes a system operation `o` in a process `z`. Each process defines a scope for file descriptor variables (`fd<sub>f</sub>`), pointing to file paths. Operations can introduce or delete file descriptor variables, and perform file system updates (`produce`, `consume`).
- An expression (`e ∈ Expr`) can be a constant path, a file descriptor variable, or `p at e`, allowing the interpretation of the path `p` relative to the expression `e`. The result of an expression is a path.
- The newproc `z<sub>1</sub> from z<sub>2</sub>` statement creates a fresh process (scope) `z<sub>1</sub>`, optionally copying all file descriptor variables from an existing process `z<sub>2</sub>` to `z<sub>1</sub>`. This models process forking.
`
![BuildFS Syntax](/Critique_1/Syntax.png)

#### Semantics
`
- Every task (t) in Task is evaluated on a state (σ) in State = Proc֒ → ProcFD.
- State (σ) is a map providing the file descriptor table (ProcFD) for each process.
- File descriptor table (π) in ProcFD = FileDesc֒ → Path is a map providing the file path for each file descriptor in a process.
- Conceptually, the state models the file descriptor feature of POSIX-compliant operating systems.
- The result of a task evaluation ⟦t⟧  is a new state along with the file accesses performed by the task.
- The domain of file accesses (r) in FileAcc = P(Path) × P(Path) is a pair containing the set of consumed and produced files by the task.
- Projections r↓c and r↓p provide the set of files consumed and produced by the task, respectively.
- Statements, operations, and expressions are evaluated using the file descriptor table (π) of the process.
- For example, after evaluating a buildfs task on state (σ = ⊥), a new state (σ ′) is obtained, and the set of consumed files (r↓c) is {"/f1/f3"}, while the set of produced files (r↓p) is {"/f2/f4"}.
`

#### Task Graph
`
- The task graph is a component that stores the input files, output files, and dependencies of every task as declared in build scripts.
- Computed by traversing the specification of each task in a buildfs program, collecting input/output files and task dependencies.
- Later used to ensure the correctness of a build execution.

Task Graph Definition `(G = (V, E))`:
  - Nodes (v ∈ V) are either a task (t ∈ Task) or a file (p ∈ Path).
  - Edges (E ⊆ V × V × L) define relationships with labels L = {in, out, before}:
    - Edge `p (in) → t` indicates file p as an input of task t.
    - Edge `t (out) → p` indicates task t producing file p.
    - Edge `t1 (before) → t2` indicates a task dependency, where t1 precedes t2.

Computing Task Graph:
- Given a build execution modeled as a buildfs program (Figure 4).
- Gradually compute the task graph by inspecting the specification of each task entry.
- Edges `(l→)` are constructed based on the header part of a task construct:
  - For task entry task `t1 (p1): p2 after t2`:
    1. Add `p1 in → t1` edge.
    2. Add `t1 out → p2` edge.
    3. Add `t2 before → t1` edge.
`
![Task Graph](/Critique_1/TaskGraph.png)

### Correctness of Build Executions
In order to detect faults in build executions, authors define two relationships:
1. **Subsumption**: The subsumption relation `p1 ⊑G p2` says that the file path `p1` is subsumed (contained) within the file path `p2`. Subsumption has two sub-relations: The relation `p1 ⊑G p2` holds when,
  1. [par-dir]:  `p2` is the parent directory of `p1`
  2. [indirect]: `p2` relies on `p1`, i.e., there is at least one task in the task graph `G` that produces `p2` using `p1`.
2. **Happens-Before**: The happens-before relation `t1 ≺G t2` states that the task `t1` is executed before `t2`.

#### Verifying Correctness of Tasks

1. Missing inputs: Given a task graph `G` and a state `σ`, a task `t ∈ Task = task τ k1: k2 after d = s` manifests a missing input on state `σ`, when `∃p ∈ r↓c p <del>⊑</del>G k1` i.e when there exists a file path `p` consumed by this task for which `p <del>⊑</del>G k1`, we say that the task has a missing input on the state `σ`.
  ![Missing Input](/Critique_1/MissingInput.png)
2. Missing outputs: Given a task graph `G` and a state `σ`, a task `t ∈ Task = task τ k1: k2 after d = s` manifests a missing input on state `σ`, when `∃p ∈ r↓p p <del>⊑</del>G k2` i.e when there exists a file path `p` produced by this task for which `p <del>⊑</del>G k2`, we say that the task has a missing input on the state `σ`.
3. Order violations: The definition for ordering violations focuses on checking if two tasks with conflicting file access are executed in the correct order. The happens-before relation ensures that tasks are executed in the right order, preventing scenarios where a task consuming a file is executed before the task creating it. For instance, if task t1 creates a file p, and this same file is consumed by another task t2, the relation t1 ≺+G t2 must hold. If t1 ≺+G t2 does not hold, the build system might execute t2 before t1, leading to potential issues such as accessing a non-existent file and resulting in build failure.

Therefore, a build execution is saif to be correct if
- all tasks result in valid succeeding states
- there is no order violation

### Testing Approach

![Testing Approach](/Critique_1/TestingApproach.png)

1. Generation: This process aims to delineate the execution boundaries for each task and gather information from build definitions, such as declared input/output files and dependencies. To achieve this, strategic instrumentation points are inserted before and after the execution of each build task, enhancing their execution by invoking specific native functions. These functions accept string arguments containing information from build definitions or signifying the initiation and completion of a task. Subsequently, the dynamic analysis identifies these function calls, extracts their arguments, and utilizes the information to construct task specifications. Intermediate file operations are also mapped to their respective tasks. 
2. Analysis and Fault Detection:
  1. *Missing Input\Output:* For every task `t` and file access `p`:
    - If `p` is consumed and the subsumption relation between `p` and the file inputs of `t` does not hold, report a missing input on `p`.
    - If `p` is produced and the subsumption relation between `p` and the file outputs of `t` does not hold, report a missing output on `p`.
  2. *Ordering Violation:* For every task tt and file access pp:
    - If `p` was accessed elsewhere (in another task `t'`) in the buildfs program, verify whether the execution order between `t` and `t'` is deterministic using the `happens-before` relation. If the `happens-before` relation is undefined, report an ordering violation.

  ### Misclassifications
  #### False Negatives
  ##### False Positives

  ### Implementation
  `BuildFS` is implemented in `OCaml` and uses `strace` to trace system operations. It supports two modes of operation:
  1. Offline (for debugging): Takes the `strace` output from a previous run of the system, creates its BuildFS file and provides the analysis
  2. Online: In this mode, `BuildFS` creates two processes:
    1. First process runs `strace` on the build command
    2. Second process reads the `strace` output and runs `BuildFS` generation and analysis
    These two processes communicate through pipes and run concurrently (effeciency!)
  
Gradle plugins are created to facilitate instrumentaion of Gradle build programs. A shell script named `fsmake-shell`
 is created to instrument Make builds. Example below.
 ![Build Files Instrumentation](/Critique_1/Instrumentation.png)

 ### Evaluation

 #### RQ1: (Effectiveness) What is the effectiveness of our approach in locating faults in build scripts?
 #### RQ2: (Fault Importance) What is the perception of developers regarding the detected faults?
 #### RQ3: (Fault Patterns) What are the main fault patterns?
 #### RQ4: (Performance) What is the performance of our approach?
 #### RQ5: (Comparison with state-of-the-art) Howdoes the proposed approach perform with regards to other tools, i.e., mkcheck?





