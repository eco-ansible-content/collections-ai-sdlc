# Simple Module Example: `hv_memory`

An example of a relatively simple Ansible Windows/Hyper-V module. This shows the baseline structure of split-plugins, documentation, and the 4-stage integration testing loop.

## Documentation & Metadata (`plugins/modules/hv_memory.py`)
```python
# -*- coding: utf-8 -*-
# Copyright (c) 2026, Ansible Cloud Team (@ansible)
# GNU General Public License v3.0+ (see COPYING or https://www.gnu.org/licenses/gpl-3.0.txt)

from __future__ import absolute_import, division, print_function
__metaclass__ = type

DOCUMENTATION = r'''
---
module: hv_memory
short_description: Manage Hyper-V Virtual Machine Memory Settings
description:
  - Manage memory allocation for Hyper-V Virtual Machines.
  - Configure Static RAM or enable Dynamic Memory (Dynamic RAM) to optimize host density.
  - Allows precise tuning of Startup, Minimum, Maximum, Buffer, and Priority settings.
options:
  name:
    description:
      - The name of the virtual machine to configure.
    type: str
    required: true
    aliases: [ vm_name ]
  dynamic_memory_enabled:
    description:
      - Specifies whether Dynamic Memory should be enabled.
      - Enabling this allows the host to reclaim unused memory from the VM to increase overall density.
    type: bool
  startup_bytes:
    description:
      - The amount of memory the VM requires to start up.
      - When Dynamic Memory is disabled, this effectively acts as the static RAM assignment.
      - Accepts an integer in bytes or a string like "4GB" or "1024MB".
    type: raw
  minimum_bytes:
    description:
      - The minimum amount of memory the VM will be allocated when Dynamic Memory is enabled.
      - Accepts an integer in bytes or a string like "512MB".
    type: raw
  maximum_bytes:
    description:
      - The maximum amount of memory the VM can grow to consume when Dynamic Memory is enabled.
      - Accepts an integer in bytes or a string like "16GB".
    type: raw
  buffer:
    description:
      - The percentage of memory the host should attempt to keep free as a buffer within the VM.
    type: int
  priority:
    description:
      - The memory weight/priority of the virtual machine when the host is under memory pressure.
      - Accepts an integer between 0 and 100.
    type: int
author:
  - Ansible Cloud Team (@ansible)
'''

EXAMPLES = r'''
- name: Configure Static Memory (8GB)
  microsoft.hyperv.hv_memory:
    name: StaticAppVM
    dynamic_memory_enabled: false
    startup_bytes: "8GB"

- name: Configure Dynamic Memory for high density (4GB to 16GB)
  microsoft.hyperv.hv_memory:
    name: DensityOptimizedVM
    dynamic_memory_enabled: true
    startup_bytes: "4GB"
    minimum_bytes: "2GB"
    maximum_bytes: "16GB"
    buffer: 20
    priority: 80
'''

RETURN = r'''
name:
    description: Name of the virtual machine.
    returned: always
    type: str
    sample: WebServer01
dynamic_memory_enabled:
    description: The final state of Dynamic Memory on the VM.
    returned: always
    type: bool
    sample: true
startup_bytes:
    description: The startup memory assigned to the VM, in bytes.
    returned: always
    type: int
    sample: 4294967296
minimum_bytes:
    description: The minimum memory guaranteed to the VM, in bytes.
    returned: always
    type: int
    sample: 2147483648
maximum_bytes:
    description: The maximum memory the VM can consume, in bytes.
    returned: always
    type: int
    sample: 17179869184
buffer:
    description: The current memory buffer percentage.
    returned: always
    type: int
    sample: 20
priority:
    description: The current memory priority weight.
    returned: always
    type: int
    sample: 50
'''
```

## PowerShell Logic (`plugins/modules/hv_memory.ps1`)
```powershell
#!powershell

# Copyright (c) 2026, Ansible Cloud Team (@ansible)
# GNU General Public License v3.0+ (see COPYING or https://www.gnu.org/licenses/gpl-3.0.txt)

#AnsibleRequires -CSharpUtil Ansible.Basic
#AnsibleRequires -PowerShell ansible_collections.microsoft.hyperv.plugins.module_utils.HyperV

$spec = @{
    options = @{
        name = @{ type = "str"; required = $true; aliases = @("vm_name") }
        dynamic_memory_enabled = @{ type = "bool" }
        startup_bytes = @{ type = "raw" }
        minimum_bytes = @{ type = "raw" }
        maximum_bytes = @{ type = "raw" }
        buffer = @{ type = "int" }
        priority = @{ type = "int" }
    }
    supports_check_mode = $true
}

$module = [Ansible.Basic.AnsibleModule]::Create($args, $spec)

$name = $module.Params.name
$startup_bytes = $module.Params.startup_bytes
$minimum_bytes = $module.Params.minimum_bytes
$maximum_bytes = $module.Params.maximum_bytes

if ($null -ne $startup_bytes) {
    $startup_bytes = Convert-ToByte -SizeString $startup_bytes
    $module.Params.startup_bytes = $startup_bytes
}
if ($null -ne $minimum_bytes) {
    $minimum_bytes = Convert-ToByte -SizeString $minimum_bytes
    $module.Params.minimum_bytes = $minimum_bytes
}
if ($null -ne $maximum_bytes) {
    $maximum_bytes = Convert-ToByte -SizeString $maximum_bytes
    $module.Params.maximum_bytes = $maximum_bytes
}

$module.Result.name = $name

# Define the mapping between Ansible params and Hyper-V properties
$propertyMap = @(
    @{ Param = "dynamic_memory_enabled"; Property = "DynamicMemoryEnabled"; Type = "bool" }
    @{ Param = "startup_bytes"; Property = "Startup"; Type = "long"; CmdletParam = "StartupBytes" }
    @{ Param = "minimum_bytes"; Property = "Minimum"; Type = "long"; CmdletParam = "MinimumBytes" }
    @{ Param = "maximum_bytes"; Property = "Maximum"; Type = "long"; CmdletParam = "MaximumBytes" }
    @{ Param = "buffer"; Property = "Buffer"; Type = "int" }
    @{ Param = "priority"; Property = "Priority"; Type = "int" }
)

try {
    $vm = Get-VM -Name $name -ErrorAction SilentlyContinue

    if (-not $vm) {
        $module.FailJson("Virtual Machine '$name' not found.")
    }

    $mem = Get-VMMemory -VMName $name -ErrorAction SilentlyContinue
    if (-not $mem) {
        $module.FailJson("Failed to retrieve memory configuration for VM '$name'.")
    }

    $changed = Test-HyperVPropertiesChanged -PropertyMap $propertyMap -CurrentObject $mem -AnsibleParams $module.Params
    $module.Result.changed = $changed

    if ($changed) {
        if ($vm.State -ne 'Off' -and $null -ne $module.Params.startup_bytes) {
            $module.FailJson("Cannot apply Memory changes (Startup Bytes) while the VM is not Off. Stop the VM first.")
        }

        if ($module.CheckMode) {
            Set-HyperVResultFromMap -PropertyMap $propertyMap -CurrentObject $mem -ModuleResult $module.Result
            # Override with desired
            foreach ($map in $propertyMap) {
                $paramValue = $module.Params.($map.Param)
                if ($null -ne $paramValue) {
                    $module.Result.($map.Param) = $paramValue
                }
            }
            $module.ExitJson()
        }

        # Build parameters using map but handle the CmdletParam overrides
        $cmdParams = @{ VMName = $name }
        foreach ($map in $propertyMap) {
            $paramValue = $module.Params.($map.Param)
            if ($null -ne $paramValue) {
                $targetParam = if ($null -ne $map.CmdletParam) { $map.CmdletParam } else { $map.Property }
                $cmdParams.($targetParam) = $paramValue
            }
        }

        Set-VMMemory @cmdParams | Out-Null
        $mem = Get-VMMemory -VMName $name
    }

    Set-HyperVResultFromMap -PropertyMap $propertyMap -CurrentObject $mem -ModuleResult $module.Result

    $module.ExitJson()
}
catch {
    $module.FailJson("Failed to manage VM Memory: $($_.Exception.Message)")
}
```

## Integration Test Tasks (`tests/integration/targets/hv_memory/tasks/main.yml`)
```yaml
---
- name: Define test VM name
  set_fact:
    test_vm_name: "AnsibleTestMem_{{ 99999 | random }}"

- block:
    - name: Setup - Create a temporary dummy VM for testing
      microsoft.hyperv.hv_vm:
        name: "{{ test_vm_name }}"
        state: present
        generation: 2
        memory_startup_bytes: "256MB"

    - name: Run tests
      ansible.builtin.include_tasks: tests.yml

  always:
    - name: Teardown - Remove temporary dummy VM
      microsoft.hyperv.hv_vm:
        name: "{{ test_vm_name }}"
        state: absent
      ignore_errors: true
```

## Integration Test Aliases (`tests/integration/targets/hv_memory/aliases`)
```text
shippable/windows/group1
windows
```

## Integration Test Meta (`tests/integration/targets/hv_memory/meta/main.yml`)
```yaml
dependencies:
  - setup_hyperv
```
