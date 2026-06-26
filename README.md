# Entire SSH wrapper

This repository contains dummy content for testing.

## `entire-ssh`

`entire-ssh` is a small wrapper around `ssh` intended for use as Git's SSH
command:

```sh
git config core.sshCommand "$PWD/entire-ssh"
```

The wrapper runs `ssh` with OpenSSH connection multiplexing enabled:

- `ControlMaster=auto` reuses an existing SSH control connection when one is
  available, or creates one when it is not.
- `ControlPath=$XDG_RUNTIME_DIR/entire-ssh-$uid/%C` stores the per-user control
  socket in a runtime directory. `%C` lets OpenSSH derive a safe unique socket
  name for each destination.
- `ControlPersist=120s` keeps the master connection alive briefly after the
  first command exits, so follow-up Git commands can reuse it.

This helps with the pre-push hook that also pushes the entire checkpoint branch.
During a normal `git push`, Git invokes the pre-push hook before completing the
push. If that hook starts another `git push` to publish the checkpoint branch,
the nested push would normally need to open and authenticate a second SSH
connection. With `entire-ssh`, both pushes can share the same OpenSSH control
connection, which avoids repeated authentication prompts and makes the hook less
likely to fail because SSH is waiting for another credential or interactive
confirmation.

## Limitations

- This probably only works with OpenSSH. The options used by `entire-ssh`
  (`ControlMaster`, `ControlPath`, `ControlPersist`, and `%C`) are OpenSSH
  features, so other SSH clients may ignore them or fail with an unsupported option
  error. It also depends on being able to create a private control socket directory
  under `$XDG_RUNTIME_DIR` or `/tmp`.
- Potential race-condition: if the `git push` from `entire`'s hook runs too early,
  the SSH connection might not be ready yet. This is probably solvable by checking
  whether the connection is ready to be used before using `git push`.
- Reused connections inherit the master connection's authentication and SSH
  session settings for the same user, host, and port. If two Git operations need
  to reach the same destination with different identities, certificates, agent
  configuration, forwarding settings, or other connection-level options, the
  later command may reuse the earlier master instead of applying its own
  settings.
- Multiplexed Git commands still depend on the server accepting additional
  sessions over the existing SSH connection. Servers with restrictive `MaxSessions`
  settings, forced-command setups, or unusual Git hosting policies may reject
  additional sessions even though the local control socket exists.
- The master connection can outlive the Git command for up to `ControlPersist`.
  That is intentional, but it means a broken network connection or abruptly
  killed process can leave a stale control socket until OpenSSH notices and
  reconnects or the runtime directory is cleaned up.
