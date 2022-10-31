# How to back up and restore your Weaviate data with ease

## Introduction
Maintaining data integrity is one of the key goals for database users. So it should come as no surprise that backing up the data is an important part of database best practices for many.

Although it has been possible to back up Weaviate data, in the past doing so required our users to perform many manual and inelegant steps. So, we have introduced a [backup feature](https://weaviate.io/developers/weaviate/current/configuration/backups.html) in Weaviate v1.15 that streamlines the back up process, whether it be to a local file system or to a cloud storage provider.

This tutorial will show you how you can use this feature to back up your data, and restore it to another instance of Weaviate. By the end of this tutorial, you will have:
- Spun up two instances of Weaviate,
- Populated one Weaviate instance with a new schema & data,
- Backed up the Weaviate instance, and
- Restored the backup data to the other instance.


## Preparations
> For beginners, we recommend checking out our [Getting Started guide](https://weaviate.io/developers/weaviate/current/getting-started/index.html), which should take just a short while to get through. :)

To get started, clone [this repository](https://github.com/weaviate-tutorials/weaviate-backup) then run `docker-compose up -d` to spin up Weaviate. The Docker configuration file (`docker-compose.yaml`) has been set up to spin up two Weaviate instances for this tutorial. You should be able to connect to them at `http://localhost:8080` and `http://localhost:8090` respectively. We’ll call them W1 and W2 from this point on for convenience.

The `docker-compose.yaml` also specifies the below parameters to enable local backups.
```
environment:
  …
  ENABLE_MODULES: 'backup-filesystem'
  BACKUP_FILESYSTEM_PATH: '/tmp/backups'
volumes:
  - ./backups:/tmp/backups
```

This enables the `backup-filesystem` module to back up data from Weaviate to the filesystem, and sets `/tmp/backups` as the `BACKUP_FILESYSTEM_PATH` which is the backup path within the Docker container. 

The `volumes` parameter below [mounts a volume](https://weaviate.io/developers/weaviate/current/configuration/persistence.html) from outside the container to within it. Setting it to `./backups:/tmp/backups` maps `./backups` on the local device to `/tmp/backups` within the container, so the generated backup data will end up in the `./backups` directory as you will see later on. Now let’s dive into it to see the backup functionality in action!

## Populate one instance
Both of our Weaviate instances W1 and W2 should be empty. Run `scripts/0_query_instances.sh` and you should see that neither instances contain a schema. (If not, run `scripts/0_delete_all.sh`.)

As our first order of action we will populate W1 with data to play with. Run `scripts/1_create_schema.sh` to create a schema, and `scripts/2_import.sh` to import data. 

Now, running `scripts/0_query_instances.sh` will return results showing the `Author` and `Book` classes in the schema as well as the objects.

## Back up & restore data
Now let’s move on to creating our first backup. First, check that our backups directory (`./backups`) is empty, and delete any contents if this is not the case. 

Initiating a backup involves just a short bit of code. The below will for example back up all classes in W1 using an ID called `my-first-backup`.
```bash
curl \
-X POST \
-H "Content-Type: application/json" \
-d '{
     "id": "my-very-first-backup"
    }' \
http://localhost:8080/v1/backups/filesystem
```
> Note: The `backup_id` must be unique. Attempting to re-use an existing ID will cause Weaviate to throw an exception.

Now try running `scripts/3_backup.sh` yourself to back up data from W1. If you check the contents of the backup directory again, you should see a new directory called `my-first-backup` containing the backup data files.

Restoring this data is similar. The command below will restore our backup:
```bash
curl \
-X POST \
-H "Content-Type: application/json" \
-d '{
     "id": "my-very-first-backup"
    }' \
http://localhost:8090/v1/backups/filesystem/my-very-first-backup/restore
```
Run `scripts/4_restore.sh` to restore the W1 backup data to W2. Now, check the schemas again for W1 and W2. Do they *both* now contain the same schema? What about the data objects? They should be identical.   

Great. You have successfully backed up data from W1 and restored it onto W2! 

## Backup features
Before we finish, let’s go back to talk a little more about additional backup options, and some important notes.

### Local backup location
If you wish to back up your data to a different location, you can edit the `volumes` parameter in `docker-compose.yaml` to replace `./backups` with the desired location.

For example, changing it from `./backups:/tmp/backups` to `./my_archive:/tmp/backups` would change the backup destination from `./backups` to `./my_archive/`.

### Cloud storage systems
Note, you can also configure Weaviate backup to work with **cloud storage systems** like **Google Cloud Storage** (**GCS**) and **S3-compatible blob storage** (like **AWS S3** or **MinIO**).

Each requires a different Weaviate backup module (`backups-gcs` and `backups-s3`), configuration parameters and authentication. Check the documentation to learn more about [GCS backup](https://weaviate.io/developers/weaviate/current/configuration/backups.html#gcs-google-cloud-storage) and [S3 backup](https://weaviate.io/developers/weaviate/current/configuration/backups.html#s3-aws-or-s3-compatible).

### Partial backup and restores
Weaviate’s backup functionality allows you to back up or restore a subset of available classes. This might be very useful in cases where, for example, you may wish to partially export a subset of data to a development environment, or to import an updated class.

For example, the below call will restore only the `Author` class regardless of whether any other classes have been also included in `my-first-backup`.
```bash
curl \
-X POST \
-H "Content-Type: application/json" \
-d '{
     "id": "my-very-first-backup",
     "include": ["Author"]
    }' \
http://localhost:8090/v1/backups/filesystem/my-very-first-backup/restore
```
Delete everything in W2 first with with `scripts/9_delete_w2.sh`, and try out the partial restore with `scripts/4a_partial_restore.sh`. You should see that W2 will only contain one class even though its data was restored from a backup that contains multiple classes.

The restore function allows you to restore a class as long as the target Weaviate instance does not already contain that class. So if you run another operation to restore the `Book` class to W2, it will result in an instance containing both `Author` and `Book` classes.

### Asynchronous operations
Did your returned messages from `backup` or `restore` requests include `"status":"STARTED"`? Isn't it interesting that the status was not indicative of a completion?

That is because Weaviate’s backup operation can be initiated and monitored asynchronously. 

You don’t need to maintain a connection to the server for the operation to complete, and you can look in on the status with a method such as below:
```bash
curl http://localhost:8090/v1/backups/filesystem/my-very-first-backup/restore
```

Weaviate remains available for read and write operations, while backup operations are ongoing. And you can poll the endpoint to check its status, without worrying about any potential downtime.
 
## Wrap-up
We hope that this gave you a good overview of the new backup feature available in Weaviate. More importantly, we think that it will make it easier and faster for you to back up your data and help make your application more robust. 

To recap, it allows you to back up one or more classes from an instance of Weaviate to a backup, and to restore one or more classes from a backup to Weaviate. Weaviate remains functional during these processes, and you can poll the backup or restore operation status periodically if you wish. 

Weaviate currently supports backing up to your local filesystem, AWS or GCS. But as the backup orchestration is decoupled from the remote backup storage backends, it is relatively easy to add new providers and use them with the existing backup API.

If you would like to use another storage provider to use with Weaviate, we encourage you to open a feature request or consider contributing it yourself. For either option, join our Slack community to have a quick chat with us on how to get started.