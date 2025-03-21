---
author: SoniaLopezBravo
ms.author: sonialopez
ms.date: 08/08/2023
ms.service: azure-quantum
ms.subservice: computing
ms.topic: include
no-loc: [target, targets]
---

## Resource estimation with Qiskit in Azure Quantum notebook

In this example, you'll create a quantum circuit for a multiplier based on the construction presented in [Ruiz-Perez and Garcia-Escartin (arXiv:1411.5949)](https://arxiv.org/abs/1411.5949) which uses the Quantum Fourier Transform to implement arithmetic. 

### Create a new notebook in your workspace

1. Log in to the [Azure portal](https://portal.azure.com/) and select your Azure Quantum workspace.
1. Under **Operations**, select **Notebooks**
2. Click on **My notebooks** and click **Add New**
3. In **Kernel Type**, select **IPython**.
4. Type a name for the file, and click **Create file**.

When your new notebook opens, it automatically creates the code for the first cell, based on your subscription and workspace information.

```python
provider = AzureQuantumProvider (
    resource_id = "",
    location = ""
)
```

### Load the required imports

First, you'll need to import an additional modules from `azure-quantum` and `qiskit`.

Click **+ Code** to add a new cell, then add and run the following code:

```python
from azure.quantum.qiskit import AzureQuantumProvider
from qiskit import QuantumCircuit, transpile
from qiskit.circuit.library import RGQFTMultiplier
from qiskit.tools.monitor import job_monitor
```

Create a backend instance and set the Resource Estimator as your target. 

```python
backend = provider.get_backend('microsoft.estimator')
```

### Create the quantum algorithm

You can adjust the size of the multiplier by changing the `bitwidth` variable. The circuit generation is wrapped in a function that can be called with the `bitwidth` value of the multiplier. The operation will have two input registers, each the size of the specified `bitwidth`, and one output register that is twice the size of the specified `bitwidth`.
The function will also print some logical resource counts for the multiplier extracted directly from the quantum circuit.

```python
def create_algorithm(bitwidth):
    print(f"[INFO] Create a QFT-based multiplier with bitwidth {bitwidth}")
    
    # Print a warning for large bitwidths that will require some time to generate and
    # transpile the circuit.
    if bitwidth > 18:
        print(f"[WARN] It will take more than one minute generate a quantum circuit with a bitwidth larger than 18")

    circ = RGQFTMultiplier(num_state_qubits=bitwidth, num_result_qubits=2 * bitwidth)

    # One could further reduce the resource estimates by increasing the optimization_level,
    # however, this will also increase the runtime to construct the algorithm.  Note, that
    # it does not affect the runtime for resource estimation.
    print(f"[INFO] Decompose circuit into intrinsic quantum operations")

    circ = transpile(circ, basis_gates=SUPPORTED_INSTRUCTIONS, optimization_level=0)

    # print some statistics
    print(f"[INFO]   qubit count: {circ.num_qubits}")
    print("[INFO]   gate counts")
    for gate, count in circ.count_ops().items():
        print(f"[INFO]   - {gate}: {count}")

    return circ
  ```
  
> [!NOTE]
> You can submit physical resource estimation jobs for algorithms that have no T states, but that have at least one measurement. 

### Estimate the quantum algorithm
  
Create an instance of your algorithm using the `create_algorithm` function. You can adjust the size of the multiplier by changing the `bitwidth` variable.

```python
bitwidth = 4

circ = create_algorithm(bitwidth)
```

Estimate the physical resources for this operation using the default assumptions. You can submit the circuit to the Resource Estimator backend using the `run` method, and then run `job_monitor` to await completion.

```python
job = backend.run(circ)
job_monitor(job)
result = job.result()
result
```

This creates a table that shows the overall physical resource counts. You can inspect cost details by collapsing the groups, which have more information. 

> [!TIP]
> For a more compact version of the output table, you can use `result.summary`.

For example, if you collapse the *Logical qubit parameters* group, you can more easily see that the error correction code distance is 15. 

|Logical qubit parameter | Value |
|----|---|
|QEC scheme | surface_code |
|Code distance |15 |
|Physical qubits | 450 |
|Logical cycle time    | 6us |
|Logical qubit error rate  |     3.00E-10 |
|Crossing prefactor |       0.03|
|Error correction threshold   |      0.01|
|Logical cycle time formula | (4 * `twoQubitGateTime` + 2 * `oneQubitMeasurementTime`) * `codeDistance`|
|Physical qubits formula	      | 2 * `codeDistance` * `codeDistance`|

In the *Physical qubit parameters* group you can see the physical qubit properties that were assumed for this estimation. For example, the time to perform a single-qubit measurement and a single-qubit gate are assumed to be 100 ns and 50 ns, respectively.

> [!TIP]
> You can also access the output of the Resource Estimator as a Python dictionary using the `result.data()` method.

For more information, see [the full list of output data](xref:microsoft.quantum.overview.resources-estimator#output-data) for the Resource Estimator.

#### Space-time diagrams

The distribution of physical qubits used for the algorithm and the T factories is a factor which may impact the design of your algorithm. You can visualize this distribution to better understand the estimated space requirements for the algorithm. 

```python
result.diagram.space
```
:::image type="content" source="../media/resource-estimator-space-diagram-qiskit.PNG" alt-text="Pie diagram showing the distribution of total physical qubits between algorithm qubits and T factory qubits. There's a table with the breakdown of number of T factory copies and number of physical qubits per T factory.":::

The space diagram shows the proportion of algorithm qubits and T factory qubits. Note that the number of T factory copies, 28, contributes to the number of physical qubits for T factories as $\text{T factories} \cdot \text{physical qubit per T factory}= 28 \cdot 18,000 = 504,000$.

You can can also visualize the time required to execute the algorithm, the T factory runtime and how many T factory invocations can run during the runtime of the algorithm. For more information, see [T factory physical estimation](xref:microsoft.quantum.learn-how-resource-estimator-works#t-factory-physical-estimation).

```python
result.diagram.time
```
:::image type="content" source="../media/resource-estimator-time-diagram-qiskit.PNG" alt-text="Diagram showing the number of T factory invocations during the runtime of the algorithm. There's also a table with the breakdown of the number of T factory copies, number of T factory invocations, T states per invocation, etc.":::

Since the T factoy runtime is 83 microsecs, the T factory can be invoked a total of 543 times during the runtime of the algorithm. One T factory produces one T state, and to execute the algorihtm you need a total of 15,180 T states. Therefore, you need 28 copies of the T factories executed in parallel. The total number of T factory copies is computed as $ \frac{\text{T states} \cdot \text{T factory duration}}{\text{T states per T factory} \cdot
\text{algorithm runtime}}=\frac{15,180 \cdot 83,200 \text{ns}}{1 \cdot 45,270,000 \text{ns}}=28$. Note that in the diagram, each blue arrow represents the 28 copies of the T factory repeatedly invoked 543 times.

> [!NOTE]
> You can't visualize the time and space diagrams in the same cell.

### Change the default values and estimate the algorithm

When submitting a resource estimate request for your program, you can specify some optional parameters. Use the `jobParams` field to access all the values that can be passed to the job execution and see which default values were assumed:

```python
result.data()["jobParams"]
```

```output
{'errorBudget': 0.001,
 'qecScheme': {'crossingPrefactor': 0.03,
  'errorCorrectionThreshold': 0.01,
  'logicalCycleTime': '(4 * twoQubitGateTime + 2 * oneQubitMeasurementTime) * codeDistance',
  'name': 'surface_code',
  'physicalQubitsPerLogicalQubit': '2 * codeDistance * codeDistance'},
 'qubitParams': {'instructionSet': 'GateBased',
  'name': 'qubit_gate_ns_e3',
  'oneQubitGateErrorRate': 0.001,
  'oneQubitGateTime': '50 ns',
  'oneQubitMeasurementErrorRate': 0.001,
  'oneQubitMeasurementTime': '100 ns',
  'tGateErrorRate': 0.001,
  'tGateTime': '50 ns',
  'twoQubitGateErrorRate': 0.001,
  'twoQubitGateTime': '50 ns'}}
 ```

These are the Target parameters that can be customized: 

* `errorBudget` - the overall allowed error budget
* `qecScheme` - the quantum error correction (QEC) scheme
* `qubitParams` - the physical qubit parameters
* `constraints` - the constraints on the component-level
* `distillationUnitSpecifications` - the specifications for T factories distillation algorithms

For more information, see [Target parameters](xref:microsoft.quantum.overview.resources-estimator#target-parameters) for the Resource Estimator.

#### Change qubit model

Next, estimate the cost for the same algorithm using the Majorana-based qubit parameter `qubit_maj_ns_e6`

```python
job = backend.run(circ,
    qubitParams={
        "name": "qubit_maj_ns_e6"
    })
job_monitor(job)
result = job.result()
result
```

You can nspect the physical counts programmatically. For example, you can show all physical resource estimates and their breakdown using the `physicalCounts` field 
in the result data. This will show the logical qubit error and logical T state error rates required to match the error budget. By default, runtimes are shown in nanoseconds.

```python
result.data()["physicalCounts"]
```

```output
{'breakdown': {'adjustedLogicalDepth': 6168,
  'cliffordErrorRate': 1e-06,
  'logicalDepth': 6168,
  'logicalQubits': 45,
  'numTfactories': 23,
  'numTfactoryRuns': 523,
  'numTsPerRotation': 17,
  'numTstates': 12017,
  'physicalQubitsForAlgorithm': 2250,
  'physicalQubitsForTfactories': 377568,
  'requiredLogicalQubitErrorRate': 1.200941538165922e-09,
  'requiredLogicalTstateErrorRate': 2.773848159551746e-08},
 'physicalQubits': 379818,
 'runtime': 61680000}
 ```

You can also explore details about the T factory that was created to execute the algorithm.

```python
result.data()["tfactory"]
```

```output
{'eccDistancePerRound': [1, 1, 5],
 'logicalErrorRate': 1.6833177305222897e-10,
 'moduleNamePerRound': ['15-to-1 space efficient physical',
  '15-to-1 RM prep physical',
  '15-to-1 RM prep logical'],
 'numInputTstates': 20520,
 'numModulesPerRound': [1368, 20, 1],
 'numRounds': 3,
 'numTstates': 1,
 'physicalQubits': 16416,
 'physicalQubitsPerRound': [12, 31, 1550],
 'runtime': 116900.0,
 'runtimePerRound': [4500.0, 2400.0, 110000.0]}
 ```

You can use this data to produce some explanations of how the T factories produce the required T states.

```python
data = result.data()
tfactory = data["tfactory"]
breakdown = data["physicalCounts"]["breakdown"]
producedTstates = breakdown["numTfactories"] * breakdown["numTfactoryRuns"] * tfactory["numTstates"]

print(f"""A single T factory produces {tfactory["logicalErrorRate"]:.2e} T states with an error rate of (required T state error rate is {breakdown["requiredLogicalTstateErrorRate"]:.2e}).""")
print(f"""{breakdown["numTfactories"]} copie(s) of a T factory are executed {breakdown["numTfactoryRuns"]} time(s) to produce {producedTstates} T states ({breakdown["numTstates"]} are required by the algorithm).""")
print(f"""A single T factory is composed of {tfactory["numRounds"]} rounds of distillation:""")
for round in range(tfactory["numRounds"]):
    print(f"""- {tfactory["numModulesPerRound"][round]} {tfactory["moduleNamePerRound"][round]} unit(s)""")
```

```output
A single T factory produces 1.68e-10 T states with an error rate of (required T state error rate is 2.77e-08).
23 copies of a T factory are executed 523 time(s) to produce 12029 T states (12017 are required by the algorithm).
A single T factory is composed of 3 rounds of distillation:
- 1368 15-to-1 space efficient physical unit(s)
- 20 15-to-1 RM prep physical unit(s)
- 1 15-to-1 RM prep logical unit(s)
```

#### Change quantum error correction scheme

Now, rerun the resource estimation job for the same example on the Majorana-based qubit parameters with a floqued QEC scheme, `qecScheme`.

```python
job = backend.run(circ,
    qubitParams={
        "name": "qubit_maj_ns_e6"
    },
    qecScheme={
        "name": "floquet_code"
    })
job_monitor(job)
result_maj_floquet = job.result()
result_maj_floquet
```

#### Change error budget

Let's rerun the same quantum circuit with an `errorBudget` of 10%.

```python
job = backend.run(circ,
    qubitParams={
        "name": "qubit_maj_ns_e6"
    },
    qecScheme={
        "name": "floquet_code"
    },
    errorBudget=0.1)
job_monitor(job)
result_maj_floquet_e1 = job.result()
result_maj_floquet_e1
```
