# volume-backup

An utility to backup and restore [docker volumes](https://docs.docker.com/engine/reference/commandline/volume/). 

**Note**: Make sure no container is using the volume before backup or restore, otherwise your data might be damaged. See [Miscellaneous](#miscellaneous) for instructions.

**Note**: When using docker-compose, make sure to backup and restore volume labels. See [Miscellaneous](#miscellaneous) for more information.

## Credits

Original code by [loomchild](https://github.com/loomchild/volume-backup)

For more info, read his article on [Medium](https://medium.com/@jareklipski/backup-restore-docker-named-volumes-350397b8e362)

## Backup

### Backup to standard output

This avoids mounting a second backup volume and allows to redirect it to a file, network, etc.

Syntax:

    docker run -v [volume-name]:/volume --rm --log-driver none mcicolella/docker-volumes-backup backup - > [archive-name]

For example:

    docker run -v some_volume:/volume --rm --log-driver none mcicolella/docker-volumes-backup backup - > some_archive.tar.bz2

will archive volume named `some_volume` to `some_archive.tar.bz2` archive file.

**Note**: `--log-driver none` option is necessary to avoid storing an entire backup in a temporary stdout JSON file. More info in [Docker logging documentation](https://docs.docker.com/config/containers/logging/configure/) and in [this issue](https://github.com/loomchild/volume-backup/issues/39).

**WARNING**: This method should not be used under PowerShell on Windows as no usable backup will be generated.

### Backup to a file

Syntax:

    docker run -v [volume-name]:/volume -v [output-dir]:/backup --rm mcicolella/docker-volumes-backup backup [archive-name]

For example:

    docker run -v some_volume:/volume -v /tmp:/backup --rm mcicolella/docker-volumes-backup backup some_archive

will archive volume named `some_volume` to `/tmp/some_archive.tar.bz2` archive file.

## Restore

Restore will fail if the target volume is not empty (use `-f` flag to override).

### Restore from standard input

This avoids mounting a second backup volume.

**Note**: Don't forget the `-i` switch for interactive operation.

Syntax:

    cat [archive-name] | docker run -i -v [volume-name]:/volume --rm mcicolella/docker-volumes-backup restore -

For example:

    cat some_archive.tar.bz2 | docker run -i -v some_volume:/volume --rm mcicolella/docker-volumes-backup restore -

will clean and restore volume named `some_volume` from `some_archive.tar.bz2` archive file.

### Restore from a file

Syntax:

    docker run -v [volume-name]:/volume -v [output-dir]:/backup --rm mcicolella/docker-volumes-backup restore [archive-name]

For example:

    docker run -v some_volume:/volume -v /tmp:/backup --rm mcicolella/docker-volumes-backup restore some_archive

will clean and restore volume named `some_volume` from `/tmp/some_archive.tar.bz2` archive file.

### Copy volume between hosts

One good example of how you can use the output to stdout would be directly migrating the volume to a new host

Syntax:

    docker run -v [volume-name]:/volume --rm --log-driver none mcicolella/docker-volumes-backup backup - |\
         ssh [receiver] docker run -i -v [volume-name]:/volume --rm mcicolella/docker-volumes-backup restore -

**Note**: In case there are no traffic limitations between the hosts you can trade CPU time for bandwidth by turning off compression as shown in the example below.

For example:

    docker run -v some_volume:/volume --rm --log-driver none mcicolella/docker-volumes-backup -c none - |\
         ssh user@new.machine docker run -i -v some_volume:/volume --rm mcicolella/docker-volumes-backup restore -c none -
    
## Miscellaneous

1. Upgrade / update volume-backup
    ```
    docker pull mcicolella/docker-volumes-backup
    ```

1. volume-backup is also available from GitHub Container Registry (ghcr.io), to avoid DockerHub usage limits:
    ```
    docker pull ghcr.io/loomchild/volume-backup
    ```
    **Note**: you'll need to write `ghcr.io/loomchild/volume-backup` instead of just `loomchild/volume-backup` when running the utility.

1. Find all containers using a volume (to stop them before backing-up)
    ```
    docker ps -a --filter volume=[volume-name]
    ```

1. Exclude some files from the backup and send the archive to stdout
    ```
    docker run -v [volume-name]:/volume --rm --log-driver none mcicolella/docker-volumes-backup -e [excluded-glob] - > [archive-name]
    ```

1. Use different compression algorithm for better performance
    ```
    docker run -v [volume-name]:/volume --rm --log-driver none mcicolella/docker-volumes-backup backup -c pigz - > [archive-name]
    ```
1. Show simple progress indicator using verbose `-v` flag (works both for backup and restore)
    ```
    docker run -v [volume-name]:/volume --rm --log-driver none mcicolella/docker-volumes-backup backup -v - > [archive-name]
    ```
1. Pass additional arguments to the Tar utility using `-x` option
    ```
    docker run -v [volume-name]:/volume --rm --log-driver none mcicolella/docker-volumes-backup backup -x --verbose - > [archive-name]
    ```
1. Volume labels are not backed-up or restored automatically, but they might be required for your application to work (e.g. when using docker-compose). If you need to preserve them, create a label backup file as follows: `docker inspect [volume-name] -f "{{json .Labels}}" > labels.json`. When restoring your data, target volume needs to be created manually with labels before launching the restore script: `docker volume create --label "label1" --label "label2" [volume-name]`.
