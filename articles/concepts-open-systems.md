---
author: SoniaLopezBravo
description: Learn how quantum operations affect the states of open systems.
ms.author: sonialopez
ms.date: 06/09/2023
ms.service: azure-quantum
ms.subservice: core
ms.topic: conceptual
no-loc: ['Q#', '$$v', '$$', "$$", '$', "$", $, $$, '\cdots', 'bmatrix', '\ddots', '\equiv', '\sum', '\begin', '\end', '\sqrt', '\otimes', '{', '}', '\text', '\phi', '\kappa', '\psi', '\alpha', '\beta', '\gamma', '\delta', '\omega', '\bra', '\ket', '\boldone', '\\\\', '\\', '=', '\frac', '\text', '\mapsto', '\dagger', '\to', '\begin{cases}', '\end{cases}', '\operatorname', '\braket', '\id', '\expect', '\defeq', '\variance', '\dd', '&', '\begin{align}', '\end{align}', '\Lambda', '\lambda', '\Omega', '\mathrm', '\left', '\right', '\qquad', '\times', '\big', '\langle', '\rangle', '\bigg', '\Big', '\Bigg' ,'|', '\mathbb', '\vec', '\in', '\texttt', '\ne', '<', '>', '\leq', '\geq', '~~', '~', '\begin{bmatrix}', '\end{bmatrix}', '\_', '\rho', '\quad', '\sim', '\left\','\right\', '\%', '%', Quantum Development Kit, target, targets]
title: Open systems
uid: microsoft.quantum.concepts.opensystems
---

# Open quantum systems

Quantum systems that are isolated from their environments such that no other system interacts with the qubits are called *closed quantum systems*. By contrast, a device that is subject to some amount of interaction, or *noise*, from its environment is an *open quantum system*. In general, these interactions between the system and the environment significantly change the dynamics of the system and result in quantum dissipation, such that the information contained in the system is lost to its environment.

The Quantum Development Kit provides a [noise simulator](xref:microsoft.quantum.machines.overview.noise-simulator) for simulation of open quantum systems. This feature allows for simulating the behavior of Q# programs under the influence of noise, and also for using the *stabilizer representation* (also known as CHP simulation) of quantum algorithms, that is, algorithms consisting solely of [CNOT](xref:Microsoft.Quantum.Intrinsic.CNOT), [Hadamard](xref:Microsoft.Quantum.Intrinsic.H), and phase gates.

This article explains some of the basic concepts of open quantum systems, and how quantum operations can affect the states of open systems.

## Invoking the noise simulators from Python

You need to create a Python host program that invokes the quantum program and can further process returned results. For additional details, see [Ways to run a Q# program](xref:microsoft.quantum.user-guide-qdk.overview.host-programs).

1. You start by importing the [QuTiP library](https://qutip.org/), a popular Python library for manipulating states and processes of closed and open quantum systems.

```python
import qutip as qt
import numpy as np
import matplotlib.pyplot as plt

```

1. You can enable the use of the noise simulators by using the `qsharp` package:

```python
import qsharp
```

## Revisiting quantum states

Before discussing the representation of open quantum systems, it's helpful to quickly revisit representations of closed quantum systems. In particular, the state of an $n$-qubit register can be represented as a vector of $2^n$ complex numbers. For example, the state of a single qubit can be written as a vector of the form

$$
\begin{aligned}
    \ket{\psi} = \alpha \ket{0} + \beta \ket{1} = \left( \begin{matrix}
        \alpha \\ \beta
    \end{matrix} \right)
\end{aligned}
$$

where $\alpha$ and $\beta$ are complex numbers such that $|\alpha|^2 + |\beta|^2 = 1$.

In Q#, you can ask the default simulator to dump the state that it uses to simulate quantum programs, getting back a description of that state as a vector of this form. For more information, see [Dump functions](xref:microsoft.quantum.user-guide-qdk.overview.testingdebugging#dump-functions).

1. In the same folder as the Python host program, create the following Q# program in a file called `OpenSystemsConcepts.qs`:

```qsharp
namespace OpenSystemsConcepts {
    open Microsoft.Quantum.Intrinsic;
    open Microsoft.Quantum.Canon;
    open Microsoft.Quantum.Diagnostics;
    open Microsoft.Quantum.Measurement;
    open Microsoft.Quantum.Random;

    operation DumpPlus() : Unit {
        use q = Qubit();
        within {
            H(q);
        } apply {
            DumpMachine();
        }
    }
}
```

> [!NOTE]
> The `@EntryPoint()` attribute used for Q# applications cannot be used with host programs. An error will be raised if it is present in the Q# file being called by a host.

1. Add the following code to your host program to import the Q# operation `DumpPlus`:

```python
from OpenSystemsConcepts import DumpPlus

print(DumpPlus.simulate())
```

Calling the <xref:Microsoft.Quantum.Diagnostics.DumpMachine> generates the following output:

```output
# wave function for qubits with ids (least to most significant): 0
∣0❭:     0.707107 +  0.000000 i  ==     ***********          [ 0.500000 ]     --- [  0.00000 rad ]
∣1❭:     0.707107 +  0.000000 i  ==     ***********          [ 0.500000 ]     --- [  0.00000 rad ]
```

> [!NOTE]
> Notice that the output may appear differently if you run the example in a Python notebook, as Jupyter notebook, instead of at the command line, as that interface understands how to forward HTML-based diagnostics from the IQ# kernel to the IPython kernel.

The first row provides a comment with the IDs of the corresponding qubits in their significant order.
The rest of the rows describe the probability amplitude of measuring the basis state vector $\ket{n}$ in both Cartesian and polar formats. In detail for the first row:

* **`∣0❭:`** this row corresponds to the `0` computational basis state.
* **`0.707107 +  0.000000 i`**: the probability amplitude in Cartesian format.
* **` == `**: the `equal` sign separates both equivalent representations.
* **`**********  `**: A graphical representation of the magnitude, the number of `*` is proportionate to the probability of measuring this state vector.
* **`[ 0.500000 ]`**: the numeric value of the magnitude.
* **`    ---`**: A graphical representation of the amplitude's phase (see the following output).
* **`[ 0.0000 rad ]`**: the numeric value of the phase (in radians).

The above diagnostic tells you that at the point when `DumpMachine` is called, the state of the qubit is given by the vector $\ket{+} \mathrel{:=} (\ket{0} + \ket{1}) / \sqrt{2} \approx 0.7071 \ket{0} + 0.7071 \ket{1}$.

You can also write this state in QuTiP notation, using `qt.basis(2, i)` to represent $\ket{i}$ on a single qubit:

```python
ket0 = qt.basis(2, 0)
ket1 = qt.basis(2, 1)
ket_plus = (1 / np.sqrt(2)) * (ket0 + ket1)
print(ket_plus)
```

```output
Quantum object: dims = [[2], [1]], shape = (2, 1), type = ket
Qobj data =
[[0.707]
 [0.7.7]]
```

1. When measuring a qubit in the $\ket{+}$ state in the $Z$-basis, you get 0 and 1 with equal probability. This can be used as a quantum random bit generator.

```qsharp
operation SampleRandomBit() : Result {
    use q = Qubit();
    H(q);
    return MResetZ(q);
}
```

```python
from OpenSystemsConcepts import SampleRandomBit
print(sum(SampleRandomBit.simulate() for _ in range(100)))
```

```output
54
```

1. Though the $\ket{+}$ state is not inherently random, you can deterministically return to the $\ket{0}$ state by applying another Hadamard operation:

```qsharp
operation ApplyHTwiceAndMeasure() : Result {
    use q = Qubit();
    H(q);
    H(q);
    return MResetZ(q);
}
```

```python
from OpenSystemsConcepts import ApplyHTwiceAndMeasure
print(sum(ApplyHTwiceAndMeasure.simulate() for _ in range(100)))
```

```output
0
```

## Preparing random states

As opposed to preparing $\ket{+}$ and then measuring it, you could also consider flipping a coin classically, and using the outcome to prepare either the $\ket{0}$ or $\ket{1}$ state. Thus, if the outcome is "heads", the qubit is prepared in the $\ket{0}$ state, and if the outcome is "tails", the qubit is prepared in the $\ket{1}$ state.

1. Add the following code to your Q# program.

```qsharp
operation PrepareAndMeasureRandomState() : Result {
    use q = Qubit();
    if DrawRandomBool(0.5) {
        X(q);
    }
    return MResetZ(q);
}
```

The `PrepareAndMeasureRandomState` operation applies the [`X`](xref:Microsoft.Quantum.Intrinsic.X) to a qubit with 50% of probability, as in a classical coin flip. Doing so, the results is the 50/50 outcomes for the $\ket{0}$ state and the $\ket{1}$ state :

```python
from OpenSystemsConcepts import PrepareAndMeasureRandomState
print(sum(PrepareAndMeasureRandomState.simulate() for _ in range(100)))
```

```output
45
```

1. However, now if you apply `H` again, the result of the operation doesn't get back to a deterministic outcome:

```qsharp
operation ApplyHToRandomStateAndMeasure() : Result {
    use q = Qubit();
    if DrawRandomBool(0.5) {
        X(q);
    }
    H(q); // Doesn't get us back to 0 state
    return MResetZ(q);
}
```

```python
from OpenSystemsConcepts import PrepareAndMeasureRandomState
print(sum(ApplyHToRandomStateAndMeasure.simulate() for _ in range(100)))
```

```output
42
```

### Density operator

As it turns out, there is no single vector that represents the state prepared by the `ApplyHToRandomStateAndMeasure` operation unless you know the outcome of the random coin flip `(DrawRandomBool(0.5))`. If you don't know the outcome of the coin flip, the quantum state is given by the following *ensemble* of state vectors,

$$
\rho = \lbrace \ket{0} \text{ with probability } \frac{1}{2}, \ket{1}  \text{ with probability } \frac{1}{2} \rbrace .
$$

Given a quantum state $\ket{\psi}$ the probability of the outcome $\ket{\phi}$ after a measurement is given by the Born's rule,

$$
\begin{align}
    \Pr(\phi | \psi) & = \left|\left\langle \phi | \psi \right\rangle\right|^2 \\\\
                     & = \left\langle \phi | \psi \right\rangle \left\langle \psi | \phi \right\rangle.
\end{align}
$$

The trick here is to average over the different state vectors that could be prepared by the `ApplyHToRandomStateAndMeasure` operation:

$$
\begin{align}
    \text{Pr}(\phi | \rho) & = \mathbb{E}\_{\psi \sim \rho} \left( \text{Pr}(\phi | \psi) \right) \\\\
                    & = \mathbb{E}\_{\psi \sim \rho} \left( \left\langle \phi | \psi \right\rangle \left\langle \psi | \phi \right\rangle \right) \\\\
                    & = \sum_i \text{Pr}(\psi_i) \left\langle \phi | \psi_i \right\rangle \left\langle \psi_i | \phi \right\rangle \\\\
                    & = \left\langle \phi \Bigg| \left( \sum_i \text{Pr}(\psi_i) \ket{\psi_i} \bra{\psi_i} \right) \Bigg| \phi \right\rangle.
\end{align}
$$

Factoring out $\bra{\phi}$ and $\ket{\phi}$ in the last step gives us a neat new way of writing out ensembles of state vectors as matrices called [**density operators**](xref:microsoft.quantum.concepts.dirac#density-operators). For example, the ensemble $\rho$ can also be written out as the following density operator,

$$
\begin{align}
    \rho & = \sum_i \Pr(\psi_i) \ket{\psi_i} \bra{\psi_i} \\\\
         & = \frac{1}{2} \ket{0} \bra{0} + \frac{1}{2} \ket{1} \bra{1} \\\\
         & = \frac{1}{2} \left( \begin{matrix}
             1 & 0 \\\\ 0 & 1
         \end{matrix} \right).
\end{align}
$$

Using density operators, you can write both ensembles of states, as well as the projectors on to those state vectors. For example, the $\ket{+}$ state can be written as the density operator

$$
\begin{align}
    \ket{+}\bra{+} =
    \frac{1}{2} \left( \begin{matrix}
        1 & 1 \\\\ 1 & 1
    \end{matrix} \right).
\end{align}
$$

That is, even though both `SampleRandomBit` and `PrepareAndMeasureRandomState` both prepare density operators with the same diagonal elements (and thus have the same measurement probabilities in the $Z$-basis), the two density operators have different off-diagonal elements.

We say that density operators in general represent *mixed states*, and that states that can be written as $\ket{\psi}\bra{\psi}$ for some state vector $\ket{\psi}$ (fo example, $\ket{+}\bra{+}$) are *pure states*. For more information about the differences between mixed states and pure states, see [density operators](xref:microsoft.quantum.concepts.dirac#density-operators).

## Representing quantum processes

Considering the [Hadamard](xref:Microsoft.Quantum.Intrinsic.H) operation, you can simulate its action on a pure state $\ket{\psi}$ as $H \ket{\psi}$. By direct analogy, you can simulate what $H$ does to a density operator by multiplying on the left and right both:

$$
\begin{align}
    \rho \longmapsto H\rho H^{\dagger} = H \rho H.
\end{align}
$$

More generally, though, you can also represent processes in which one of several unitary operations is applied at random. For example, suppose $H$ operation works 95% of the time, but the other 5% of the time does nothing. You can simply add the density operators weighted by the probability of each case:

$$
\begin{align}
    \rho \longmapsto 0.95 H \rho H + 0.05 \rho.
\end{align}
$$

One way to model this is by thinking of the $H$ operation not as being represented by a unitary matrix, but by a function from density operators to density operators.

$$
\begin{align}
    \Lambda_H(\rho) = H\rho H.
\end{align}
$$

Since density functions represent an average over different preparations, and since averages are linear, such functions must in general must also be linear.

A linear function that maps a density operator to another density operator is called a *quantum process*.

> [!TIP]
> In addition to linearity, quantum processes that can be realized in practice fulfill the condition of being completely positive and trace preserving (CPTP) maps. A linear map that is both completely positive and trace preserving is known as a *quantum channel*.

There are different ways to represent quantum processes, but the most common method when working with noise models for open quantum systems is to use *superoperators*. Just as operators are linear functions from vectors to vectors, superoperators are linear functions from operators to operators, and can be written using matrices.

For example, you can use `qt.to_super` to convert a few unitary matrices into superoperators. Notice that the 2 × 2 identity operator $𝟙$ we use to simulate the [I](xref:Microsoft.Quantum.Intrinsic.I) operation is a 4 × 4 identity matrix:

```python
# Convert the 𝟙 matrix used to simulate Microsoft.Quantum.Intrinsic.I (single qubit no-op) into a superoperator.
print(qt.to_super(qt.qeye(2)))
```

```output
Quantum object: dims = [[[2], [2]], [[2], [2]]], shape = (4, 4), type = super, isherm = True
Qobj data =
[[1.0  0.0  0.0  0.0]
 [0.0  1.0  0.0  0.0]
 [0.0  0.0  1.0  0.0]
 [0.0  0.0  0.0  1.0]]
```

As expected, the superoperator resulting from the identity operator maps all operators to themselves.

On the other hand, for the $X$ operation the result is different:

```python
# Convert the 𝑋 matrix used to simulate Microsoft.Quantum.Intrinsic.X into a superoperator.
print(qt.to_super(qt.sigmax()))
```

```output
Quantum object: dims = [[[2], [2]], [[2], [2]]], shape = (4, 4), type = super, isherm = True
Qobj data =
[[0.0  0.0  0.0  1.0]
 [0.0  0.0  1.0  0.0]
 [0.0  1.0  0.0  0.0]
 [1.0  0.0  0.0  0.0]]
```

Notice that each column is a stack of the elements in an operator output by the function $\Lambda_X(\rho) = X \rho X^{\dagger} = X \rho X$.

Since $\Lambda(\ket{0}\bra{0}) = X\ket{0} \bra{0}X = \ket{1}\bra{1}$, the first column is a stack of the elements of $\ket{1}\bra{1} = \left(\begin{matrix} 0 & 0 \\\\ 0 & 1 \end{matrix}\right)$. Similarly, the second column is a stack of the elements of $\Lambda_X(\ket{0}\bra{1}) = \ket{1}\bra{0}$:

```python
print(np.array((ket1 * ket0.dag()).data.todense().flat))
```

```output
[0.+0.j, 0.+0.j, 1.+0.j, 0.+0.j]
```

The same pattern can be used  in converting other unitary operators such as $H$ and $Z$:

```python
H_super = qt.to_super(qt.qip.operations.hadamard_transform())
print(H_super)
```

```output
Quantum object: dims = [[[2], [2]], [[2], [2]]], shape = (4, 4), type = super, isherm = True
Qobj data =
[[0.5 0.5 0.5 0.5]
 [0.5 -0.5 0.5 -0.5]
 [0.5 0.5 -0.5 -0.5]
 [0.5 -0.5 -0.5 0.5]]
```

```python
Z_super = qt.to_super(qt.sigmaz())
print(Z_super)
```

```output
Quantum object: dims = [[[2], [2]], [[2], [2]]], shape = (4, 4), type = super, isherm = True
Qobj data =
[[1.0 0.0 0.0 0.0]
 [0.0 -1.0 0.0 0.0]
 [0.0 0.0 -1.0 0.0]
 [0.0 0.0 0.0 1.0]]
```

Looking at the $Z$ superoperator example, notice there is not a $-1$ sign in the lower-right hand corner. This is because density operators and superoperators do not have the same global phase ambiguity that state vectors and unitary operators do.

In particular, consider the $Z$ operation acting on a qubit in the $\ket{1}$ state. When simulating this with state vectors, the result is $Z \ket{1} = -\ket{1}$, where the $-$ sign in front of $\ket{1}$ is, in this case, an insignificant global phase. On the other hand, when using open systems notation, the same operation results in $\Lambda_Z(\ket{1}\bra{1}) = Z\ket{1} \bra{1}Z^{\dagger} = Z\ket{1} \bra{1}Z = (-\ket{1})(-\bra{1}) = \ket{1}\bra{1}$, such that the global phases on the "ket" and "bra" parts of $\ket{1}\bra{1}$ cancel each other out.

More generally, suppose that $U \ket{\phi} = e^{i\phi} \ket{\phi}$ for some unitary operator $U$, some phase $\phi$, and some state vector $\ket{\phi}$. Then, since $\bra{\phi} U^\dagger = (U \ket{\phi})^\dagger = (e^{i \phi} \ket{\phi})^\dagger = \bra{\phi} e^{-i\phi}$, this results in the same cancellation:

$$
\begin{align}
    \Lambda_U (\ket{\phi} \bra{\phi}) & = U \ket{\phi} \bra{\phi} U^{\dagger} \\\\
                                      & = e^{i\phi} \ket{\phi} \bra{\phi} e^{-i\phi} \\\\
                                      & = \ket{\phi} \bra{\phi}
\end{align}
$$

On the other hand, relative phases are still represented in open systems notation, which can be confirmed by looking at the matrices for $\ket{+}\bra{+}$ and $\ket{-}\bra{-}$:

```python
print(ket_plus * ket_plus.dag())
```

```output
Quantum object: dims = [[2], [2]], shape = (2, 2), type = oper, isherm = True
Qobj data =
[[0.5 0.5]
 [0.5 0.5]]
```

```python
ket_minus = (ket0 - ket1) / np.sqrt(2)
print(ket_minus * ket_minus.dag())
```

```output
Quantum object: dims = [[2], [2]], shape = (2, 2), type = oper, isherm = True
Qobj data =
[[0.5 -0.5]
 [-0.5 0.5]]
```

That is, the off-diagonal elements of density operators describe the relative phases between each computational basis state.

## Noisy quantum processes

Using superoperators not only allows you to represent familiar unitary operations, but also those functions from density operators to density operators that arise in describing noise. For example, the quantum process $\Lambda(\rho) = 0.95 H\rho H + 0.05 \rho$ can be written as a superoperator by summing the superoperators for $H$ and $I$ weighted by the probability for each case:

```python
lambda_noisy_h = 0.95 * qt.to_super(qt.qip.operations.hadamard_transform()) + 0.05 * qt.to_super(qt.qeye(2))
print(lambda_noisy_h)
```

```output
Quantum object: dims = [[[2], [2]], [[2], [2]]], shape = (4, 4), type = super, isherm = True
Qobj data =
[[0.525 0.475 0.475 0.475]
 [0.475 -0.425 0.475 -0.475]
 [0.475 0.475 -0.425 -0.475]
 [0.475 -0.475 -0.475 0.525]]
```

Unlike the unitary matrix $H$ that describes the ideal action of the Hadamard operation, the superoperator $\Lambda_{\text{noisy}H}$ simulates the action of the Hadamard operation when it has a 5% probability of doing nothing.

### Noise simulation in Q\#

Similar to the `.simulate()` method, Q# operations imported into Python expose a `.simulate_noise()` method that can be used to run Q# programs against the noise simulators.

By default, `.simulate_noise()` method assumes an *ideal error* model, that is, no noise. To configure a particular error model, you can use the `qsharp.get_noise_model` and `qsharp.set_noise_model` functions to get and set the current noise model for the preview simulators. Each error model is represented as a dictionary from intrinsic operation names to objects representing the errors in those intrinsic operations. For more information, see [Configuring open systems in noise models](xref:microsoft.quantum.machines.overview.noise-simulator#configuring-open-systems-noise-models).

You can simulate the superoperator $\Lambda_{text{noisy}H}$ previously defined as a noise model and run the `DumpPlus` operation.

1. First, run the code with an ideal noise model so you can compare the outcome.

```python
qsharp.config['simulators.noisy.nQubits'] = 1
qsharp.set_noise_model_by_name('ideal')

# Run with an ideal noise model first.
print(DumpPlus.simulate_noise())
```

```output
'text/plain': 'Mixed state on 1 qubits: [ [0.5000000000000001 + 0 i, 0.5000000000000001 + 0 i] [0.5000000000000001 + 0 i, 0.5000000000000001 + 0 i] ]'
```

1. To configure the noise model, set `lambda_noisy_h` as the noise model to follow, and then run the `DumpPlus` operation to see the effects of the noisy Hadamard operation on the quantum state.

```python
noise_model = qsharp.set_noise_model_by_name('ideal', h=lambda_noisy_h)

# Run again, using the new noisy superoperator for the `H` operation.
print(DumpPlus.simulate_noise())
```

```output
'text/plain': 'Mixed state on 1 qubits: [ [0.5249999999999999 + 0 i, 0.47500000000000026 + -7.536865798903469E-33 i] [0.5249999999999999 + 0 i, 0.47500000000000026 + 7.536865798903469E-33 i & 0.4750000000000007 + 0 i] ]'
```

Comparing the two quantum states, you can see that when adding a probability to the Hadamard operation of failing into the noise model, the noise simulator uses the density operator representation together with the defined noise model to simulate the effect of that noise on the state of the qubits.

There are many other kinds of noise as well. For example, a very well known noise model is the *completely depolarizing noise*, which results in the equal probability of applying the I, X, Y, or Z operations to a single qubit:

```python
[I, X, Y, Z] = map(qt.to_super, [qt.qeye(2), qt.sigmax(), qt.sigmay(), qt.sigmaz()])
completely_depolarizing_process = 0.25 * (I + X + Y + Z)
print(completely_depolarizing_process)
```

```output
Quantum object: dims = [[[2], [2]], [[2], [2]]], shape = (4, 4), type = super, isherm = True
Qobj data =
[[0.5 0.0 0.0 0.5]
 [0.0 0.0 0.0 0.0]
 [0.0 0.0 0.0 0.0]
 [0.5 0.0 0.0 0.5]]
```

From the output, you can see that the completely depolarizing noise maps all input density operators to the same output, namely $𝟙 / 2$. That is, completely depolarizing noise replaces whatever the previous state was with the maximally mixed state.

The depolarizing noise can also be of a finite strength (that is, *not completely* depolarizing) by taking a linear combination of the identity and completely depolarizing processes.

```python
def depolarizing_noise(strength=0.05):
    return strength * completely_depolarizing_process + (1 - strength) * qt.to_super(I)
    
print(depolarizing_noise(0.05))
```

```output
Quantum object: dims = [[[2], [2]], [[2], [2]]], shape = (4, 4), type = super, isherm = True
Qobj data =
[[0.975 0.0 0.0 0.025]
 [0.0 0.950 0.0 0.0]
 [0.0 0.0 0.950 0.0]
 [0.025 0.0 0.0 0.975]]
```

Some noise models can be represented as a mixture of unitary operators, but there are also many kinds of noise that cannot be represented this way. For instance, by analogy to completely depolarizing noise, a process that has some probability of resetting its input state can be represented as a mixture of a superoperator that always resets and the identity process.

To see this, consider the process $\Lambda_{\text{Reset}}(\rho) = Tr(\rho) \ket{0}\bra{0}$. Since this process does the same thing to each density operator as input, the first and fourth columns should be the same, namely $\left(\begin{array} 1 & 0 & 0 & 0 \end{array} \right)^{\text{T}}$.

On the other hand, the second and third columns represent the output of $\Lambda_{\text{Reset}}$ acting on $\ket{0}\bra{1}$ and $\ket{1}\bra{0}$ respectively. Neither of these is a valid density operator on their own — indeed, $Tr(\ket{0}\bra{1}) = Tr(\ket{1}\bra{0})$ such that the trace factor zeros out both of these columns.

```python
lambda_reset = qt.Qobj([
    [1, 0, 0, 1],
    [0, 0, 0, 0],
    [0, 0, 0, 0],
    [0, 0, 0, 0],
], dims=completely_depolarizing_process.dims)
```

You can use `lambda_reset` to represent the *amplitude damping noise*, in which there's a finite chance of resetting each qubit:

```python
def amplitude_damping_noise(strength=0.05):
    return strength * lambda_reset + (1 - strength) * qt.to_super(I)
print(amplitude_damping_noise(0.05))
```

```output
Quantum object: dims = [[[2], [2]], [[2], [2]]], shape = (4, 4), type = super, isherm = False
Qobj data =
[[1.0 0.0 0.0 0.050]
 [0.0 0.950 0.0 0.0]
 [0.0 0.0 0.950 0.0]
 [0.0 0.0 0.0 0.950]]
```

## Example: Decaying Ramsey signal

In this example, you will study the evolution of a qubit when you repeatedly apply the π/4 phase gate or [`S`](xref:Microsoft.Quantum.Intrinsic.S) to it and then measure. This simple example works as a good toy model for Ramsey interferometry, which is a technique for measuring particle transition frequencies with atomic precision.

In the absence of noise, a Ramsey signal keeps oscillating back and forth with each iteration of `S`. Add the following code to your Q# quantum program:

```qsharp
operation ApplySRepeatedlyAndMeasure(nRepetions : Int) : Result {
    use q = Qubit();
    within {
        H(q);
    } apply {
        for _ in 1..nRepetions {
            S(q);
        }
    }
    return MResetZ(q);
}
```

The operation `ApplySRepeatedlyAndMeasure` sets a qubit in superposition, applies the π/4 phase operation `nRepetions` times, and then returns the measurement in $Z$-axis.
Add the following code to your host program to simulate the `ApplySRepeatedlyAndMeasure` operation in the absence of noise.

```python
from OpenSystemsConcepts import ApplySRepeatedlyAndMeasure

qsharp.set_noise_model_by_name('ideal')

ns = np.arange(1, 101)
signal_wo_noise = [
    sum(ApplySRepeatedlyAndMeasure.simulate_noise(nRepetions=n) for _ in range(100))
    for n in ns
]

print(plt.plot(ns, signal_wo_noise))
```

:::image type="content" source="media/ramsey-signal-ideal-model.png" alt-text="Plot of the Ramsey signal in the absense of noise.":::

On the other hand, if the $S$ operation has some finite amplitude damping noise, then the signal will eventually decay:

```python
S = qt.Qobj([
    [1, 0],
    [0, 1j]
])

qsharp.set_noise_model_by_name(
    'ideal',
    h=(
        depolarizing_noise(0.025) *
        qt.to_super(qt.qip.operations.hadamard_transform())
    ),
    s=(
        amplitude_damping_noise(0.025) *
        qt.to_super(S)
    )
)

# Note that this can take a few moments to run.
signal = [
    sum(ApplySRepeatedlyAndMeasure.simulate_noise(nRepetions=n) for _ in range(100))
    for n in ns
]

print(plt.plot(ns, signal_wo_noise, label='Ideal Signal'))
print(plt.plot(ns, signal, label='With Amplitude Damping Noise'))
plt.legend()
```

:::image type="content" source="media/ramsey-signal-noise-model.png" alt-text="Plot of the ideal Ramsey signal and in the presence of noise.":::

## Representing noise in continuous-time rotations

So far, you have mostly considered quantum operations such as [`H`](xref:Microsoft.Quantum.Intrinsic.H) or [`X`](xref:Microsoft.Quantum.Intrinsic.X).
In particular, these operations are both of type `Qubit => Unit is Adj + Ctl`, indicating that they do not have any additional parameters beyond the target qubit that they act upon.

By contrast, the [`Rz` operation](xref:Microsoft.Quantum.Intrinsic.Rz) has type `(Double, Qubit) => Unit is Adj + Ctl`, representing the angle of the rotation to be performed as well as the target.
In general, the noise incurred by `Rz(0.123)` need not be the same as the noise incurred by `Rz(0.456)`, such that you also require a way to represent *continuous-time* noise as well as the discrete case considered above.

In closed quantum systems (that is, quantum systems that evolve without noise), continuous-time evolution is given by a Hamiltonian operator $H$ that represents the time derivative of a quantum state as per the Schrödinger equation:
$$
\begin{equation*}
    \frac{\partial}{\partial t} \ket{\psi(t)} = -iH \ket{\psi(t)}.
\end{equation*}
$$
This is solved by taking the matrix exponential of $H$,
$$
\begin{aligned}
    \ket{\psi(t)} & = U(t) \ket{\psi(0)} \text{ where} \\
    U(t) & = \exp(-itH).
\end{aligned}
$$

By analogy, you can represent the derivative of a density operator in the absence of noise by rearranging the Schrödinger equation:
$$
\begin{equation*}
    \frac{\partial}{\partial t} \rho(t) = -i[H, \rho],
\end{equation*}
$$
where $[A, B] \mathrel{:=} AB - BA$ is the *commutator* of $A$ with $B$.
The effect of noise can be included in the above by adding one or more *jump operators* $\{L_i\}$ representing the various kinds of noisy processes competing with coherent evolution:
$$
\begin{equation*}
    \frac{\partial}{\partial t} \rho(t) = -i[H, \rho] + \sum_i \left(
        L_i \rho L_i^\dagger - \frac12 \{L_i^\dagger L, \rho\}
    \right)
\end{equation*}
$$
where $\{A, B\} \mathrel{:=} AB + BA$ is the *anticommutator* of $A$ with $B$.

To represent the above derivative as a single matrix as you did with $U(t)$ in the closed systems case, you need a way to turn left- and right-multiplication of a matrix into left-multiplication of a vector. You can use what's known as the *column-stacking convention* to turn $\rho$ into a vector $|\rho⟫$; for example,
$$
    \Bigg|\left(\begin{matrix} a & b \\ c & d \end{matrix}\right)\Bigg\rangle\!\!\!\!\Bigg\rangle =
    \left(\begin{matrix}a \\ c \\ b \\ d\end{matrix}\right).
$$
You can then use the identity that $|ABC⟫ = C^\mathrm{T} \otimes A |B⟫$ to turn the above open systems derivative into a single matrix
$$
    G = -i(𝟙 \otimes H - H^\mathrm{T} \otimes I) + \sum_i \left(
        L_i^* \otimes L_i
        - \frac12 (L_i^\dagger L_i)^\mathrm{T} \otimes 𝟙
        - \frac12 𝟙 \otimes (L_i^\dagger L_i)^\mathrm{T}
    \right).
$$
This then gives us that
$$
    \rho(t) = \exp(tG) \rho(0).
$$

You can assign such generators to a noise model using the `to_generator` function provided by the `qsharp` package.
For example, the ideal action of the `Rz` operation is given by the Hamiltonian $H = -\frac12 Z$; we can represent a misalignment of this Hamiltonian together with $T_1$ and $T_2$ processes by passing each to `to_generator`:

```python
qsharp.set_noise_model_by_name(
    'ideal',
    rz=to_generator(
        -0.48 * qt.sigmaz(),
        t1_dissipation(100.0),
        t2_dissipation(25.0)
    )
)
```

You can also add in fixed quantum processes that apply before and after continuous-time evolution using the `pre` and `post` keyword arguments:

```python
qsharp.set_noise_model_by_name(
    'ideal',
    rz=to_generator(
        -0.48 * qt.sigmaz(),
        t1_dissipation(100.0),
        t2_dissipation(25.0),
        post=depolarizing_process(0.95)
    )
)
```

## Example: Revisiting Ramsey evolution

The [`S` operation](xref:Microsoft.Quantum.Intrinsic.S) used in our Ramsey example above is a special case of the [`Rz` operation](xref:Microsoft.Quantum.Intrinsic.Rz); specifically, `S` applies a rotation about the $Z$-axis by an angle of $\pi / 4$. By representing continuous-time noise, we can consider the more general case as well.

Let's go on and define a new Q# operation for the continuous-time Ramsey example:

```qsharp
open Microsoft.Quantum.Measurement;

operation MeasureRamsey(nShots : Int, t : Double) : Int {
    mutable nUp = 0;
    for idxShot in 1..nShots {
        use q = Qubit();
        within {
            H(q);
        } apply {
            Rz(t, q);
        }
        if MResetZ(q) == One {
            set nUp += 1;
        }
    }
    return nUp;
}
```

You can then assign a continuous-time noise process to the `Rz` operation:

```python
import qsharp
import qutip as qt

qsharp.set_noise_model_by_name(
    'ideal',
    rz=to_generator(
        -0.48 * qt.sigmaz(),
        t1_dissipation(100.0),
        t2_dissipation(25.0),
        post=depolarizing_process(0.95)
    )
)
```


:::image type="content" source="media/ramsey-signal-noise-model.png" alt-text="Plot of the ideal Ramsey signal and in the presence of noise.":::


Finally, you can simulate at a variety of different evolution times and plot the effect of the above noise:

```python
import numpy as np

ts = np.linspace(0, 10, 101)
n_ups_noiseless = np.array([MeasureRamsey.simulate(nShots=500, t=t) for t in ts])
n_ups = np.array([MeasureRamsey.simulate_noise(nShots=500, t=t) for t in ts])
```

Here, note that the elements of `ts` are not even multiples of $\pi / 4$, such that you can't reproduce the above results without assigning a continuous-time generator to describe the noise incurred by calling `Rz`.

## Next steps

- [Grover's algorithm](xref:microsoft.quantum.concepts.grovers)
- [Quantum Intermediate Representation](xref:microsoft.quantum.concepts.qir)
- [Vectors and matrices](xref:microsoft.quantum.concepts.vectors)
- [T gates and T factories](xref:microsoft.quantum.concepts.tfactories)
* [QDK noise simulators](xref:microsoft.quantum.machines.overview.noise-simulator)
