# Nextcloud FPM Alpine

The target release is `nextcloud:35.0.2-fpm-alpine`. The `NC_VERSION` stack
variable in Portainer overrides the Compose and Dockerfile defaults, so update
that variable when selecting a release. PostgreSQL stays on major version 15.

## Major upgrades

Follow the [official upgrade rules](https://docs.nextcloud.com/server/stable/admin_manual/maintenance/upgrade.html)
and review the [critical changes](https://docs.nextcloud.com/server/stable/admin_manual/release_notes/index.html).

Do not skip major releases. First reach the latest maintenance release of the
current major version, then upgrade to the latest maintenance release of the
next major. The October 2026 path from this installation is:

`31.0.14 -> 32.0.15 -> 33.0.10 -> 34.0.5 -> 35.0.2`

This deployment's owner requires database-only backups to preserve server disk
space. Before every transition, stop web traffic and application writers, create
a compressed custom-format PostgreSQL dump, verify it with `pg_restore --list`,
and verify its checksum after copying it off the database container. Do not copy
the Nextcloud files, Redis or other side services. Restore the matching database
and matching application image together if recovering a previous release.

Review third-party app compatibility and disable those apps before upgrading.
After each transition, update and restore compatible
apps, confirm `occ status` reports no pending database upgrade, and run
`php -f /var/www/html/cron.php` two or three times as `www-data` to finish
background migrations before proceeding. Keep web traffic and the normal cron
container stopped while preparing the next transition.

`files_downloadactivity` is archived upstream and has no compatible release for
Nextcloud 34 or 35. Leave it disabled instead of forcing compatibility. When
updating a disabled Group folders app, its downloaded code may require a separate
`occ app:enable groupfolders` to complete the app migration; rerun the app update
check afterward.

## SMB support

The image includes both the Samba `smbclient` command and the PHP `smbclient`
extension. The extension is compiled from the official upstream GitHub release
because the PECL download host was unreachable from the deployment server.
Downloads use TLS verification, retries and a pinned SHA-256 checksum.

`SMBCLIENT_VERSION` and `SMBCLIENT_SHA256` are build arguments; update them
together when selecting another extension release. The build verifies that
`smbclient` and `bz2` load after removing compiler dependencies.

Check `php --ri smbclient` and `php occ files_external:dependencies` after
deployment. The existing `wg_nw` network and route to the SMB subnet are retained.
