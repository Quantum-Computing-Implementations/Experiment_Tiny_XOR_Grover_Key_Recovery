# Experiment: Quantum Key Recovery on Toy XOR Cipher via Grover's Algorithm

[![Qiskit](https://img.shields.io/badge/Qiskit-1.1%2B-blue.svg)](https://qiskit.org/)
[![OpenQASM](https://img.shields.io/badge/OpenQASM-3.0-darkgreen.svg)](https://openqasm.com/)
[![Project Type](https://img.shields.io/badge/Project%20Type-Educational%20%2F%20Research-orange.svg)](#educational-purpose--scope)
[![Python](https://img.shields.io/badge/Python-3.9%2B-green.svg)](https://python.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![IBM Quantum](https://img.shields.io/badge/IBM%20Quantum-Runtime%20V2-purple.svg)](https://quantum.ibm.com/)

> [!NOTE]
> ### 🎓 Educational Project Disclaimer
> This repository is an **educational and pedagogical research project** designed to illustrate the core mechanics of **Grover's Quantum Search Algorithm**, quantum circuit construction, oracle uncomputation, and real-world NISQ hardware noise dynamics. 
> 
> The 4-bit XOR cipher investigated here is intentionally simplified as a **toy model** so that every single qubit, quantum gate, phase kickback mechanism, and reflection step can be visualized, traced, and analyzed step-by-step without requiring black-box approximations. It is not intended for attacking real-world cryptographic schemes (e.g., AES).

An end-to-end quantum cryptanalysis study implementing a **Known-Plaintext Attack (KPA)** on a 4-bit toy XOR cipher using **Grover's Quantum Search Algorithm**. The project demonstrates both ideal noiseless simulation using `AerSimulator` and physical execution on the 156-qubit superconducting quantum processor **`ibm_marrakesh`**.

---

## 🔬 Problem Formulation & Cryptographic Model

In a Known-Plaintext Attack (KPA), an adversary possesses a known plaintext block $P$ and its matching ciphertext block $C$, encrypted under a secret key $K$:
$$C = P \oplus K$$

Given:
- **Plaintext ($P$)**: `1010` ($10_{10}$)
- **Target Ciphertext ($C$)**: `1100` ($12_{10}$)
- **Key Space ($N$)**: $N = 2^4 = 16$ candidate keys ($k \in \{0, 1\}^4$)

The adversary searches for the unique key $K$ satisfying $P \oplus K = C$:
$$K = P \oplus C = 1010_2 \oplus 1100_2 = 0110_2 \quad (6_{10})$$

---

## ⚛️ Quantum Circuit Architecture

The complete circuit utilizes **13 quantum bits (qubits)** and **4 classical bits**:

```
q[0..3]   (Plaintext):  |1010> ──────────────────────────────────────── (Classical Input)
q[4..7]   (Key Reg):    |0000> ──[H]─[  Grover Oracle  ]─[  Diffuser  ]─[M] -> c[0..3]
q[8..11]  (Ciphertext): |0000> ──────[  XOR Evaluation ]─[ Uncompute  ]─ (Restored to |0>)
q[12]     (Ancilla):    |1> ────[H]──[  Phase Kickback ]──────────────── (Restored)
```

### Register Allocation Map
| Register | Qubits | Role | Implementation |
| :--- | :--- | :--- | :--- |
| **Plaintext Register** ($P$) | `q[0..3]` | Input State | Initialized classically to $1010_2$ via Pauli-$X$ gates on `q[0]` and `q[2]`. |
| **Key Candidate Register** ($k$) | `q[4..7]` | Search Space | Initialized into uniform superposition $\frac{1}{\sqrt{16}}\sum_{k}|k\rangle$ via $H^{\otimes 4}$. Rotated via Diffusers. |
| **Output / Ciphertext Register** ($C'$) | `q[8..11]` | Oracle Work | Computes $C' = P \oplus k$ via CNOTs. Uncomputed back to $|0000\rangle$ after marking. |
| **Ancilla Qubit** | `q[12]` | Phase Kickback | Initialized in $|-\rangle$ to apply $(-1)^{f(k)}$ to the matching state via Multi-Controlled-X ($MCX$). |
| **Classical Register** ($c$) | `c[0..3]` | Measurement | Records collapsed state of key register `q[4..7]` upon final measurement. |

---

## 📐 Grover Search Theoretical Calculations

1. **Optimal Iterations ($R$)**:
   For $N = 16$ states and $M = 1$ target solution:
   $$R \approx \left\lfloor \frac{\pi}{4}\sqrt{\frac{N}{M}} \right\rceil = \left\lfloor \frac{\pi}{4}\sqrt{16} \right\rceil = \lfloor 3.1416 \rceil = 3 \text{ iterations}$$

2. **Theoretical Peak Probability**:
   With initial angle $\sin(\theta) = \frac{1}{\sqrt{16}} = 0.25$ ($\theta \approx 0.2527 \text{ rad}$), after $R = 3$ iterations:
   $$\theta_3 = (2R + 1)\theta = 7\theta \approx 1.7688 \text{ rad}$$
   $$\mathcal{P}_{\text{theory}}(\text{Target Key}) = \sin^2(7\theta) \approx \sin^2(1.7688) \approx 96.13\%$$

---

## 📊 Experimental Results & Physical Hardware Analysis

The circuit was executed with **1024 shots** on both `AerSimulator` and physical hardware `ibm_marrakesh` (Job ID: `dc2tecmq4uuc73a5ulag`):

| Metric / Parameter | Aer Simulator (Ideal) | IBM Marrakesh (Physical QPU) | Notes & Physics Interpretation |
| :--- | :--- | :--- | :--- |
| **Target Key State** | `0110` ($6$) | `0110` ($6$) | Target key from $P \oplus C = 1010_2 \oplus 1100_2$ |
| **Target Counts** | **988 / 1024** | **91 / 1024** | Hardware peak still identifies target among the highest states |
| **Empirical Success Rate**| **96.48%** | **8.89%** | Uniform random baseline is $\frac{1}{16} = 6.25\%$ (~64 shots) |
| **Circuit Depth** | 22 (Logical) | **1074** (Transpiled) | $\sim 48\times$ depth overhead from coupling graph routing |
| **2-Qubit Gates (CZ)**| 0 native CZ | **486 CZ gates** | Primary driver of coherent gate errors and decoherence |
| **1-Qubit Gates (SX)**| - | **950 SX gates** | Single-qubit pulse rotations |

### NISQ Physical Limitations & Decoherence
- **Gate Fidelity Decay**: With an average two-qubit gate error of $\epsilon_{CZ} \approx 0.8\%$, cumulative circuit fidelity is approximately:
  $$\mathcal{F} \approx (1 - 0.008)^{486} \approx 0.020 \quad (\sim 2\%)$$
- **Decoherence**: Circuit depth of 1074 exceeds coherence times ($T_1, T_2$), driving the output toward a uniform mixed state. Despite this noise floor, target key `0110` persists above the baseline (91 counts vs 64 expected for pure noise), confirming physical constructive interference.

---

## 📂 Repository Structure

```text
Experiment_Tiny_XOR_Grover_Key_Recovery/
├── 📁 Experiment_Artifacts/
│   ├── grover_xor_circuit.qasm
│   ├── grover_xor_circuit.qpy
│   ├── simulator_counts.json
│   └── simulator_histogram.png
├── 📁 notebooks/
│   ├── 01_Classical_Encryption.ipynb
│   └── 02_Experiment_Tiny_XOR_Grover_Key_Recovery.ipynb
├── .gitignore
├── requirements.txt
└── README.md
```

### 📋 File & Directory Breakdown

| Item | Type | Description |
| :--- | :---: | :--- |
| 📁 [`Experiment_Artifacts/`](Experiment_Artifacts/) | **Directory** | Standalone circuit files and simulation benchmark datasets |
| ├── `grover_xor_circuit.qasm` | Circuit | Vendor-neutral **OpenQASM 3.0** circuit export |
| ├── `grover_xor_circuit.qpy` | Binary | Fast, lossless **Qiskit binary** circuit format |
| ├── `simulator_counts.json` | Dataset | Raw 1024-shot measurement count distributions across all 16 states |
| └── `simulator_histogram.png` | Plot | High-resolution publication chart of Grover amplitude amplification |
| 📁 [`notebooks/`](notebooks/) | **Directory** | Sequential Jupyter research workflows (Parts 1 & 2) |
| ├── [`01_Classical_Encryption.ipynb`](notebooks/01_Classical_Encryption.ipynb) | Notebook | **Part 1**: Classical XOR cipher, Decryption, & KPA Brute-Force search ($\mathcal{O}(N)$) |
| └── [`02_Experiment_Tiny_XOR_Grover_Key_Recovery.ipynb`](notebooks/02_Experiment_Tiny_XOR_Grover_Key_Recovery.ipynb) | Notebook | **Part 2**: Grover Quantum Key Recovery ($\mathcal{O}(\sqrt{N})$) on Aer & IBM QPU (`ibm_marrakesh`) |
| 📄 [`requirements.txt`](requirements.txt) | Config | Required Python libraries (`qiskit`, `qiskit-aer`, `qiskit-ibm-runtime`) |
| 📖 [`README.md`](README.md) | Docs | End-to-end documentation, mathematical proofs, and hardware analysis |

---

## 📦 Experiment Artifacts

The `Experiment_Artifacts/` folder contains standalone reproduction files:
- **`grover_xor_circuit.qasm`**: Standalone OpenQASM 3.0 source code for porting to any QASM-compliant simulator or hardware provider.
- **`grover_xor_circuit.qpy`**: Serialized binary representation for fast, lossless deserialization in Python with `qiskit.qpy.load()`.
- **`simulator_counts.json`**: Exact raw histogram count dictionaries across all 16 basis states.
- **`simulator_histogram.png`**: Publication-ready graphical plot showing Grover amplitude amplification contrast.

---

## 🚀 Getting Started

### 1. Clone the repository
```bash
git clone https://github.com/Quantum-Computing-Implementations/Experiment_Tiny_XOR_Grover_Key_Recovery.git
cd Experiment_Tiny_XOR_Grover_Key_Recovery
```

### 2. Install dependencies
```bash
pip install -r requirements.txt
```

### 3. Run the notebooks
Launch JupyterLab or VS Code:
```bash
jupyter lab notebooks/02_Experiment_Tiny_XOR_Grover_Key_Recovery.ipynb
```

To run on IBM Quantum hardware, provide your IBM Quantum API Token:
- In Google Colab: Add secret `IBM_QUANTUM_TOKEN`
- Locally: set environment variable `IBM_QUANTUM_TOKEN` or save credentials via `QiskitRuntimeService.save_account(token=...)`.

---

## 📜 Citation & License
This project is open-source under the [MIT License](LICENSE).

---

## 👥 Authors

- **Mina Wagih** - [@MinaWagihz](https://github.com/MinaWagihz)
