.. _start:

Getting Started
==================================

The :math:`\mathrm{t|ket}\rangle` compiler is a powerful tool for optimising and manipulating platform-agnostic quantum circuits, focussed on enabling superior performance on NISQ (Noisy Intermediate-Scale Quantum) devices. The pytket package provides an API for interacting with :math:`\mathrm{t|ket}\rangle` and transpiling to and from other popular quantum circuit specifications.

We recommend using a Python 3.6+ environment when possible. Install pytket from PyPI using:

::

    pip install pytket

This will install the :math:`\mathrm{t|ket}\rangle` compiler binaries as well as the pytket package. For those using an older version of pytket, keep up to date by installing with the ``--upgrade`` flag for additional features and bug fixes.

There are separate packages for managing the interoperability between pytket and other quantum software packages which can also be installed via PyPI:

* ``pytket_cirq``
* ``pytket_projectq``
* ``pytket_pyquil``
* ``pytket_pyzx``
* ``pytket_qiskit``

The :py:class:`Circuit` Class
-----------------------------
:py:class:`Circuit` objects provide an abstraction of quantum circuits. They consist of a set of qubits/quantum wires and a collection of operations applied to them in a given order. These wires have open inputs and outputs, rather than assuming any fixed input state.

We can start by building a circuit with some blank wires and add some instructions.

::

    from pytket import Circuit
    c = Circuit(2,2)
    c.H(0)
    c.Rz(0, 0.25)
    c.CX(1,0)

Breaking this down: ``c = Circuit(2,2)`` creates a new circuit with two qubits and two bits. Each qubit is identified by its numerical index, ranging from 0 to (no. of qubits)-1. ``c.H(0)`` applies a Hadamard gate to qubit 0. Instructions are always added to the end of the circuit. Rotational gates can be parameterised by the angle of rotation, specified in half-turns - for example ``c.Rz(0, 0.25)`` adds a Z rotation by PI*0.25 radians. Multi-qubit operations will take an ordered sequence of qubit indices, such as ``c.CX(1,0)`` applying a CX gate with the control on qubit 1 and the target on qubit 0.

We can add measurements onto qubits to observe their state and store the result in a given bit.

::

    c.Measure(0,0)
    c.Measure(1,1)

We assume all measurements are made in the computational (Z) basis - others can be performed by adding the appropriate change-of-basis operation before the measurement. By default, measurement results for qubit :math:`i` are written to bit :math:`i` in an anonymous classical register. When transpiling to another quantum SDK (e.g. Qiskit), a single classical register is created, collating all measurement results. The measurement can also be sent to a named classical register using ``c.Measure(0,"RegName")``. :math:`\mathrm{t|ket}\rangle` does not currently support manipulation of the classical information or classically-controlled gates.

Some more advanced circuit construction features exist, such as the ability to construct the inverse circuit for any pure circuit (i.e. when it has no measurements).

::

    c_dg = c.dagger()

When constructing larger circuits, it is often useful to define small subroutines which can be applied multiple times. We can use this idea in pytket with circuit appending:

::

    c_sub = Circuit(2)
    c_sub.CX(0, 1)
    c_sub.Rz(1, 1.25)
    c_sub.CX(0, 1)
    c_sub.Rz(1, 0.5)

    c.append(c_sub)
    c.H(0)
    c.append(c_sub)

``c.append(c_sub)`` will copy ``c_sub`` into ``c``, and attach the input for qubit :math:`i` in ``c_sub`` to the output for qubit :math:`i` in ``c``. Further information on how to interact directly with :py:class:`Circuit` objects can be found in the API reference for the :py:class:`Circuit` class.

Compilation
-----------
In classical computing, a compiler is required to translate from a high-level language (e.g. C++) to the instruction set of the target machine (e.g. RISC-V). During this process, the program is automatically optimised to reduce the number of instructions, memory consumption, energy consumption, etc. The same needs exist in quantum computing to take a circuit description and enable it to run on a given simulator, hardware device, or cloud system (such as the IBM Quantum Experience or Rigetti's Quantum Cloud Service).

pytket is a retargetable compiler, with the ability to accept and produce circuits for any of the following:

* Google Cirq
* IBM Qiskit
* OpenQASM (via Qiskit)
* Rigetti pyQuil
* ProjectQ compilation engines
* PyZX

This translation facility allows, for example, circuits generated by the variational forms in Qiskit Aqua to be run on Rigetti QCS.

However, near-term quantum devices do not currently provide the nice abstraction of a "perfect" machine - they are troubled by imperfect fidelity of operations, gradual decoherence over time, and restricted qubit-adjacency only allowing two-qubit gates between specific positions on the architecture.

In pytket, there is a distinction between circuits for a perfect machine (the :py:class:`Circuit` class) and those that have been configured to conform to a specific device's constraints (the :py:class:`PhysicalCircuit` class). From a structural perspective, they are both simply collections of gates applied to some qubits, but they have different interpretations - :py:class:`Circuit` objects are device independent and qubit indexing refers to the logical qubits, whereas the :py:class:`PhysicalCircuit` gates act on physical qubits, indexed according to the numbering of the :py:class:`Architecture` nodes.

One such architectural constraint is the problem of qubit connectivity on heterogeneous architectures which can be solved by a quantum compiler with placement and routing procedures. This takes the adjacency graph of the architecture's qubits (the coupling map) and identifies a good mapping from the qubits of the circuit to the positions on the device to make it possible to perform as many of the two-qubit operations as possible. The circuit is then modified by introducing swaps to rearrange the logical qubits such that any multi-qubit operations occur between neighbouring physical qubits.

These are both performed simultaneously during a call to ``pytket._routing.route``. This takes a :py:class:`Circuit` object and an :py:class:`Architecture` object (encapsulating the coupling map of the device). More on the :py:class:`Architecture` class can be found in the API Reference.

::

    from pytket._routing import route, Architecture
    arc = Architecture([(0,1), (1,2), (2,3)], 4)
    routed_circuit = route(circuit, arc)
    routed_circuit.decompose_SWAP_to_CX()
    routed_circuit.redirect_CX_gates(arc)

The ``route`` method outputs a :py:class:`PhysicalCircuit` that is tailored to ``arc``. The output circuit will be semantically identical to the original logical circuit but will only have multi-qubit gates between pairs of nodes in the coupling map. The map between the logical qubits and the physical nodes they live on will change throughout the :py:class:`PhysicalCircuit`. However, any measurement operations will still map logical qubits to the same classical registers, so the qubit permutations will not affect the readouts from the device.

Routing typically sacrifices circuit size/depth (from inserting swaps) to satisfy device constraints. Sadly, the problems of decoherence and imperfect fidelities mean that physical devices will accumulate noise proportional to the size/depth of the circuit being run. :math:`\mathrm{t|ket}\rangle` provides some optimisation passes to rewrite parts of the circuit to produce an equivalent one with fewer operations/less depth. Some of the rewrite rules may not preserve the architectural constraints considered during routing, so we provide a safe ``OptimisePostRouting`` pass as well as a selection of more powerful :py:class:`Transform` passes which exploit common structures in quantum circuits.

::

    from pytket._transform import Transform
    Transform.OptimisePhaseGadgets(circuit)
    routed_circuit = route(circuit, arc)
    routed_circuit.decompose_SWAP_to_CX()
    routed_circuit.redirect_CX_gates(arc)
    Transform.OptimisePostRouting().apply(routed_circuit)

To see these in action, take a look at the jupyter notebooks on the `pytket GitHub repository <http://github.com/CQCL/pytket/tree/master/examples>`_.