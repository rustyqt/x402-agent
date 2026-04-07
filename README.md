# x402-agent

## Getting Started

Clone the repository with all submodules:

```bash
git clone --recurse-submodules https://github.com/rustyqt/x402-agent.git
```

If you've already cloned without submodules, initialize them with:

```bash
git submodule update --init --recursive
```

## Submodules

- `x402/` — [x402 Payment Protocol SDK](https://github.com/x402-foundation/x402)
- `web3.py/` — [Ethereum Web3.py](https://github.com/ethereum/web3.py)

## Updating Submodules

To pull the latest changes including submodule updates:

```bash
git pull --recurse-submodules
```
