- Feature Name: MicroOS Management
- Start Date: (fill with today's date, YYYY-MM-DD)

# Summary
[summary]: #summary

Improve the management of openSUSE MicroOS and similar systems (e.g. openSUSE Leap Micro and SUSE Linux Micro) in Uyuni.

# Overview

4. UI and API updates to give control over custom states
5. Rebooting transactional systems
   - Automatic reboot during bootstrap via UI
6. Fixes to service.enabled / service.disabled
7. Out of scope: Salt Formulas, Multiple OS states in DB

# Detailed design
[design]: #detailed-design

## Uyuni does not use `transactional_update` executor anymore
The smallest unit Salt can handle is the SLS file. To control which SLS files are applied
in a transaction or not, Uyuni stops relying on the `transactional_update` executor. Instead, Uyuni
either calls `state.apply $list_of_sls_files` or `transactional_update.apply $list_of_sls_files`.

All internal states that interact with the live system are applied with `state.apply.`
Internal states that change the operating system, e.g. by installing packages, are applied
with `transactional_update.apply`. Today, many of our SLS files combine installing packages
and making use of them directly. That does not work on transactional systems, those SLS
files need to be split.

At a later time, users are given the choice for their custom states on a per-SLS basis.
Since that requires quite a bit of work on the database schema, UI and API, we go for an
all-or-nothing approach first. We add a new config value:
`java.salt_custom_states_use_transactional_update = True`. This value defaults to the same
behaviour as today, in order to allow for backwards-compatibility for existing SLS files.

### Internal States Filesystem Structure
Up to now, we bundle prerequisites (e.g. package installations) with the main part in SLS
files. Since that does not work on transactional, we're now using the following structure:

``` text
hardware/
        prereq.sls
        profileupdate.sls
ansible/
        prereq.sls
        runplaybook.sls
```

### Internal States (`state.apply`)
- `actionachains.{startssh,resumessh}`
- `ansible.runplaybook`
- `cocoattest.requestdata`
- `hardware.profileupdate`
- `images.*`
- `packages.profileupdate`
- `packages.redhatproductinfo`
- `srvmonitoring.status`
- `util.sync*`
- `util.systeminfo_full`
- `util.systeminfo`

### Internal States (`transactional_update.apply`)
- `ansible` - rename to `ansible.prereq`
- `certs`
- `channels`
- `cleanup_minion`
- `cleanup_ssh_minion`
- `configuration.deploy_files` NOTE: reconfiguring a service through `/etc` is special as there is an overlayfs.
- `distupgrade`
- `packages.patch*`
- `packages.pkg*`
- `packages`
- `reboot`
- `services.docker`
- `services.kiwi-image-server`
- `services.reportdb-user`
- `services.salt-minion` REVIEW: Is all of this still needed? NOTE: `file.managed` in `/etc`
- `srvmonitoring.disable`
- `srvmonitoring.enable`
- `switch_to_bundle`
- `update-salt`
- `uptodate`
- `util.disable_fqdns_grain`: NOTE: configures in `/etc`, restarts a service (currently broken)
- `util.mgr_mine_config_clean_up`: NOTE: configures in `/etc`, restarts a service (currently broken)
- `util.mgr_rotate_saltssh_key`
- `util.mgr_start_event_grains` NOTE: configures in `/etc`
- `util.mgr_switch_to_venv_minion`

### Unsupported Internal States
- `rebootifneeded` - The way this is written is incompatible with transactional systems
- `appstreams.configure` - only useful for RHEL systems
- REVIEW `bootstrap.autoinstall` - Uyuni does not know that it targets a transactional system but
  this state only works with `transactional_update.apply`

### Configurable States (`java.salt_custom_states_use_transactional_update`)

-   `custom`
-   `custom_groups`
-   `custom_org`
-   `recurring`
-   `remotecommands`

#### Required Changes

- Extract installation steps in `cocoattest` to a `cocoattest.prereq`
- Extract `dmidecode` installation steps in `hardware.profileupdate` to `hardware.prereq`

### TODO

- `scap` NOTE: `remediate=True` is likely OS-altering

## Automatic reboots during bootstrapping

When bootstrapping a new system, Uyuni relies on information present on the client system
to know what kind of system it is. This includes finding out if the new system is a
transactional system. Bootstrapping happens with Salt SSH and `state.apply`. The bootstrap
SLS file contains logic to install our Salt Minion package correctly on both
traditionally-managed and transactional systems.

The bootstrap SLS file installs the Salt Minion package into the next snapshot. We need to reboot the Minion after installing this package.

### Add Inhibitor Lock to Salt SSH

Applications can set _inhibitor locks_ to block or delay system shutdown and sleep states.
Salt SSH sets a _delay_ inhibitor lock to stop the system from rebooting immediately. Salt
SSH has time to return job results back to the Salt Master, unless it takes longer than
_InhibitDelayMaxSecs_. This config setting is specified in `logind.conf(5)` and can't be
overridden by Salt SSH. The default is 5 seconds.

### Request a reboot without delay

With a delay lock taken, `bootstrap/init.sls` can request a reboot from systemd from the
main process. The reboot will be delayed until Salt SSH execution terminates and releases
the lock.

# Bug Fixes

## Make state functions available for transactional systems

-   `service.enabled`: currently needs dbus, we need a way that does not require dbus for enabling the service
-   `service.disabled` currently needs dbus, we need a way that does not require dbus for enabling the service


# Drawbacks
[drawbacks]: #drawbacks

Why should we **not** do this?
* More work for us maintaining SLS files with the new layout, as we need to think were to put different states
* 

# Alternatives
[alternatives]: #alternatives

- What other designs/options have been considered?
- What is the impact of not doing this?

# Unresolved questions
[unresolved]: #unresolved-questions

- What are the unknowns?
- What can happen if Murphy's law holds true?
