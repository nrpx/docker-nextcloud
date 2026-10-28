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
WireGuard owns `192.168.36.1`; FPM uses `192.168.36.2` (overridable with
`NC_APP_WG_IP`). This prevents Docker from assigning the gateway's address to
Nextcloud while the WireGuard container is stopped.

## Talk HPB memory

Keep the `talk-hpb` service's `nofile` limits at soft `65536` and hard `1048576`.
With a huge inherited host limit, Erlang can preallocate gigabytes for its port
table even without TURN sessions; see [eturnal issue 97](https://github.com/processone/eturnal/issues/97).
Recreate the service after changing the limits, then check `eturnalctl info`
and the container's memory usage.

## PHP-FPM pool and stalled requests

The image bakes in `zz-nextcloud-pool.conf`; no init containers or host-side
configuration preparation are needed. Its dynamic pool allows up to 8 workers,
starts 3, keeps 2–4 idle, and recycles each worker after 500 requests. These are
conservative initial values for the current 2 GiB host shared with other services;
measure worker RSS and available host RAM before raising them.

A 300-second FPM wall-clock timeout reclaims stalled requests, including work
after `fastcgi_finish_request()`. On Linux PHP's `max_execution_time` alone does
not bound time waiting for I/O. The timeout also bounds legitimate single HTTP
transfers or operations; use Nextcloud's chunked uploads for large files and
review the limit if a valid operation requires more than five minutes.

Access entries include the routed request path (without query parameters),
duration and peak PHP memory in the existing size-limited container log.
FPM slow traces remain disabled: a runtime probe confirmed that the container's
current capabilities prohibit ptrace. No additional tracing capability is needed
for the access log and independent status listener.
FPM status uses an independent listener at `127.0.0.1:9001` inside the application
container, with path `/fpm-status`; inspect it through Docker exec/FastCGI.
Do not publish the listener or add a public Nginx route for it.

The October 28 incident exhausted the default five-worker pool and left Nginx
returning upstream timeouts until the application container was restarted.
Nextcloud 35.0.2 reports no pending database upgrade and passes core integrity.
The retained application log also contains remote WebDAV failures from
`files_lock`; historical logs do not identify the precise blocked call because
the prior pool had neither a slow log nor an independent status listener.
