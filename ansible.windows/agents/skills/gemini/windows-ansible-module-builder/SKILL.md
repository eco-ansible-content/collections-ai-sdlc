---
name: windows-ansible-module-builder
description: Generates or refactors Ansible collections specifically for Windows and Hyper-V. Use when the user asks to "create a new module", "build an Ansible module", or "write integration tests for Hyper-V".
---

# Ansible Module Builder Skill (Windows / Hyper-V)

You are acting as an expert automation engineer generating code for the `microsoft.hyperv` (or similar) Ansible collection.

## Workflow: The Split-Plugin Architecture

When generating a new module, you MUST create a split architecture:

1. **The Python Metadata File (`plugins/modules/hv_<name>.py`)**
   - Strictly contains `DOCUMENTATION`, `EXAMPLES`, and `RETURN` blocks.
   - Do not include execution logic here.

2. **The PowerShell Logic File (`plugins/modules/hv_<name>.ps1`)**
   - Contains pure PowerShell logic.
   - Must use `[Ansible.Basic.AnsibleModule]::Create($args)` to handle input/output.
   - Must leverage `plugins/module_utils/HyperV.psm1` (if in the Hyper-V project) for property mapping and type conversion (e.g., `Get-HyperVParametersFromMap`).

## Workflow: Integration Testing (The 4-Stage Loop)

For every module, you must generate corresponding integration tests in `tests/integration/targets/hv_<name>/`:

1. **`aliases` file:** Must contain `shippable/windows/groupN` and `windows`.
2. **`meta/main.yml`:** Include `dependencies: [setup_role]`.
3. **`tasks/main.yml`:** Must enforce the following 4 testing stages:
   - **Initial Change:** Verify the action works (`changed: true`).
   - **Idempotency:** Run the exact task again (`changed: false`).
   - **Check Mode:** Run with `check_mode: true`.
   - **Error Handling:** Intentionally trigger an error and `assert` the failure.
   - *Note: Always use an `always` block for teardown/cleanup.*

## Workflow: The Four-Pillar Audit

Before confirming completion to the user, ensure you have audited your generated code against:
1. **Best Practices:** Clean linting, `ErrorAction Stop` on Cmdlets.
2. **Testing Safety:** Idempotency and `check_mode` strictly tested.
3. **Code Reuse:** Utilities leveraged to avoid redundant logic.
4. **Functional Coverage:** All module inputs and execution paths have a test case.

## References & Examples
When building modules, refer to these documents in the `references/` directory for structural blueprints:
- `standard_structures.md`: High-level guide on collection and infrastructure layout.
- `simple_module_example_hv_memory.md`: Baseline example of a simple module and its 4-stage tests.
- `complex_module_example_hv_vm.md`: Deep-dive into complex state management and parameter mapping.
