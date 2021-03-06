# IDR administration

If you used the OpenStack provisioning playbook, the only accessible external ports will be SSH, HTTP, HTTPS, rsync, and several OMERO ports on `idr-proxy`.
For all administrative OMERO operations you will first need to SSH into `idr-proxy` and/or set up a SSH tunnel for other ports and servers.

Any changes that do not involve manipulating the data stored in OMERO (such as changes to the OMERO.server and OMERO.web configuration) should be done by modifying the deployment playbooks to ensure they are included in future deployments.


## Production IDR

There are two ways to access the production IDR:
- Nginx web proxy running on `idr-proxy`
- Back-end OMERO.web and OMERO.server running on `idr-omeroreadonly*` and `idr-omeroreadwrite`


### `idr-proxy`

The IDR appears as a read-only resource to public users, so the Nginx proxy has a custom caching configuration that ignores any headers sent by the back-end OMERO.web.
This should only be used for public web access, otherwise private tokens or views may inadvertently be cached.

This means if the back-end OMERO data is modified the cache may become stale.
A special cache-buster port `9000` is configured on `idr-proxy`.
Any requests to this port will force a cache refresh.

For example:

    ssh idr-proxy -L 9000:localhost:9000

And open http://localhost:9000.
Due to the way OMERO.web redirects you may be redirected to a host without the port, in which case add the port back.

In the case of large screens this may lead to a significant slowdown of the server, though it does also show the importance of the cache.

You can force a complete refresh of the cache by removing everything under `/var/cache/nginx/`.

**WARNING**: Never log in to OMERO.web on any idr-proxy port to prevent incorrect state being cached.

The read-only servers have a database statement timeout to prevent big queries from breaking the server.
If you need to cache a large response that exceeds this timeout you can use port `8009` on `idr-proxy`.


### `idr-omeroreadonly*` `idr-omeroreadwrite`
If you SSH into these servers you will have full access to OMERO.server, including OMERO.web and the command-line client.
If you need to login to OMERO you should always connect to these servers.

If you need to restart OMERO.server or OMERO.web, you must use `systemctl` and not `omero admin` or `omero web`.

The number of `idr-omeroreadonly*` servers may vary, check `/etc/hosts` oni `idr-proxy` for a full list of internal servers.

### Bio-Formats cache regeneration

Every time a new version of the software that invalidates the Bio-Formats
cache files is deployed, it might be necessary to regenerate these files
ahead of time.

Cache files should be regenerated for each imported fileset. The following SQL 
script will return a list of all the IDs for the first Image in each Fileset:

    $ echo "copy (select r.i from (select f.id, i.id as i, rank()
                  over (partition by f.id order by i.id) as x
                  from fileset f, image i where f.id = i.fileset)
                  as r where r.x = 1) to stdout;" > first_image_per_fileset.sql

For large databases, cache regeneration should be run in a parallelized manner
across the available OMERO servers (read-write and read-only):

    $ grep -oE 'omero[^ ]+$' /etc/hosts > nodes

To prevent `PermissionDenied`, create a SSH key and copy it across nodes:

    $ ssh-keygen
    $ for i in $(cat nodes); do ssh-copy-id -i ~/.ssh/id_rsa $i; done

Bio-Formats cache regeneration makes use of the
[OMERO.CLI render plugin](https://pypi.org/project/omero-cli-render/). First
of all, make sure the plugin is installed on each OMERO server:

    $ for i in $(cat nodes); do ssh $i 'pip install --user omero-cli-render'; done

The existing Bio-Formats cache can be moved from the read-write server as:

    $ ssh omeroreadwrite sudo mkdir /data/BioFormatsCache/backup
    $ ssh omeroreadwrite sudo mv /data/BioFormatsCache/* /data/BioFormatsCache/backup

Then execute the cache file regeneration using GNU parallel from the proxy
servers across all nodes using the `ids.txt` list generated by the SQL script:

    $ parallel --sshloginfile nodes -a ids.txt --results out.csv -j10 '/opt/omero/server/OMERO.server/bin/omero render -s localhost -u public -w public test --force'


## Analysis IDR

The VAE (JupyterHub) is managed separately. See https://github.com/IDR/k8s-analysis-deploy.


## Backups, restores and upgrades

The IDR has a well-defined separation between applications and data.
The following directories contain data that must be backed up:
- `idr-omeroreadwrite:/data`: The OMERO data directory
- `idr-database:/var/lib/pgsql`: The PostgreSQL data directory

The following directories are not essential but you may wish to also back them up:
- `idr-proxy:/var/cache/nginx`: The front-end web cache (can be regenerated)

If you used the OpenStack provisioning playbook, these are all separate volumes that can be backed up using the OpenStack clients.

### Restoration
If you need to restore the IDR, it is sufficient to restore these directories into a clean CentOS 7 server before running the deployment playbooks, which will take the existing data into account when installing the IDR.
The OpenStack provisioning playbook includes optional parameters to specify existing volumes to be copied.

### New releases
The public-facing IDR is read-only and we aim to minimize any downtime, so instead of upgrading the existing system the procedure for new releases involves cloning the existing server volumes into a new system:

- Ensure you have an unused floating IP.
- Run the OpenStack provisioning playbook with an alternative IDR environment parameter (`idr_environment_idr=newidr`) and set the existing volumes as the source for the new volumes.
- When you are ready to go live disassociate the floating IPs from the production and new deployments, and associate the previous production floating IP with the new deployment.

### Planned production maintenance
Changes to the live production server must be avoided where possible.
If it is necessary to stop OMERO server or OMERO web, for isntance due to maintenance by the cloud provider, a maintenance page can be displayed for all OMERO.web requests.
On idr-proxy create this flag file:

    $ touch /srv/www/omero-maintenance.flag

When maintenance is complete:

    $ rm /srv/www/omero-maintenance.flag
