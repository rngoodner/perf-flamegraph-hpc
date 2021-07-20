# perf-flamegraph-hpc

Documentation of how to use Linux perf events on an HPC cluster.
The examples presented here use FIESTA as the HPC application and LLNL Lassen as the HPC cluster.
These also assume Brendan Gregg's flame graph perl scripts (https://github.com/brendangregg/FlameGraph) have been cloned to ~/opt.

## Synopsis

1. Wrap application with a shell script calling perf with unique output name
    ```bash
    #!/bin/bash
    perf record -F 99 -g -o $(hostname).${RANDOM}.perf.data /p/gpfs1/goodner2/cup-ecs-fiesta/build-spectrum-mpi-lassen-d8f360b/fiesta ./fiesta.lua --kokkos-num-devices=1
    ```
1. Call wrapped application with workload manager (lrun, srun, mpiexec, etc.)
    ```bash
    export WRAPPER=/path/to/wrapped/application
    lrun -N4 -T1 ${WRAPPER}
    ```
1. Convert perf.data to text script
    ```bash
    perf script -i lassen581.2741.perf.data > lassen581.2741.perf.data.script
    ```
1. Stack collapse the perf script output
    ```bash
    ~/opt/FlameGraph/stackcollapse-perf.pl lassen581.2741.perf.data.script > lassen581.2741.perf.data.script.collapsed
    ```
1. Generate SVG flame graph
    ```bash
    ~/opt/FlameGraph/flamegraph.pl lassen581.2741.perf.data.script.collapsed > lassen581.2741.svg
    ```

Note: The flame graph perl scripts can also conveniently use stdin, so steps 3-5 can be completed in 1 command like so:
```bash
perf script -i lassen581.2741.perf.data | ~/opt/FlameGraph/stackcollapse-perf.pl | ~/opt/FlameGraph/flamegraph.pl > lassen581.2741.svg
```

## Modifying results

Since we are just piping text results to various scripts it is easy to intercept and modify the results before outputting the final svg. Here are some useful examples.

### Sum results of all nodes

We can do this with `cat` after all script files are created.

```bash
# Create all script files for all nodes
find . -name "*.data" -exec bash -c 'perf script -i{} > {}.script' \;

# Make flame graph of all nodes combined
cat *.script | ~/opt/FlameGraph/stackcollapse-perf.pl | ~/opt/FlameGraph/flamegraph.pl > all-nodes.svg
```

### Filter functions that appear in graph

We can do this by using `grep` after collapsing the stacks and before creating the flame graph svg.
This example builds a graph containing only stacks that had a function with "mpi" in the name.

```bash
perf script -i lassen581.2741.perf.data | ~/opt/FlameGraph/stackcollapse-perf.pl | grep -iF "mpi" | ~/opt/FlameGraph/flamegraph.pl > lassen581.2741.mpi.svg
```

### Editing title

SVG files are just text files so we can easily change the default title to something more helpful.

```bash
sed -i 's/Flame Graph/Flame Graph - FIESTA, 1 node (lassen581)/g' lassen581.2741.svg
```

## More information about running perf with mpi

The main problem we needed to solve to run perf together with mpi on an hpc cluster was to ensure each perf call was able create an output file with a unique name, otherwise the default output name "perf.data" woud conflict for each node.
The way we did this was to wrap calling our application in a bash script so we could take advantage of `$(hostname)` and `${RANDOM}`.

Note that this will only work with one thread per node. If one is launching multiple tasks on a single node (or when using mpi on a laptop or workstation) the perf call can be done outside of the workload manager like `perf record -F 99 -g -- mpiexec -n 4 myprogram`. By default perf will follow and sample all child processes. The output will be a single "perf.data" file.

## References

The above information is what I discovered to be unique or tricky to using perf and flame graphs on an hpc cluster.
Standard usage of this technique is very well documented.
Here are the references I found useful for learning about perf and flame graphs.

* https://www.brendangregg.com/flamegraphs.html
* https://www.brendangregg.com/perf.html
* https://www.youtube.com/watch?v=D53T1Ejig1Q
* https://github.com/brendangregg/FlameGraph
* `man perf record`
