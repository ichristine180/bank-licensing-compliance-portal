# bank-licensing-compliance-portal

Parent repository for the Bank Licensing & Compliance Portal workspace.

## Structure

- `docs/` contains project documentation and design assets.
- `moduleRepos/backend/` is the backend Git submodule.
- `moduleRepos/frontend/` is the frontend Git submodule.

## Clone With Modules

Clone the parent repository together with its modules:

```bash
git clone --recurse-submodules https://github.com/ichristine180/bank-licensing-compliance-portal.git
```

If the parent repo is already cloned, fetch the modules with:

```bash
git submodule update --init --recursive
```
