## Restore your deployment

The Warden container will be lost after a VM reboot, but you can restore your deployment with `bosh cck`, BOSH's command for recovering from unexpected errors. Alternatively *Pause* the VM from the VirtualBox UI (or `vagrant suspend`) before shutting down, or making your computer sleep.

Note: the network routes may also need to be restored.  To do this run the `./bin/add-route` script before starting the `bosh cck`.

```
$ bosh cck
```

Choose `2` to recreate each missing VM:

```
Problem 1 of 13: VM with cloud ID `vm-74d58924-7710-4094-86f2-2f38ff47bb9a' missing.
  1. Ignore problem
  2. Recreate VM using last known apply spec
  3. Delete VM reference (DANGEROUS!)
Please choose a resolution [1 - 3]: 2
...
```

Type yes to confirm at the end:

```
Apply resolutions? (type 'yes' to continue): yes

Applying problem resolutions
  missing_vm 212: Recreate VM using last known apply spec (00:00:13)
  missing_vm 215: Recreate VM using last known apply spec (00:00:08)
...
Done                    13/13 00:03:48
```
