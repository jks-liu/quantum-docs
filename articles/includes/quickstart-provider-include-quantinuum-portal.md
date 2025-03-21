---
author: bradben
ms.author: brbenefield
ms.date: 07/17/2023
ms.service: azure-quantum
ms.subservice: qdk
ms.topic: include
no-loc: [target, targets]
---

## Define a quantum circuit

Open a new cell and create a quantum circuit in the [OpenQASM](https://en.wikipedia.org/wiki/OpenQASM) representation. For example, the following example creates a Teleportation circuit:

```py
circuit = """OPENQASM 2.0;
include "qelib1.inc";
qreg q[3];
creg c0[3];
h q[0];
cx q[0], q[1];
cx q[1], q[2];
measure q[0] -> c0[0];
measure q[1] -> c0[1];
measure q[2] -> c0[2];
"""
```

## List all targets

[!INCLUDE [Quantinuum target name update](quantinuum-name-change.md)]

Use the `targets()`method to list all the targets in your workspace that can run your circuit, including the
current queue time and availability.

> [!NOTE]
> All the targets in your workspace may not be listed - only the targets that can accept a Cirq or OpenQASM circuit will be listed here. 

In a new cell, run:

```python
print("This workspace's targets:")
for target in service.targets():
     print(target)
```

```output
This workspace's targets:
<Target name="quantinuum.qpu.h1-1", avg. queue time=0 s, Degraded>
<Target name="quantinuum.sim.h1-1sc", avg. queue time=1 s, Available>
<Target name="quantinuum.qpu.h1-2", avg. queue time=217300 s, Unavailable>
<Target name="quantinuum.sim.h1-2sc", avg. queue time=0 s, Available>
<Target name="quantinuum.sim.h1-1e", avg. queue time=40 s, Available>
<Target name="quantinuum.sim.h1-2e", avg. queue time=64 s, Available>
<Target name="quantinuum.qpu.h1", avg. queue time=0 s, Unavailable>
<Target name="ionq.qpu", avg. queue time=229 s, Available>
<Target name="ionq.simulator", avg. queue time=3 s, Available>
<Target name="ionq.qpu.aria-1", avg. queue time=1136774 s, Available>
```

## Select a target and run your program

To check your circuit before running it on actual quantum hardware, you can use the [Quantinuum API validator](xref:microsoft.quantum.providers.quantinuum#api-validator), `quantinuum.sim.h1-1sc`, which returns a `Job` object. For more information, see [Azure Quantum job reference](xref:microsoft.quantum.optimization.job-reference).

Run the following code to set the target to the API Validator and submit your circuit with 500 shots:

```python
target = workspace.get_targets(name="quantinuum.sim.h1-1sc")
job = target.submit(circuit, num_shots=500)
```

Wait until the job is complete and then fetch the results.

```python
results = job.get_results()
print(results)
```

```output
........
{'c0': ['000',
'000',
'000',
'000',
'000',
'000',
'000',
...
]}
```

You can also visualize the results using [Matplotlib](https://matplotlib.org/stable/users/installing/index.html).

```python
import pylab as pl

pl.hist(results["c0"])
pl.ylabel("Counts")
pl.xlabel("Bitstring")
```

![Quantinuum job output](../media/quantinuum-results.png)

Looking at the histogram, you may notice that all the measurements are 0.  This is because that, while the Quantinuum API validator target ensures that your code will run successfully on Quantinuum hardware, it also returns 0 for every quantum measurement. For an accurate measurement of your circuit, you need to run it on quantum hardware.

## Estimate job cost to run on quantum hardware

Before running a job on actual quantum hardware, or a [quantum processing unit](xref:microsoft.quantum.target-profiles) (QPU), you can estimate how much it will cost to run. To estimate the cost of running a job on the QPU, you can use the `estimate_cost` method.

The following code changes the target to the System Model H1, `quantinuum.qpu.h1-1`, and uses the `estimate_cost` method to estimate the cost of running the job:

```python
target = workspace.get_targets(name="quantinuum.qpu.h1-1")
cost = target.estimate_cost(circuit, num_shots=500)

print(f"Estimated cost: {cost.estimated_total}")
```

```output
Estimated cost: 8.6
```

This prints the estimated cost in H-System Quantum Credits (HQCs).

For the most current pricing details, see [Azure Quantum pricing](xref:microsoft.quantum.providers-pricing#quantinuum), or view pricing options in the **Providers** blade of your workspace. To see your current credit status and usage, select **Credits and quotas**.

## Run on a Quantinuum QPU 

After running successfully on the API validator and estimating the QPU cost, it's time to run your circuit on the hardware. 

> [!NOTE] 
> The time required to run a circuit on the QPU depends on current queue times. You can view the average queue time for a target by selecting the **Providers** blade of your workspace.

Use the same `submit` method and operations that you used previously with the API Validator to submit your job and display the results:

```python
target = workspace.get_targets(name="quantinuum.qpu.h1-1")
job = target.submit(circuit, num_shots=500)
```

```python
results = job.get_results()
print(results)

pl.hist(results["c0"])
pl.ylabel("Counts")
pl.xlabel("Bitstring")
```

![Quantinuum job output qpu](../media/quantinuum-results-qpu.png)

Note that the measurements now are roughly split between 0 and 1.
