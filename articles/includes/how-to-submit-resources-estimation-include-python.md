---
author: SoniaLopezBravo
ms.author: sonialopez
ms.date: 08/07/2023
ms.service: azure-quantum
ms.subservice: computing
ms.topic: include
no-loc: [target, targets]
---

## Resource estimation with Q# and Python

In this example, you'll create a quantum multiplication algorithm and estimate its costs on a fault-tolerant quantum computer. You leverage the `azure_quantum` API to configure and submit the resource estimation job.

### Create a new Notebook in your workspace

1. Log in to the [Azure portal](https://portal.azure.com/).
1. In the left blade, select **Notebooks**.
1. Click **My Notebooks** and click **Add New**.
1. In **Kernel Type**, select **IPython**.
1. Type a name for the file, and click **Create file**. 

When your new notebook opens, it automatically creates the code for the first cell, based on your subscription and workspace information.

```py
from azure.quantum import Workspace
workspace = Workspace (
    subscription_id = <your subscription ID>, 
    resource_group = <your resource group>,   
    name = <your workspace name>,          
    location = <your location>        
    )
```

### Load the required imports

First, you need to select the Resource Estimator as the target.

Click **+ Code** to add a new cell, then add and run the following code:

```python
from azure.quantum.target.microsoft import MicrosoftEstimator, QubitParams
from azure.quantum.target.microsoft.target import MicrosoftEstimatorQubitParams
```

You also import the `qsharp` Python package to write Q# code in Jupyter cells, and the `Microsoft.Quantum.Numerics` package that is required for this example algorithm. Click **+ Code** to add a new cell.

```python
import qsharp
qsharp.packages.add("Microsoft.Quantum.Numerics")
```

### Create the quantum algorithm

Next, create a multiplier using the [MultiplyI](/qsharp/api/qsharp/microsoft.quantum.arithmetic.multiplyi) operation.  You can configure the size of the multiplier with a `bitwidth` parameter that can be passed as operation argument. The `EstimateMultiplication` operation will have two input registers, each the size of the specified `bitwidth`, and one output register that is twice the size of the specified `bitwidth`.

Click **+ Code** to add a new cell, then add the following Q# code using the [%%qsharp magic command](xref:microsoft.quantum.how-to.python-local#the-qsharp-magic-command).

```python
%%qsharp

open Microsoft.Quantum.Arithmetic;

operation EstimateMultiplication(bitwidth : Int) : Unit {
    use factor1 = Qubit[bitwidth];
    use factor2 = Qubit[bitwidth];
    use product = Qubit[2 * bitwidth];
    
    MultiplyI(LittleEndian(factor1), LittleEndian(factor2), LittleEndian(product));
}
```

> [!NOTE]
> You can submit physical resource estimation jobs for algorithms that have no T states, but that have at least one measurement.

### Estimate the quantum algorithm

Now, you estimate the physical resources for this operation using the default assumptions. You can submit the operation to the Resource Estimator target using the `qsharp.azure.execute` function. This function calls the `EstimateMultiplication` operation and passes the operation argument `bitwidth=8`.

```python
result = qsharp.azure.execute(EstimateMultiplication, bitwidth=8)
result
```

The `qsharp.azure.execute` function creates a result object, which can be used to display a table with the overall physical resource counts. You can inspect cost details by collapsing the groups, which have more information.

> [!TIP]
> For a more compact version of the output table, you can use `result.summary`.

For example, if you collapse the *Logical qubit parameters* group of the results table, you can see that the error correction code distance is 13.

|Logical qubit parameter | Value |
|----|---|
|QEC scheme  |surface_code |
|Code distance  |  13 |
|Physical qubits| 338 |
|Logical cycle time |   5us 200ns |
|Logical qubit error rate  |     3.00E-9 |
|Crossing prefactor  |0.03|
|Error correction threshold |      0.01|
|Logical cycle time formula | (4 * `twoQubitGateTime` + 2 * `oneQubitMeasurementTime`) * `codeDistance`|
|Physical qubits formula|2 * `codeDistance` * `codeDistance`|

In the *Physical qubit parameters* group, you can see the physical qubit properties that were assumed for this estimation. 
For example, the time to perform a single-qubit measurement and a single-qubit gate are assumed to be 100 ns and 50 ns, respectively.

> [!TIP]
> You can access the output of the Azure Quantum Resources Estimator as a Python dictionary using the `result.data()` method.

For more information, see [the full list of output data](xref:microsoft.quantum.overview.resources-estimator#output-data) of the Resource Estimator.

#### Space-time diagrams

The distribution of physical qubits used for the algorithm and the T factories is a factor which may impact the design of your algorithm. You can use `result.diagram.space` to visualize this distribution to better understand the estimated space requirements for the algorithm. 

```python
result.diagram.space
```

:::image type="content" source="../media/resource-estimator-space-diagram.PNG" alt-text="Pie diagram showing the distribution of total physical qubits between algorithm qubits and T factory qubits. There's a table with the breakdown of number of T factory copies and number of physical qubits per T factory.":::

The space diagram shows the proportion of algorithm qubits and T factory qubits. Note that the number of T factory copies, 15, contributes to the number of physical qubits for T factories as $\text{T factories} \cdot \text{physical qubit per T factory}= 15 \cdot 9,680 = 145,200$.

You can can also visualize the time required to execute the algorithm, the T factory runtime and how often the T factory invocations can be invoked during the runtime of the algorithm. For more information, see [T factory physical estimation](xref:microsoft.quantum.learn-how-resource-estimator-works#t-factory-physical-estimation).

```python
result.diagram.time
```

:::image type="content" source="../media/resource-estimator-time-diagram.PNG" alt-text="Diagram showing the number of T factory invocations during the runtime of the algorithm. There's also a table with the breakdown of the number of T factory copies, number of T factory invocations, T states per invocation, etc.":::

Since the T factory runtime is 57 microsecs, the T factory can be invoked a total of 54 times during the runtime of the algorithm. One T factory produces one T state, and to execute the algorithm you need a total of 800 T states. Therefore, you need 15 copies of the T factories executed in parallel. The total number of T factory copies is computed as $ \frac{\text{T states} \cdot \text{T factory duration}}{\text{T states per T factory} \cdot
\text{algorithm runtime}}=\frac{800 \cdot 57,200 \text{ns}}{1 \cdot 3,161,600 \text{ns}}=15$. Note that in the diagram, each blue arrow represents the 15 copies of the T factory repeatedly invoked 54 times.

> [!NOTE]
> You can't visualize the time and space diagrams in the same cell.

### Change the default values and estimate the algorithm

When submitting a resource estimate request for your program, you can specify some optional parameters. Use the `jobParams` field to access all the target parameters that can be passed to the job execution and see which default values were assumed:

```python
result['jobParams']
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

These are the target parameters that can be customized: 

* `errorBudget` - the overall allowed error budget
* `qecScheme` - the quantum error correction (QEC) scheme
* `qubitParams` - the physical qubit parameters 
* `constraints` - the constraints on the component-level
* `distillationUnitSpecifications` - the specifications for T factories distillation algorithms

For more information, see [Target parameters](xref:microsoft.quantum.overview.resources-estimator#target-parameters) for the Resource Estimator.

#### Change qubit model

Next, estimate the cost for the same algorithm using the Majorana-based qubit parameter, `qubitParams`, "qubit_maj_ns_e6".

```python
result = qsharp.azure.execute(EstimateMultiplication, bitwidth=8, 
            jobParams={
                "qubitParams": {
                    "name": "qubit_maj_ns_e6"
                }})
result
```

Inspect the physical counts programmatically. For example, you can show all physical resource estimates and its breakdown using the `physicalCounts` field in the result's data. Here you'll learn what logical qubit error and logical T state error rates are required to match the error budget. By default, runtimes are shown in nanoseconds.

```python
result['physicalCounts']
```

```output
{'breakdown': {'adjustedLogicalDepth': 608,
  'cliffordErrorRate': 1e-06,
  'logicalDepth': 608,
  'logicalQubits': 84,
  'numTfactories': 10,
  'numTfactoryRuns': 80,
  'numTsPerRotation': None,
  'numTstates': 800,
  'physicalQubitsForAlgorithm': 4200,
  'physicalQubitsForTfactories': 164160,
  'requiredLogicalQubitErrorRate': 9.790100250626566e-09,
  'requiredLogicalTstateErrorRate': 6.25e-07},
 'physicalQubits': 168360,
 'runtime': 6080000}
 ```

You can also explore details about the T factory that was created to execute the algorithm.

```python
result['tfactory']
```

```output
{'eccDistancePerRound': [1, 1, 3],
 'logicalErrorRate': 2.5244447992120037e-07,
 'unitNamePerRound': ['15-to-1 space efficient physical',
  '15-to-1 RM prep physical',
  '15-to-1 RM prep logical'],
 'numInputTstates': 20520,
 'numUnitsPerRound': [1368, 20, 1],
 'numRounds': 3,
 'numTstates': 1,
 'physicalQubits': 16416,
 'physicalQubitsPerRound': [12, 31, 558],
 'runtime': 72900.0,
 'runtimePerRound': [4500.0, 2400.0, 66000.0]}
```

You can use this data to produce some explanations of how the T factories produce the required T states.

```python
tfactory = result["tfactory"]
breakdown = result["physicalCounts"]["breakdown"]
producedTstates = breakdown["numTfactories"] * breakdown["numTfactoryRuns"] * tfactory["numTstates"]

print(f"""A single T factory produces {tfactory["numTstates"]} T state(s) with an error rate of {tfactory["logicalErrorRate"]:.2e} (required T state error rate is {breakdown["requiredLogicalTstateErrorRate"]:.2e}).""")
print(f"""{breakdown["numTfactories"]} copie(s) of a T factory are executed {breakdown["numTfactoryRuns"]} time(s) to produce {producedTstates} T states ({breakdown["numTstates"]} are required by the algorithm).""")
print(f"""A single T factory is composed of {tfactory["numRounds"]} rounds of distillation:""")
for round in range(tfactory["numRounds"]):
    print(f"""- {tfactory["numModulesPerRound"][round]} {tfactory["moduleNamePerRound"][round]} unit(s)""")
```

```output
A single T factory produces 1 T state(s) with an error rate of 2.52e-07 (required T state error rate is 6.25e-07).
10 copie(s) of a T factory are executed 80 time(s) to produce 800 T states (800 are required by the algorithm).
A single T factory is composed of 3 rounds of distillation:
- 1368 15-to-1 space efficient physical unit(s)
- 20 15-to-1 RM prep physical unit(s)
- 1 15-to-1 RM prep logical unit(s)
```

#### Change quantum error correction scheme

You can rerun the resource estimation job for the same example on the Majorana-based qubit parameters with a floqued QEC scheme, `qecScheme`.

```python
result_maj_floquet = qsharp.azure.execute(EstimateMultiplication, bitwidth=8,
            jobParams={
                "qubitParams": {
                    "name": "qubit_maj_ns_e6"
                },
                "qecScheme": {
                    "name": "floquet_code"
                }})
result_maj_floquet
```

#### Change error budget

Next, rerun the same quantum circuit with an `errorBudget` of 10%.

```python
result_maj_floquet_e1 = qsharp.azure.execute(EstimateMultiplication, bitwidth=8,
            jobParams={
                "qubitParams": {
                    "name": "qubit_maj_ns_e6"
                },
                "qecScheme": {
                    "name": "floquet_code"
                },
                "errorBudget": 0.1})
result_maj_floquet_e1
```

### Advanced analysis of the resource estimation results

Now that you’ve learned how to retrieve physical resource estimates and how to access them programmatically, you can perform more elaborate experiments. In this part, you'll evaluate the costs for the Quantum Fourier Transform based multiplier for different bit widths, qubit parameters, and quantum error correction codes.

Add a new cell and import the following required packages.

```python
import numpy as np                         # To store experimental data from job results
from matplotlib import pyplot as plt       # To plot experimental results
from matplotlib.colors import hsv_to_rgb   # To automatically find colors for plots
```

#### Batching with the Resource Estimator

A resource estimation job consist of two types of job parameters: [target parameters](xref:microsoft.quantum.overview.resources-estimator#target-parameters), that is qubit model, QEC schemes, and error budget; and, optionally, operation arguments, that is, arguments that can be passed to the QIR program. The Azure Quantum Resource Estimator allows you to submit jobs with multiple configuration of job parameters, or multiple *items*, as a single job to avoid rerunning multiple jobs on the same quantum program. For more information about batching, see [Run multiple configurations as a single job](xref:microsoft.quantum.work-with-resource-estimator#run-multiple-configurations-as-a-single-job).

In the following example, you use two of the six pre-defined qubit parameter models, and one customized model based on the model `"qubit_gate_ns_e3"` (accessed via the constant `QubitParams.GATE_NS_E3`), in which you'll set the error rates to $10^{-3.5}$. As operation arguments, you are choosing bit widths that are powers-of-2, ranging from 8 to 64.

```python
estimator = MicrosoftEstimator(workspace=workspace)

# target parameters
target_params = [
    ("Gate-based ns, 10⁻³", MicrosoftEstimatorQubitParams(name=QubitParams.GATE_NS_E3)),
    ("Gate-based ns, 10⁻³ᐧ⁵", MicrosoftEstimatorQubitParams(name=QubitParams.GATE_NS_E3, one_qubit_measurement_error_rate=0.00032, one_qubit_gate_error_rate=0.00032, two_qubit_gate_error_rate=0.00032, t_gate_error_rate=0.00032)),
    ("Gate-based ns, 10⁻⁴", MicrosoftEstimatorQubitParams(name=QubitParams.GATE_NS_E4))
]
# operation arguments
bitwidths = [8, 16, 32, 64]

# This is to access the names of the target parameters
names = [name for (name, _) in target_params]
```

> [!NOTE]
> Note that `target_params` are a tuple of the name and the qubit parameter. The qubit parameter is an instance of the class `MicrosoftEstimatorQubitParams`. The `name` field is the only required field. The rest of the fields are optional and will be set to the default values if not specified.

Next, you create a batching job, in which you create to the Resource Estimator all the combination of `target_params` and `bitwidths`. The bit width is assigned by accessing it through the `arguments` field of the item. Then, you submit the job with these items for the multiplication algorithm to the Azure Quantum Resource Estimator.

```python
params = estimator.make_params(num_items=len(bitwidths) * len(target_params))

for i, (_, target_param) in enumerate(target_params):
    for j, bitwidth in enumerate(bitwidths):
        params.items[i * len(bitwidths) + j].qubit_params = target_param
        params.items[i * len(bitwidths) + j].arguments["bitwidth"] = bitwidth

job = estimator.submit(EstimateMultiplication, input_params=params)
results = job.get_results()
```

#### Plotting the results

Now that you have all the results, you can extract some data from it, such as the number of physical qubits, the total runtime in nanoseconds, and the QEC code distance for the logical qubits. In addition to the total number of physical qubits, you can extract their breakdown into number of physical qubits for executing the algorithm, and the number of physical qubits required for the T factories that produce the required T states.

```python
qubits = np.zeros((len(names), len(bitwidths), 3))
runtime = np.zeros((len(names), len(bitwidths)))
distances = np.zeros((len(names), len(bitwidths)))

for bitwidth_index, bitwidth in enumerate(bitwidths):
    for name_index, name in enumerate(names):
        # Note that the results are ordered by target parameters first, then by bitwidth
        data = results.data(name_index * len(bitwidths) + bitwidth_index)

        qubits[(name_index, bitwidth_index, 0)] = data['physicalCounts']['physicalQubits']
        qubits[(name_index, bitwidth_index, 1)] = data['physicalCounts']['breakdown']['physicalQubitsForAlgorithm']
        qubits[(name_index, bitwidth_index, 2)] = data['physicalCounts']['breakdown']['physicalQubitsForTfactories']

        runtime[(name_index, bitwidth_index)] = data['physicalCounts']['runtime']

        distances[(name_index, bitwidth_index)] = data['logicalQubit']['codeDistance']
```

Finally, you can use [Matplotlib](https://matplotlib.org/) to plot the number of physical qubits and the runtime as bar plots, and the QEC code distances as a
scatter plot. For the physical qubits, the plot shows the partition into qubits required for the algorithm and qubits required for the T factories.

```python
fig, axs = plt.subplots(1, 3, figsize=(22, 6))

num_experiments = len(names)                         # Extract number of experiments form names (can be made smaller)
xs = np.arange(0, len(bitwidths))                    # Map bitwidths to numeric indexes for plotting
full_width = .8                                      # Total width of all bars (should be smaller than 1)
width = full_width / num_experiments                 # Fractional width of a single bar
xs_left = xs - (((num_experiments - 1) * width) / 2) # Starting x-coordinate for bars

# Split axes into qubit and runtime plots
ax_qubits, ax_runtime, ax_code_distance = axs

# Plot physical qubits
for i in range(num_experiments):
    ax_qubits.bar(xs_left + i * width, qubits[i,:,1], width, label=f"{names[i]} (Alg.)", color=hsv_to_rgb((i / num_experiments, 1.0, .8)))
    ax_qubits.bar(xs_left + i * width, qubits[i,:,2], width, bottom=qubits[i,:,1], label=f"{names[i]} (T fac.)", color=hsv_to_rgb((i / num_experiments, 0.3, .8)))
ax_qubits.set_title("#Physical qubits")
ax_qubits.set_xlabel("Bitwidth")
ax_qubits.set_xticks(xs)
ax_qubits.set_xticklabels(bitwidths)
ax_qubits.legend()

# Plot runtime
for i in range(num_experiments):
    ax_runtime.bar(xs_left + i * width, np.array(runtime[i,:]) / 1e6, width, label=names[i], color=hsv_to_rgb((i / num_experiments, 1.0, .8)))
ax_runtime.set_title("Runtime (ms)")
ax_runtime.set_xlabel("Bitwidth")
ax_runtime.set_xticks(xs)
ax_runtime.set_xticklabels(bitwidths)
ax_runtime.legend()

# Plot code distances
for i in range(num_experiments):
    ax_code_distance.scatter(xs, distances[i,:], label=names[i], marker='*', color=hsv_to_rgb((i / num_experiments, 1.0, 0.8)))
ax_code_distance.set_title("QEC code distance")
ax_code_distance.set_xlabel("Bitwidth")
ax_code_distance.set_xticks(xs)
ax_code_distance.set_xticklabels(bitwidths)
ax_code_distance.legend()

fig.suptitle("Resource estimates for multiplication")
plt.show()
```

:::image type="content" source="../media/plot-resource-estimation-multiplication.png" alt-text="Plot showing a bar chart of the physical resource estimation of physical qubits, runtime and code distance for three different settings of input parameters.":::

#### Accessing the results table

The result of the resource estimation job is displayed in a table with multiple results coming from the list of items. By default the maximum number of items to be displayed is five. To display a list of $N$ items where $N > 5$, use `results[0:N]`.  

Notice that the items are ordered by target parameters first, then by bitwidths. Therefore, all items with `bitwidth = 8` are at indices ${0, 4, 8}$, items with `bitwidth = 16` are at indices ${1, 5, 9}$, items with `bitwidth = 32` are at indices ${2, 6, 10}$, and items with `bitwidth = 48` are at indices ${3, 7, 11}$. The step size of 4 corresponds to the number of different bit widths.

You can display all estimation results for the first bit width in a side-by-side table.  

```python
bitwidth_index = 0 # items using the first bit width, that is bitwidth = 8
results[bitwidth_index::len(bitwidths)]
```

 :::image type="content" source="../media/advance-estimation-index0.PNG" alt-text="Screenshot of the table of results for the configuration with index 0.":::

You can also access individual results by providing a number as index. For example, `results[1]` to show the results table of the configuration with the first set of target parameters and bit width 16.

Further, you can plot all items in the result object using the `plot()` function. That function takes as optional parameter an array of labels for the plot's legend. Here, the labels are derived from the names and bitwidths, similar to how the items were created.

```python
results.plot(labels=[f"{name} ({bitwidth} bit)" for name in names for bitwidth in bitwidths])
```

 :::image type="content" source="../media/advance-estimation-plot.PNG" alt-text="Plot of runtime versus number of physical qubits for every configuration.":::
