# Exercise 4: Circuit Decomposition
Wow! If you managed to solve the first three exercises, congratulations! The fourth problem is supposed to puzzle even the quantum experts among you, so don’t worry if you cannot solve it. If you can, hats off to you!

You may recall from your quantum mechanics course that quantum theory is unitary. Therefore, the evolution of any (closed) system can be described by a unitary. But given an arbitrary unitary, can you actually implement it on your quantum computer?

**"A set of quantum gates is said to be universal if any unitary transformation of the quantum data can be efficiently approximated arbitrarily well as a sequence of gates in the set."** (https://qiskit.org/textbook/ch-algorithms/defining-quantum-circuits.html)

Every gate you run on the IBM Quantum Experience is transpiled into single qubit rotations and CNOT (CX) gates. We know that these constitute a universal gate set, which implies that any unitary can be implemented using only these gates. However, in general it is not easy to find a good decomposition for an arbitrary unitary. Your task is to find such a decomposition.

You are given the following unitary:


```python
from may4_challenge.ex4 import get_unitary
U = get_unitary()
print(U)
print("U has shape", U.shape)
```

#### What circuit would make such a complicated unitary?

Is there some symmetry, or is it random? We just updated Qiskit with the introduction of a quantum circuit library (https://github.com/Qiskit/qiskit-terra/tree/master/qiskit/circuit/library). This library gives users access to a rich set of well-studied circuit families, instances of which can be used as benchmarks (quantum volume), as building blocks in building more complex circuits (adders), or as tools to explore quantum computational advantage over classical computation (instantaneous quantum polynomial complexity circuits).


```python
from qiskit import QuantumCircuit, Aer, QuantumRegister
from may4_challenge.ex4 import check_circuit, submit_circuit
from qiskit.compiler import transpile
from qiskit.extensions.quantum_initializer.isometry import Isometry
from qiskit.quantum_info import Operator
import numpy as np
from scipy.linalg import hadamard
from qiskit.quantum_info.operators.predicates import is_isometry
```

**Using only single qubit rotations and CNOT gates, find a quantum circuit that approximates that unitary $U$ by a unitary $V$ up to an error $\varepsilon = 0.01$, such that $\lVert U - V\rVert_2 \leq \varepsilon$ !** 

Note that the norm we are using here is the spectral norm, $\qquad \lVert A \rVert_2 = \max_{\lVert \psi \rVert_2= 1} \lVert A \psi \rVert$.

This can be seen as the largest scaling factor that the matrix $A$ has on any initial (normalized) state $\psi$. One can show that this norm corresponds to the largest singular value of $A$, i.e., the square root of the largest eigenvalue of the matrix $A^\dagger A$, where $A^{\dagger}$ denotes the conjugate transpose of $A$.

**When you submit a circuit, we remove the global phase of the corresponding unitary $V$ before comparing it with $U$ using the spectral norm. For example, if you submit a circuit that generates $V = \text{e}^{i\theta}U$, we remove the global phase $\text{e}^{i\theta}$ from $V$ before computing the norm, and you will have a successful submission. As a result, you do not have to worry about matching the desired unitary, $U$, up to a global phase.**

As the single-qubit gates have a much higher fidelity than the two-qubit gates, we will look at the number of CNOT-gates, $n_{cx}$, and the number of u3-gates, $n_{u3}$, to determine the cost of your decomposition as 

$$
\qquad \text{cost} = 10 \cdot n_{cx} + n_{u3}
$$

Try to optimize the cost of your decomposition. 

**Note that you will need to ensure that your circuit is composed only of $u3$ and $cx$ gates. The exercise is considered correctly solved if your cost is smaller than 1600.**

---
For useful tips to complete this exercise as well as pointers for communicating with other participants and asking questions, please take a look at the following [repository](https://github.com/qiskit-community/may4_challenge_exercises). You will also find a copy of these exercises, so feel free to edit and experiment with these notebooks.

---


```python
##### build your quantum circuit here
qc = QuantumCircuit(4)
# apply operations to your quantum circuit here
def recursiveKronecker(k, hmat):
    if 2**(k-1) == 1:
        return hmat
    else:
        return np.kron(hmat, recursiveKronecker(k-1, hmat))



h2 = np.matrix([[1,1],[1,-1]])
h2 = h2/(2**0.5)
h = recursiveKronecker(4, h2)

v = h*U*h

qc.h([0,1,2,3])
qc.isometry(v,[0,1,2,3],[])
qc.h([0,1,2,3])
sim = Aer.get_backend('qasm_simulator') 
new_qc = transpile(qc,
                   backend = sim,
                   seed_transpiler=11,
                   optimization_level=2, 
                   basis_gates = ['u3','cx'])

qc_basis = new_qc.decompose()

qc.draw()
```


```python
##### check your quantum circuit by running the next line
check_circuit(qc_basis)
```

    Circuit stats:
    ||U-V||_2 = 1.7512293786350708
    (U is the reference unitary, V is yours, and the global phase has been removed from both of them).
    Cost is 1427
    
    Something is not right with your circuit: the circuit differs from the unitary more than 0.01


You can check whether your circuit is valid before submitting it with `check_circuit(qc)`. Once you have a valid solution, please submit it by running the following cell (delete the `#` before `submit_circuit`). You can re-submit at any time.



```python
# Send the circuit as the final answer, can re-submit at any time
submit_circuit(qc_basis) 
```



<div style="border: 2px solid black; padding: 2rem;">
    <p>
        Success 🎉! Your circuit has been submitted. Return to the
        <a href="https://quantum-computing.ibm.com/challenges/4anniversary/?exercise=4" target="_blank">
            IBM Quantum Challenge page
        </a>
        and check your score and ranking.
    </p>
    <p>
        Remember that you can submit a circuit as many times as you
        want.
    </p>
</div>

