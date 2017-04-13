# Admin console

Once your cluster is up and running, you can log into the admin console
using

```bash
ssh admin@a.cluster.node -p 2222
```

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

The documentation for the rest of the admin console commands is missing due to time constraints. Commands have help text, however, that can be accessed with `help <cmd>`.