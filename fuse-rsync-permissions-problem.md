# Rsyncing an fuse-mounted dir - permissions problem

## Symptom
On trying to `rsync` a fuse-mounted dir as `root`, we get an error:

```
sync: [sender] link_stat "path/to/dir" failed: Permission denied (13)
```

This is e.g. the case with apps using `fuse`, like Cryptomator.

## Cause
`fuse` does not allow root access by default. See e.g. [here](https://community.cryptomator.org/t/cryptomator-vault-folders-files-sudo-permissions-denial/13091/3).

## Fix

Use `rsync` as the owner user on source.

When using Ansible module `ansible.posix.synchronize`, use `become: false` for the task which `rsync`s the `fuse`-mounted dir.

Alternatively, add custom flags on mounting (see e.g. [Cryptomator docs](https://docs.cryptomator.org/desktop/volume-type/#desktop-volume-type-fuse-fuse)).

## Notes

[Docs for `ansible.posix.synchronize`](https://docs.ansible.com/projects/ansible/latest/collections/ansible/posix/synchronize_module.html) say:

> The user and permissions for the synchronize `src` are those of the user running the Ansible task on the local host (or the remote_user for a `delegate_to` host when `delegate_to` is used).

However, this seems not to work like that here: `rsync` with this module in Ansible works when setting `become` is false, and does not work otherwise (which it should according to the quote).

## Links
- https://community.cryptomator.org/t/cryptomator-vault-folders-files-sudo-permissions-denial/13091/3
- https://docs.ansible.com/projects/ansible/latest/collections/ansible/posix/synchronize_module.html
