# Kinetex xSwap

Kinetex xSwap smart contracts for audit

## Development

This project uses the following stack:

- Language: Solidity v0.8.16
- Framework: Hardhat
- Node.js: v18
- Yarn: v1.22

### Setup Environment

1. Ensure you have relevant Node.js version. For NVM: `nvm use`

2. Install dependencies: `yarn install`

3. Setup environment variables:

    * Clone variables example file: `cp .env.example .env`
    * Edit `.env` according to your needs

### Commands

Below is the list of commands executed via `yarn` with their descriptions:

 Command                | Alias            | Description
------------------------|------------------|------------
 `yarn hardhat`         | `yarn h`         | Call [`hardhat`](https://hardhat.org/) CLI
 `yarn build`           | `yarn b`         | Compile contracts (puts to `artifacts` folder)

## Licensing

The primary license for Kinetex xSwap is the Business Source License 1.1 (`BUSL-1.1`), see [`LICENSE`](./LICENSE).
However, some files are dual licensed under `GPL-2.0-or-later`:

- Several files in `contracts/` may also be licensed under `GPL-2.0-or-later` (as indicated in their SPDX headers),
  see [`LICENSE_GPL2`](./LICENSE_GPL2)

### Other Exceptions

- All `@openzeppelin` library files are licensed under `MIT` (as indicated in its SPDX header),
  see [`LICENSE_MIT`](./LICENSE_MIT)
