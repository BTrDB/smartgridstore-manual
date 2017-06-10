# Admin console

Once your cluster is up and running, you can log into the admin console
using

```bash
ssh admin@a.cluster.node -p 2222
```
Depending on your DNS settings name resolution might fail and the admin console can be accessed by checking the IP address that the service is running on.  You can access that information by using `kubectl`

```kubectl get services --all-namespaces```

Use the external IP address displayed from the results returned by kubectl.

```ssh admin@XXX.XXX.XXX.XXX -p 2222```


The default username is `admin` and the password is `sgs-default-admin-password`.

When you log in you should see a screen like:

```
                              _____   ______   ______                            
                             / ____| /  ____| /  ____|                           
                            | (___   | |  __ | (____                             
                             \___ \  | | |_ | \____ \                            
                             ____) | | |__| |  ____) |                           
                            |_____/  \______| |_____/                            

                          Smart Grid Store admin console                         
                       (c) 2017 Michael Andersen, Sam Kumar                      
                 (c) 2017 Regents of the University of California                
              ----------------------------------------------------  

>        
```

The console groups commands into categories, and you navigate it in a fashion similar
to a filesystem. To list the commands in the current module, use `ls` and to change module
you use `cd`. For example, to change the admin password use:

```
> cd acl
acl> cd users
acl/users> cd admin
acl/users/admin> passwd newpassword
password changed
```
The acl command is used to add users so they may access the admin console.

Top level menu:

The full command set available is:

```
mrplotter/ - configure Mr. Plotter
acl/       - manage users and permissions
manifest/  - configure registered phasor measurement units
btrdb/     - tune the BTrDB cluster
```
The following documentation is specifically developed for the deployment and ingestion of microPMU data to an operational v4 BTrDB with properly installed Kubernetes and Ceph clusters.

#### Visualization with Mr. Plotter

The mrplotter command is used to configure the permissions to the data in BTrDB as it appears in Mr. Plotter's navigation tree. After creating a user:password combination, we need to establish a metatag that will be applied to the data streams.  The metatag is a `group` that will let you segment different streams per user.

A default BTrDB installation will have exactly one metatag defined called `public`.  If you create a tagdef of "" and give it the name everything will have access to every stream being ingested into BTrDB once it has been applied to a user name.

You can see the default users by executing the command `lsusers` which will display the username and the tags they have been assigned.

To see the existing tags defined in the system execute the command `lstagdefs`

Commands:

```
mrplotter> ls
adduser     - creates a new user account
setpassword - sets a user's password
rmuser      - deletes user accounts
rmusers     - deletes all user accounts with a certain prefix
grant       - grants permission to view streams with given tags
revoke      - revokes tags from a user's permission list
showuser    - shows the tags granted to a user or users
lsusers     - shows the tags granted to all user accounts with a given prefix
deftag      - defines a new tag
undeftag    - deletes tag definitions
undeftags   - deletes tag definitions beginning with a certain prefix
addprefix   - adds a path prefix to a tag definition
rmprefix    - removes a path prefix from a tag definition
showtagdef  - lists the prefixes assigned to a tag
lstagdefs   - lists the prefixes assigned to all tags beginning with a given prefix
lsconf      - lists the path prefixes currently visible to each user
keys/       - Manage session and HTTPS keys
```

EXAMPLE HERE:


Manifest commands:

The manifest command is used to define the actual structure of the navigation tree as it applies to streams.

```
manifest> ls
add       - creates a new device with the provided descriptor
del       - deletes devices from the manifest
delprefix - deletes all devices with a certain prefix
setmeta   - set metadata
delmeta   - deletes metadata key-value pairs
lsdevs    - lists metadata for all devices with a given prefix
```

We will step through and example of setting up a stream for display in Mr. Plotter.  For the purposes of this example we're going to assume we have a microPMU identified as device p300001.  We need to add this device to our list of devices that we will ingest into BTrDB.  This entry assumes that you are using the binary transfer method from the microPMU to BTrDB ingester and not the C37.118 stream being sent from the microPMU. C37.118 is still in development and will be covered in a future update. The add command takes the following parameters.

```
add descriptor [key1=value1] [key2=value2] ...```

In our example we are going to identify the device as `descriptor` followed by the path that we want to assign as represented in the tree of Mr. Plotter.  All the descriptors should use the psl.pqube3. prefix to the microPMU ID.

```
add psl.pqube3.p300001 path=myPMU
```

As represented in Mr. Plotter the tree would show:
```
+myPMU -
       |
        - C1ANG
        - C1MAG
```
To group your devices you would add an additional element to the path parameter.

```
add psl.pqube3.p300001 path=myPMU/device1
```
*
It is IMPORTANT that you don't trail the command with a closing /
*

```
+myPMU -
       |  
        - device1
          |
           - C1ANG
           - C1MAG
```
Keep an eye on the ingester logs in Kubernetes for any problems if the tree fails to load your entries via manifest.

The documentation for the rest of the admin console commands is missing due to time constraints. Commands have help text, however, that can be accessed with `help <cmd>`.
