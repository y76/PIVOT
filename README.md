# PIVOT - Supporting Private and Efficient Consumer Requests to Data Brokers

Source code for **PIVOT**, from the paper:

> **Supporting Private and Efficient Consumer Requests to Data Brokers**
> Elina van Kempen, Pavel Frolikov, Chloe Georgiou, Gene Tsudik — University of California, Irvine
> *WPES '26: Proceedings of the 25th Workshop on Privacy in the Electronic Society*, pp. 173–185, The Hague, Netherlands, November 2026.
> DOI: [10.1145/3847192.3847365](https://doi.org/10.1145/3847192.3847365) · [ACM Digital Library (open access)](https://dl.acm.org/doi/10.1145/3847192.3847365)

## Overview

Under GDPR and CCPA, consumers can ask data brokers to **access**, **delete**, or **opt out** of the collection of their personal information. In practice, this means sending a separate request to each of hundreds of brokers, and each one runs its own identity check that often asks for more personally identifiable information (PII). Brokers that held no data on the consumer can end up learning new PII from the request itself.

PIVOT lets a consumer send these requests through an **untrusted intermediary** (MED):

- The consumer encrypts their PII **once** and is not involved after that.
- The intermediary carries out identity verification with every registered broker and **learns no PII**.
- Each broker learns only **whether its records match** the consumer. It learns no PII beyond what it already holds.

PIVOT combines **unidirectional proxy re-encryption** (AFGH06) with **additively homomorphic encryption**. Together these give a private list intersection (PLI) check between the consumer's encrypted PII and each broker's records:

1. **Consumer:** encrypts each PII element as `Enc2(-H(x_i))` and generates a re-encryption key `rk_{C→B_j}` for each broker, then sends everything to the intermediary.
2. **Broker:** encrypts its own record for the consumer as `Enc1(H(y_i))` and sends it to the intermediary.
3. **Intermediary:** re-encrypts the consumer's ciphertexts for the broker, adds them homomorphically to get `Enc(H(y_i) − H(x_i))`, blinds each result with a random scalar, and returns them to the broker.
4. **Broker:** partially decrypts each result and checks for the identity element in G_T. Identity means the element matches, and enough matches mean the identity is verified. The broker then processes the access, delete or opt-out request.

## Repository layout

| File | Role |
|---|---|
| `my_afgh06.py` | AFGH06 proxy re-encryption (Charm), with additively homomorphic changes (encoding messages as `Z^m`) |
| `IVDrip.py` | Core PIVOT operations: `consumer`, `intermediary`, `broker_enc`, `broker_pli`, `getPIIList` |
| `setup.py` | One-time generation of global parameters and key pairs, written to `params.bin` / `keys.bin` |
| `config.py` | Loads the parameters and keys at runtime for all participants |
| `consumer.py` | Consumer C: encrypts PII, builds re-encryption keys, sends to the intermediary over TLS |
| `intermediary.py` | Intermediary MED: re-encryption, homomorphic addition, blinding |
| `broker.py` | Broker BR_j: encrypts its records, partially decrypts, checks matches |
| `bench.py`, `bench.sh` | Benchmark harness used for the paper's evaluation (Section 8) |
| `data.txt` | Sample consumer PII as `label: "value"` pairs |
| `*.crt`, `*.key` | Self-signed TLS certificates for local testing only |

All parties talk over TLS-secured TCP sockets on localhost (intermediary `:8443`/`:8444`, broker `:8555`).

## Requirements

- Python 3.13 (the paper's version is 3.13.3)
- [PBC library](https://github.com/blynn/pbc) (needs GMP)
- [Charm-Crypto](https://github.com/JHUISI/charm)

A virtual environment is the recommended setup:

```bash
python3 -m venv ~/venv/charm
source ~/venv/charm/bin/activate
# build/install PBC, then Charm, following their READMEs
```

Check that Charm works:

```bash
python IVDrip.py
```

## Running

```bash
source ~/venv/charm/bin/activate

# 1. Generate parameters and keys (argument = number of brokers, default 500)
python setup.py 500

# 2. Start the parties, each in its own terminal: broker, then intermediary, then consumer.
#    Arguments: <num_brokers> <num_pii_per_broker>
python broker.py 500 10
python intermediary.py 500 10
python consumer.py 500 10
```

Each participant appends its runtime and the bytes it sent and received to `consumer.csv`, `intermediary.csv` and `broker.csv`.

## Reproducing the evaluation

```bash
# One configuration: 10 runs, 500 brokers, 10 PII elements per broker
python bench.py -n 10 -b 500 -p 10

# Full sweep from the paper (PII ∈ {10, 20, 30}; brokers ∈ {100, …, 1000})
./bench.sh
```

Key results from the paper (MacBook Pro M4, 24 GB RAM, localhost, averaged over 10 runs):

- **Consumer:** with 600 brokers, PIVOT takes 0.34 s. Encrypting directly for each broker takes 4.85 s, so PIVOT is **14.2× faster**.
- **Intermediary:** about 6 ms per consumer–broker pair at 10 PII elements, regardless of how many brokers there are. Each pair is independent, so the work parallelizes. Storage is about 81 KB per consumer.
- **Broker:** about 8 / 15 / 23 ms for 10 / 20 / 30 PII elements.

## Citation

```bibtex
@inproceedings{vankempen2026pivot,
  author    = {van Kempen, Elina and Frolikov, Pavel and Georgiou, Chloe and Tsudik, Gene},
  title     = {Supporting Private and Efficient Consumer Requests to Data Brokers},
  booktitle = {Proceedings of the 25th Workshop on Privacy in the Electronic Society},
  series    = {WPES '26},
  year      = {2026},
  pages     = {173--185},
  publisher = {ACM},
  address   = {New York, NY, USA},
  location  = {The Hague, Netherlands},
  doi       = {10.1145/3847192.3847365},
  url       = {https://doi.org/10.1145/3847192.3847365}
}
```
