
Stage Sync
==========

An Ansible role to sync production and staging. Web projects such as Django, Ruby on Rails, and others may have a staging website and a production website, each using their own separate database and S3 buckets.  

Staging and Dev environments could benefit from having a copy of the user accounts, blog posts, content, and images that exist in production.

This does not deploy "the code", but rather it synchronizes other supporting background assets: specifically the database and S3 buckets. 

V1 of the role currently supports Postgres and S3.

Instructions
------------

Add servers to the inventory (/etc/ansible/hosts) as follows:  

```
[stagesync]
db1.example.com
staging-db1.example.com
localhost

[stagesync-sourcedb]
db1.example.com

[stagesync-destdb]
staging-db1.example.com
```

In other words, place all relevant machines in the stagesync group. Also add the source and destination db's to their own groups. Use those group names. Modify the server names.  

Create a group_vars file `/etc/ansible/group_vars/stagesync/main.yml` and copy in this role's `defaults/main.yml` to begin with. Modify as necessary.  

That would be sufficient if you usually sync between one pair of locations. Production -> Staging. However, what happens if there are multiple sets of pairs: Production -> Stage1, Production -> Stage2, etc. In that case, in addition to `group_vars/stagesync/main.yml` (which can be used for most general settings), also create the following files. Example:    

```
/etc/ansible/vars/prod-to-stage1.yml
/etc/ansible/vars/prod-to-stage2.yml
/etc/ansible/vars/stage1-to-dev.yml
```

In each of those files, place more customized variables that would only apply to a particular sync pair. Example:    

```
stagesync_sourcedb: "production"
stagesync_destdb: "stage"
stagesync_destdb_role: "stage"

stagesync_copies:
  - description: production to stage media bucket
    source_bucket: production_bucket
    source_path: "files/"
    dest_bucket: staging_bucket
    dest_path: "files/"
    aws_profile: stage

stagesync_social_client_id: "123"
stagesync_social_secret: "456"
```

Then run the playbook with --extra-vars:  

```
ansible-playbook --extra-vars "@vars/prod-to-stage.yml" roles/cppalliance.stagesync/playbooks/stagesync-playbook.yml
```

Notice the variable `stagesync_copies:` above. That is important to configure properly. It's a list of source and destination S3 buckets. Multiple sets of buckets. For each item, set the "source_bucket" and "dest_bucket" that should be copied, and so on.  

Set `aws_profile` to a profile (with a certain id and key from the credentials file) that has at least read access on the source bucket, and write access to the destination bucket. Profiles are discussed more later.  

Usually the 'source' is 'production' and the 'destination' is 'staging' since you are copying assets in that direction. 'Production' has real user accounts, and real articles, that are being populated into a testing environment.

Database credentials: an assumption is made that postgres backups will run as the db superuser (postgres) on the target machines, which conveniently remove the requirement to provide credentials to ansible.  

S3 credentials: storing passwords and secrets always adds complexity to a system. Let's try something different, instead of the usual tactic of storing everything within Ansible itself. S3 credentials should be placed in `${HOME}/.aws/credentials` where both rclone and awscli are able to discover and use those credentials. Create a file `${HOME}/.aws/credentials` such as:  

```
[production]
aws_access_key_id = AKIABCDE
aws_secret_access_key = ____

[stage]
aws_access_key_id = AKIAZYXW
aws_secret_access_key = ____
```

The profiles "stage" and "production" are used in the stagesync_copies variable.  

Backup Scripts
--------------

While database dumps are an important part of this role, they are not run directly by an ansible task. Rather a backup shell script is (optionally) generated, and that's invoked. You may also use your own script, in which case set the `stagesync_backups_create_backups_scripts` variable to false. In order to be compatible with this role, the command-line arguments of the backup script should be the names of the databases to backup.  

The (optional) backup script provided by this role has the capability to backup all databases, not merely the one being synced, and upload those to S3. That is more than is required. For the purposes of stagesync all the script really needs to do is generate a dump file locally. Then the ansible role copies the file from the source to the destination, and re-imports it. Review the script and variables to see how it might also upload to S3 (which is out of scope for this role, but in-scope for daily/weekly scheduled backups). 

Dependencies
------------

Depends on awscli, gsutil, or other S3 type cli commands already being present.  

Example Playbook
----------------

See the playbook in playbooks/stagesync-playbook.yml. Example usage:  

```
ansible-playbook --extra-vars "@vars/prod-to-stage.yml" roles/cppalliance.stagesync/playbooks/stagesync-playbook.yml
```

License
-------

Boost Software License, Version 1.0.

Contributing
------------

Bug fixes are welcome. Also, features and pull requests that have a significant chance of being widely used.  

