---
author: SoniaLopezBravo
description: Learn how the Azure Quantum Resource Estimator calculates estimates
ms.date: 08/07/2023
ms.author: sonialopez
ms.service: azure-quantum
ms.subservice: qdk
ms.topic: reference
no-loc: ['Q#', '$$v', target, targets]
title: Learn how the Resource Estimator works
uid: microsoft.quantum.learn-how-resource-estimator-works
---

# Learn how the Resource Estimator works

The [Azure Quantum Resource Estimator](xref:microsoft.quantum.overview.intro-resource-estimator) computes pre- and post-layout estimation of the logical resources. It takes a QIR quantum algorithm, for example a program written in Q#, Qiskit, or a QIR generator as [PyQIR](https://github.com/qir-alliance/pyqir), and a set of [target parameters](xref:microsoft.quantum.overview.resources-estimator#target-parameters) to evaluate the resource estimates of the quantum algorithm. Optionally, the Resource Estimator can take operation arguments, that is arguments that can be passed to the QIR program.

In this article, you'll learn the workflow of the Resource Estimator and how the [output data](xref:microsoft.quantum.overview.resources-estimator#output-data) is extracted at different levels of the evaluation of the quantum program. You'll also learn the assumptions taken into account for the simulation of the resource estimation and the common issues that may prevent the resource estimates job to complete.

## Workflow of the Resource Estimator

The following diagram shows the workflow of the Resource Estimator. The resource estimation is done based on the job parameters and quantum program, and the output data is extracted at different levels of the evaluation of the quantum program.

:::image type="content" source="media/resource-estimator-workflow.png" alt-text="Diagram showing the workflow of the resource estimator. The resource estimation is done based on the job parameters and quantum program, and the output data is extracted at different levels of the evaluation of the quantum program.":::

### Code distance and T factory estimation

The Resource Estimator takes the target parameters `{qubitParams, qecScheme, errorBudget, constraints}` to compute a resource estimation of qubit technology and architecture. It calculates the QEC code distance, and from it, the number of physical qubits needed to encode one logical qubit and the runtime of one logical depth or cycle.

The Resource Estimator uses a logical layer called *planar quantum ISA* that acts as the interface between the software and hardware layers. It abstracts the details of how QEC is implemented in the layer below, retaining only a set of fault-tolerant logical operations as its instruction set. For more information, see [Assessing requirements to scale to practical quantum advantage](https://arxiv.org/abs/2211.07629).

It is crucial that the quantum gate set of a fault-tolerant quantum computer is [universal](xref:microsoft.quantum.concepts.tfactories#universal-set-of-quantum-gates), meaning it is complete for quantum computing. In the Azure Quantum Resource Estimator, the preparation of T gates is crucial because the other quantum operations are not sufficient for universal quantum computation. This is achieved via the generation of T states using [T factories](xref:microsoft.quantum.concepts.tfactories#t-factories-in-the-azure-quantum-resource-estimator). The Resource Estimator calculates how many physical qubits are needed to run one T factory and for how long the T factory runs. These values are reserved and will be used in subsequent steps of the workflow.

### Pre-layout resource estimation

The Resource Estimator takes the QIR quantum program and computes a pre-layout estimation of the logical resources. It calculates the number of logical qubits, T gates, rotation gates, CCZ gates, CCiX gates, and measurements in the input quantum program. The number of T gates includes all T gates and adjoint T gates, but not T gates required to implement rotation gates, CCZ gates, or CCiX gates.

### Algorithmic logical estimation

In the previous step, the Resource Estimator has calculated the number of logical qubits in the input algorithm. Laying out the logical qubits in the presence of nearest-neighbor constraints requires extra logical qubits. In this step, the Resource Estimator takes the number of pre-layout logical qubits and calculates the number of logical qubits required for the algorithm after layout.

The Resource Estimator also calculates the algorithmic logical depth, that is the number of logical cycles required to run the algorithm. To execute the algorithm using multi-qubit measurements, you need one multi-qubit Pauli measurement for each single-qubit measurement, the single-qubit rotations, and the T gates, three multi-qubit Pauli measurements for each of the CCZ and CCiX gates in the input program, as well as the same number of T gates per rotation in multi-qubit measurements for each of the logical cycle.

Finally, the Resource Estimator calculates the total number of required T states. To execute the algorithm, you need one T state for each T gate, four T states for each CCZ and CCiX gates, and $ 0.53 \log_2(\text{Number of rotation gates(pre-layout)} / \text{Rotation synthesis error probability}) + 5.3$ for each single-qubit rotation gates. 

### Algorithmic physical estimation

From the code distance of the QEC, the Resource Estimator has calculated the number of physical qubits required for one logical qubit and the runtime of one logical depth. 

In the previous step, [Algorithmic logical estimation](#algorithmic-logical-estimation), the Resource Estimator has calculated the number of logical qubits required. Then, the number of physical qubits required to run the algorithm after layout is $ \text{Number of logical qubits} \times \text{Number of physical qubits for one logical qubit}$.

Similarly, the runtime of the algorithm is $\text{Number of algorithmic logical depths} \times \text{Runtime of one logical depth}$.

### T factory physical estimation

In the [Algorithmic logical estimation](#algorithmic-logical-estimation) step, the Resource Estimator calculates the total number of T states needed to run the algorithm. From the target parameters, the Resource Estimator has calculated the number of physical qubits for a single [T factory](xref:microsoft.quantum.concepts.tfactories) and its runtime. 

The goal is to produce all T states within the algorithm runtime with as few T factory copies as possible. The following diagram shows an example of the runtime of the algorithm (red) and the runtime of one T factory (blue). You can see that the runtime of the T factory is shorter than the runtime of the algorithm. In this example, one T factory can distill one T state. Two questions arise:

- How often can the T factory be invoked before the end of the algorithm?
- How many copies of the T factory distillation round are necessary to create the number of T states required during the algorithm's runtime?

<img src="~/media/resource-estimator-tfactory-plot.png" width="400" alt="Diagram showing the runtime of the algorithm (red) versus the runtime of one T factory (blue). Before the end of the algorithm, the T factory can run 8 times. If we need 30 T states, and T factory can run 8 times during runtime, then we need 4 copies of the T factories running in parallel to distill 30 T states.">

Before the end of the algorithm, the T factory can be invoked eight times, which is called a distillation round. For example, let's say that based on the [Algorithmic logical estimation](#algorithmic-logical-estimation) step, you need 30 T states. A single T factory is invoked eight times during runtime of the algorithm and thus, it creates eight T states. Then you need four copies of the T factory distillation round running in parallel to distill the 30 T states needed.

> [!NOTE]
> Note that T factory copies and T factory invocations aren't the same.

You can only do full invocations of a T factory. Therefore, there may be situations in which the accumulated runtime of all T factory invocations is less than the algorithm runtime. Since qubits are reused by different rounds, the number of physical qubits for one T factory is the maximum number of physical qubits used for one round. The runtime of the T factory is the sum of the runtime in all rounds.

> [!NOTE]
> If the physical T gate error rate is lower than the required logical T state error rate, the Resource Estimator cannot perform a good resource estimation. When you submit a resource estimation job, you may encounter that the T factory cannot be found because the required logical T state error rate is either too low or too high.

For more information, see Appendix C of [Assessing requirements to scale to practical quantum advantage](https://arxiv.org/abs/2211.07629).

### Space-time diagrams

Finally, the overall physical resource estimation consists of total number of physical qubits and the runtime. The total number of physical qubits is calculated from the sum of the number of physical qubits required by the T factory copies that produce the T states that are consumed by the algorithm, plus the number of physical qubits required to execute the algorithm. The total runtime of the algorithm is decided based on the number of instructions that need to be performed to run the algorithm. It's the runtime calculated in the [Algorithmic physical estimation](#algorithmic-physical-estimation) step.

You can inspect the distribution of physical qubits used for the algorithm and the T factories using the space-time diagrams. The space diagram shows the proportion of these two. Note that the number of T factory copies contributes to the number of physical qubits for T factories.

```python
result.diagram.space
```

:::image type="content" source="media/resource-estimator-space-diagram.png" alt-text="Pie diagram showing the distribution of total physical qubits between algorithm qubits and T factory qubits. There's a table with the breakdown of number of T factory copies and number of physical qubits per T factory.":::

The time diagram shows the time required to execute the algorithm as it relates to each T factory invocation runtime and the number of T factory invocation (possibly capped if there are too many invocations). The table on the right side lists the most important metrics that are used to compute these numbers, which are described in the [T factory physical estimation](#t-factory-physical-estimation) previous step. Note that the number of T factory invocations is calculated based on the total runtime.

```python
result.diagram.time
```

:::image type="content" source="media/resource-estimator-time-diagram.png" alt-text="Diagram showing the number of T factory invocations during the runtime of the algorithm. There's also a table with the breakdown of the number of T factory copies, number of T factory invocations, T states per invocation, etc.":::

For more information about these diagrams, see the sample in [Resource estimation with Q# and Python](xref:microsoft.quantum.submit-resource-estimation-jobs#space-time-diagrams).

> [!NOTE]
> In the time diagram, each blue arrow represents the total number of copies of the T factory repeatedly invoked.

## Assumptions

The following assumptions are taken into account for the simulation of the resource estimation.

- **Uniform independent physical noise**: The noise on physical qubits and physical qubit operations is the standard circuit noise model. In particular, it is assumed that error events at different space-time locations are independent and that error rates are uniform across the system in time and space.
- **Efficient classical computation**: Classical overhead (compilation, control, feedback, readout, decoding, etc.) doesn't dominate the overall cost of implementing the full quantum algorithm.
- **Extraction circuits for planar quantum ISA (instruction set architecture)**: Stabilizer extraction circuits with similar depth and error correction performance to those for standard surface and Floquet code patches can be constructed to implement all operations of the planar quantum ISA. 
- **Uniform independent logical noise**: The error rate of a logical operation is approximately equal to its space-time volume (the number of tiles multiplied by the number of logical cycles) multiplied by the error rate of a logical qubit in a standard one-tile patch in one logical cycle.
- **Negligible Clifford costs for synthesis**: The space overhead for synthesis and space and time overhead for transport of magic states within magic state factories and to synthesis qubits are all negligible.
- **Smooth magic state consumption rate**: The rate of T state consumption throughout the compiled algorithm is almost constant, or can be made almost constant without significantly impacting the resources.

## Common issues 

 The following scenarios may prevent the resource estimates job to complete. 

- Quantum algorithm must contain at least one measurement.
- Physical T gate error rate is either too low or too high.
- Invalid values for error rate, it must be a number between 0 and 1. 

## Next steps

- [Get the most out of the Resource Estimator](xref:microsoft.quantum.work-with-resource-estimator)
- [Run your first resource estimate](xref:microsoft.quantum.quickstarts.computing.resources-estimator)
- [Use different SDKs and IDEs with Resource Estimator](xref:microsoft.quantum.submit-resource-estimation-jobs)
- [Customize resource estimates to machine characteristics](xref:microsoft.quantum.overview.resources-estimator)
- [Tutorial: Submit a QIR program to the Resource Estimator](xref:microsoft.quantum.tutorial.resource-estimator.qir)
- [Sample: Resource estimation with Q# and VS Code](https://github.com/microsoft/Quantum/tree/main/samples/azure-quantum/resource-estimation/integer-factorization-with-cli)
