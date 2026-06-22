Cluster NFS
===============

A simple NFS server and client playbook intended for Cluster-as-a-Service cloud
deployments.

Requirements
------------

The `community.general` collection is required (the client read-ahead task uses
`community.general.ini_file`). Install it with:

    ansible-galaxy collection install -r requirements.yml

Role Variables
--------------

`nfs_fstype` is the type of filesystem to create on the disk. Optional, default "xfs".

`nfs_disk_location` is the path to the block device on which to create a filesystem for export. Optional, default does not create a filesystem (e.g. as when exporting an existing directory).

`nfs_export` is the path to exported filesystem mountpoint on the NFS server. Optional, default "/srv".

`nfs_export_clients` is the client list allowed to mount the filesystem.
See "Machine Name Formats` in `man exports`. Optional string, default "*".
Note that multiple clients may be specified as a space-separated string (not a yaml list).

`nfs_export_options` are the options to apply to the export. Optional, default "rw,secure,root_squash".

`nfs_client_mnt_point` is the path to the mountpoint on the NFS clients. Optional, default "/mnt".

`nfs_client_mnt_options` allows passing mount options to the NFS client. Optional, default "defaults,nosuid,nodev".

`nfs_client_mnt_state` desired state for the mount. As passed to the ansible `mount` 
builtin module. Can be one of "absent", "mounted", "present", "unmounted" or 
"remounted". Optional, default "mounted".

`nfs_server` is the IP address or hostname of the NFS server.

`nfs_enable`: a mapping with keys `server` and `client` - values are bools determining the role of the host.

`nfs_server_num_threads` is the number of `nfsd` threads configured on the NFS server, written to the `[nfsd]` section of `/etc/nfs.conf` and also applied live via `/proc/fs/nfsd/threads` (no restart required). Optional, default `2 ×` the number of vCPUs.

`nfs_client_rahead` is a mapping of NFS mount type to client-side read-ahead value (in 512-byte blocks), written to the `[nfsrahead]` section of `/etc/nfs.conf` and applied by the `nfsrahead` helper at mount time. Raising it above the stock 128-block default avoids throttling large sequential reads off NFS exports. Set to `{}` to leave `/etc/nfs.conf` untouched. Optional, default `{nfs4: "32768", nfs: "32768", default: "128"}`.

Multiple NFS client/server configurations may be provided by defining `nfs_configurations`. This should be a list of mappings with keys/values are as per the variables above. Omitted keys/values are filled from the corresponding variable. For example if all configurations require the same non-default client mount options, define `nfs_client_mnt_options` and omit the key "nfs_client_mnt_options" from all configuration mappings.

Dependencies
------------

The `community.general` Ansible collection - see [Requirements](#requirements).

Example Playbook
----------------

Assuming:
- An inventory group `nfs_server` containing a single host
- An inventory group `nfs_clients` containing one or more clients
- The hostvar `ansible_host` containing hosts' IP address

the example below configures a root-squashed read/write share which can only
be mounted by the clients.

    ---
    - hosts:
      - nfs_server
      - nfs_clients
      become: yes
      roles:
        - role: stackhpc.nfs
          nfs_enable:
            server: "{{ inventory_hostname in groups['nfs_server'] }}"
            clients: "{{ inventory_hostname in groups['nfs_clients'] }}"
          nfs_server: "{{ hostvars['nfs_server']['ansible_host'] }}"
          nfs_export_clients: "{{ groups['nfs_clients'] | map('extract', hostvars, 'ansible_host') | join(' ') }}"

Author Information
------------------

- Holly Silk (<holly@stackhpc.com>)
