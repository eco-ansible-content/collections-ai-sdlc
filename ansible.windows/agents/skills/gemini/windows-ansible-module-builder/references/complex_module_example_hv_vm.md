# Complex Module Example: `hv_vm`

An example of a highly complex Ansible Windows/Hyper-V module, demonstrating complex state management, extensive parameter mapping, and deep integration with module utilities.

## Documentation & Metadata (`plugins/modules/hv_vm.py`)
```python
# -*- coding: utf-8 -*-
# Copyright (c) 2026, Ansible Cloud Team (@ansible)
# GNU General Public License v3.0+ (see COPYING or https://www.gnu.org/licenses/gpl-3.0.txt)

from __future__ import absolute_import, division, print_function
__metaclass__ = type

DOCUMENTATION = r'''
---
module: hv_vm
short_description: Manage Hyper-V Virtual Machines
description:
  - Create, remove, and manage the base configuration of virtual machines on a Hyper-V host.
  - Wraps New-VM and Remove-VM cmdlets.
  - Supports Generation 1 and Generation 2 VMs.
options:
  name:
    description:
      - The name of the virtual machine.
    type: str
    required: true
    aliases: [ vm_name ]
  state:
    description:
      - The desired state of the VM.
      - C(present) ensures the VM exists.
      - C(absent) ensures the VM is removed.
    type: str
    choices: [ present, absent ]
    default: present
  generation:
    description:
      - The generation of the virtual machine (1 or 2).
      - Only used when creating a new VM.
    type: int
    choices: [ 1, 2 ]
    default: 1
  memory_startup_bytes:
    description:
      - The amount of memory to assign to the virtual machine at startup.
      - Accepts an integer in bytes or a string like "4GB", "1024MB".
      - Only used when creating a new VM.
    type: raw
  boot_device:
    description:
      - The device the VM should boot from.
      - Maps to the BootDevice parameter in New-VM.
    type: str
    choices: [ CD, Floppy, IDE, LegacyNetworkAdapter, NetworkAdapter, VHD ]
author:
  - Ansible Cloud Team (@ansible)
'''

EXAMPLES = r'''
- name: Create a Gen2 VM with 4GB RAM
  microsoft.hyperv.hv_vm:
    name: Web01
    state: present
    generation: 2
    memory_startup_bytes: 4GB

- name: Remove a VM
  microsoft.hyperv.hv_vm:
    name: Web01
    state: absent
'''

RETURN = r'''
name:
    description: Name of the virtual machine.
    returned: always
    type: str
    sample: Web01
state:
    description: The final state of the virtual machine (present or absent).
    returned: always
    type: str
    sample: present
'''
```

## PowerShell Logic (`plugins/modules/hv_vm.ps1`)
```powershell
#!powershell

# Copyright (c) 2026, Ansible Cloud Team (@ansible)
# GNU General Public License v3.0+ (see COPYING or https://www.gnu.org/licenses/gpl-3.0.txt)

#AnsibleRequires -CSharpUtil Ansible.Basic
#AnsibleRequires -PowerShell ansible_collections.microsoft.hyperv.plugins.module_utils.HyperV

$spec = @{
    options = @{
        name = @{ type = "str"; required = $true; aliases = @("vm_name") }
        state = @{ type = "str"; default = "present"; choices = @("present", "absent") }
        generation = @{ type = "int"; default = 1; choices = @(1, 2) }
        memory_startup_bytes = @{ type = "raw" }
        boot_device = @{ type = "str"; choices = @("CD", "Floppy", "IDE", "LegacyNetworkAdapter", "NetworkAdapter", "VHD") }
    }
    supports_check_mode = $true
}

$module = [Ansible.Basic.AnsibleModule]::Create($args, $spec)

$name = $module.Params.name
$state = $module.Params.state

if ($null -ne $module.Params.memory_startup_bytes) {
    $module.Params.memory_startup_bytes = Convert-ToByte -SizeString $module.Params.memory_startup_bytes
}

$module.Result.name = $name

# Mapping for New-VM parameters
$vmCreateMap = @(
    @{ Param = "generation"; Property = "Generation" }
    @{ Param = "memory_startup_bytes"; Property = "MemoryStartupBytes" }
    @{ Param = "boot_device"; Property = "BootDevice" }
)

try {
    $vm = Get-VM -Name $name -ErrorAction SilentlyContinue

    switch ($state) {
        "present" {
            if ($vm) {
                $module.Result.state = "present"
                $module.ExitJson()
            }

            $module.Result.changed = $true
            $module.Result.state = "present"

            if ($module.CheckMode) {
                $module.ExitJson()
            }

            $cmdParams = @{ Name = $name }
            $cmdParams += Get-HyperVParametersFromMap -PropertyMap $vmCreateMap -AnsibleParams $module.Params

            New-VM @cmdParams | Out-Null
        }
        "absent" {
            if (-not $vm) {
                $module.Result.state = "absent"
                $module.ExitJson()
            }

            $module.Result.changed = $true
            $module.Result.state = "absent"

            if ($module.CheckMode) {
                $module.ExitJson()
            }

            if ($vm.State -eq 'Running') {
                Stop-VM -Name $name -TurnOff -Force
            }

            Remove-VM -Name $name -Force
        }
    }

    $module.ExitJson()
}
catch {
    $module.FailJson("Failed to manage VM: $($_.Exception.Message)")
}
```

## Integration Test Tasks (`tests/integration/targets/hv_vm/tasks/main.yml`)
```yaml
---
- name: Define test VM name
  set_fact:
    test_vm_name: "AnsibleTestVM_{{ 99999 | random }}"

- name: Run hv_vm integration tests
  module_defaults:
    microsoft.hyperv.hv_vm:
      name: "{{ test_vm_name }}"
  block:
    - name: Run tests
      ansible.builtin.include_tasks: tests.yml

  always:
    - name: Teardown - Ensure test VM is removed (cleanup)
      microsoft.hyperv.hv_vm:
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
