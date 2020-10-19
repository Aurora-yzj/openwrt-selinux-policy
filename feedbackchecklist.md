# Feedback Check List

These are some area's to check for anomalies:

* Context `u:r:unlabeled` and `u:r:invalid` should never be present
* The should be no *character* and *block* files in `/dev/` with context
`u:r:tmp.file`
* With the exception of `procd` (pid1), `crond`, `ntpd`,
`kernel threads`, and your session there should be no processes with
context `u:r:sys.subj`
* The command `dmesg | grep -i selinux` should not print any warnings
* The command `dmesg | grep -i denied` should not print any avc denials
