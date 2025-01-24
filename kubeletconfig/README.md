# Kubelet Configuration

## Consider the Following Guidance

- **Create KubeletConfig CRs:**
  - Create one `KubeletConfig` Custom Resource (CR) for each machine config pool with all the configuration changes you want for that pool.
  - If you are applying the same content to all of the pools, you need only one `KubeletConfig` CR for all of the pools.

- **Editing KubeletConfig CRs:**
  - Edit an existing `KubeletConfig` CR to modify existing settings or add new settings, instead of creating a CR for each change.
  - It is recommended that you create a CR only to modify a different machine config pool, or for changes that are intended to be temporary, so that you can revert the changes.

- **Managing Multiple KubeletConfig CRs:**
  - As needed, create multiple `KubeletConfig` CRs with a limit of 10 per cluster.
  - For the first `KubeletConfig` CR, the Machine Config Operator (MCO) creates a machine config appended with `kubelet`.
  - With each subsequent CR, the controller creates another kubelet machine config with a numeric suffix. For example, if you have a kubelet machine config with a `-2` suffix, the next kubelet machine config is appended with `-3`.

- **Deleting Machine Configs:**
  - If you want to delete the machine configs, delete them in reverse order to avoid exceeding the limit. For example, you delete the `kubelet-3` machine config before deleting the `kubelet-2` machine config.

