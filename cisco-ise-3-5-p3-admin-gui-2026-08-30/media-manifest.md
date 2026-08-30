# Cisco ISE reproduction media manifest

Archived: 2026-08-30  
Canonical directory: `/mnt/home/labs/iso/cisco-ise/3.5.0.527/`  
Filesystem: persistent lab media area on `/home`  
Mode: read-only (`0444`)

Redundancy status: **one local canonical copy only**. It is organized and
protected from accidental modification, but it is on the same physical storage
as the original Downloads directory. No Nextcloud/off-host copy has been made;
uploading proprietary Cisco media requires explicit authorization.

| Purpose | Filename | Bytes | SHA-256 |
|---|---|---:|---|
| ISE 3.5 base installer | `Cisco-ISE-3.5.0.527.SPA.x86_64.iso` | 14,838,290,432 | `74219e36d55533c2b1bd380e541043aad9e7ec11d535d5d27a419b34c0fc97b1` |
| ISE 3.5 Patch 3 | `ise-patchbundle-3.5.0.527-Patch3-26040703.SPA.x86_64.tar.gz` | 1,817,306,968 | `3b895a799ca1b165fe1604468b397c1d370a77d784569e9c0bef8102de2ee143` |

These SHA-256 values establish the identity of the files archived for this
experiment. They have not yet been compared with a Cisco-published checksum or
validated through a Cisco signature-verification procedure. That is a separate
authenticity gate before installation.

The media files are deliberately outside Git. Do not copy them into this
repository; use this manifest to locate and verify them.
