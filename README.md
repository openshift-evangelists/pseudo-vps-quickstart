Pseudo Virtual Private Server
-----------------------------

When using OpenShift, if you miss the idea of a virtual private server (VPS) where you SSH into your server and work directly in it, then this repository is for you. The scripts and template provided will create a pseudo VPS with a persistent volume attached, and a supervisor installed which you can configure to run your services. It can be used as a place to play around, or you can copy your files into it using ``oc rsync``, or use ``git`` to check out your web application code and work on it directly on OpenShift. If you want to ensure your web application stays running, you can add it to the supervisor configuration so it is automatically restarts if the pod restarts.

Sounds great. Just be aware that there is one limitation. That is you cannot do anything as the ``root`` user, so you cannot install additional system packages. The only packages available are those provided by the S2I builder image which you use. It has been tested with the ``nodjs``, ``php``, ``python``, ``ruby`` and ``wildfly`` S2I base images. The concept should work with any S2I builder image which uses the root directory ``/opt/app-root`` as the location for applications, with any build artifacts being installed under that directory. And, where the S2I builder image uses either the ``/tmp/src`` or ``/opt/s2i/destination/src`` directories as the transfer point for source code injected into the build process.

Deploying your Pseudo VPS
-------------------------

To deploy your pseudo VPS, first load the template file.

```
oc create -f https://raw.githubusercontent.com/openshift-evangelists/pseudo-vps-quickstart/master/templates.json
```

From the web console click on the _Add to Project_ menu, then _Select from Project_. In the search filter enter ``vps`` if necessary, then select _Pseudo VPS_.

From the command line, you can use the command:

```
oc new-app --template pseudo-vps-quickstart
```

The template parameters which can be supplied are:

```
Parameters:
    Name:       APPLICATION_NAME
    Required:   true
    Value:      pseudo-vps

    Name:       APPLICATION_MEMORY
    Required:   true
    Value:      512Mi

    Name:       BUILDER_IMAGE
    Required:   true
    Value:      python:3.5

    Name:       VOLUME_SIZE
    Required:   true
    Value:      1Gi
```

The ``BUILDER_IMAGE`` parameter can specify any of the S2I builders in the default ``openshift`` project. For example ``nodejs``, ``php``, ``python``, ``ruby`` and ``wildfly`` images. Ensure you always use a specific version tag for the image, not ``latest``. If you use ``latest``, the language runtime version could unexpectedly change and the new version may be incompatible with your application.

Accessing your Pseudo VPS
-------------------------

You can access your pseudo VPS from the web console by selecting _Applications_ in the left side navigation bar, then _Pods_. Select the pod and then the _Terminal_ tab.

From the command line, run ``oc get pods`` to get a list of the pods, and then use ``oc rsh`` to get an interactive shell running in the pod.

```
$ oc get pods
NAME                 READY     STATUS      RESTARTS   AGE
pseudo-vps-1-build   0/1       Completed   0          42m
pseudo-vps-1-pv8f5   1/1       Running     1          41m

$ oc rsh pseudo-vps-1-pv8f5
(app-root)sh-4.2$
```

You can use ``oc rsync`` to copy files into the pod, or use ``git`` to checkout files from a remote repository into the pod.

Any files placed anywhere under the directory ``/opt/app-root`` will be preserved across a restart of the container.

Working with a Code Repository
------------------------------

If the application source code you want to work in is hosted in a Git repository, the steps you can use to check it out are as follows.

```
(app-root)sh-4.2$ git init .
Initialized empty Git repository in /opt/app-root/src/.git/

(app-root)sh-4.2$ git remote add origin https://github.com/openshift-katacoda/blog-django-py.git

(app-root)sh-4.2$ git fetch
remote: Counting objects: 350, done.
remote: Compressing objects: 100% (5/5), done.
remote: Total 350 (delta 0), reused 2 (delta 0), pack-reused 345
Receiving objects: 100% (350/350), 59.05 KiB | 0 bytes/s, done.
Resolving deltas: 100% (161/161), done.
From https://github.com/openshift-katacoda/blog-django-py
 * [new branch]      master     -> origin/master

(app-root)sh-4.2$ git checkout -t origin/master -f
 Branch master set up to track remote branch master from origin.
 Already on 'master'
```

If your source code repository is designed to work with the S2I builder base image you are using, you can run ``/usr/libexec/s2i/assemble`` to build the application. If you had overridden the S2I scripts in your source code repository, run ``/opt/app-root/src/.s2i/bin/assemble``.

Note that the S2I builders may fail if they don't like the fact that the ``/tmp/src`` directory is empty. In that case you will need to first create a dummy file under ``/tmp/src`` so the directory isn't empty. You will need to do this each time the ``assemble`` script is run.

```
$ touch /tmp/src/.dummy

$ .s2i/bin/assemble
...
```

So long as the S2I builder doesn't remove the original files when building the application, you should be able to run ``assemble`` multiples, creating that dummy file under ``/tmp/src`` as necessary.

To run your web application, run the ``/usr/libexec/s2i/run`` script, or ``/opt/app-root/src/.s2i/bin/run`` if you had overridden it in your source code repository.

```
(app-root)sh-4.2$ .s2i/bin/run
 -----> Running application run script.
...
```

A web application should use the port the S2I builder base image uses. This is usually port 8080 and the templates have been setup to expect that port. The template automatically creates a route for you so the port is exposed and can be access outside of the OpenShift cluster.

Note that you can't use ``git`` to clone repositories if using the ``ruby``, ``php`` or ``wildfly`` S2I builder images. This is because they don't provide a way of adding missing entries to the ``passwd`` file and the version of ``git`` used requires an entry to work.

Copying Files from Local System
-------------------------------

If instead of using Git you want to copy files from the local system, you can use ``oc rsync``. Use ``oc get pods`` to work out the name of the pod and then run ``oc rsync``.

```
$ oc get pods
NAME                 READY     STATUS      RESTARTS   AGE
pseudo-vps-1-build   0/1       Completed   0          42m
pseudo-vps-1-pv8f5   1/1       Running     1          41m

$ oc rsync ./ pseudo-vps-1-pv8f5:/opt/app-root/src/ --no-perms
```

The ``--no-perms`` option tells ``oc rsync`` not to attempt to preserve permissions on directories and files. This is necessary, when copying files to the local container filesystem, and the directory into which files are being copied is not owned by the user ID the container is running as, but rather by the user that the S2I builder was run as. Without this option, ``oc rsync`` would fail when it attempts to change the permissions on the directory.

The ``oc rsync`` command can also be used to copy directories or files from a running container back to the local system. Copying in either direction can be run as a one-off event, or you can have ``oc rsync`` continually monitor for changes and copy files each time they are changed.

Running an Application Permanently
----------------------------------

To run your application permanently, you can add configuration for it into the ``supervisord`` configuration. The configuration file is located at ``/opt/app-root/etc/supervisord.conf``. A suitable configuration for this example would be:

```
[program:blog]
command=/opt/app-root/src/.s2i/bin/run
stdout_logfile=/proc/1/fd/1
stdout_logfile_maxbytes=0
redirect_stderr=true
```

Ensure you use the log file entries as shown so that any log messages output from the web application are captured in the pod logs.

Once you have added the entry, you can start the web application as a service by running ``supervisord reload``.

```
(app-root)sh-4.2$ supervisorctl reload
Restarted supervisord
```

If you need to subsequently stop or start the web application, use ``supervisorctl stop`` and ``supervisorctl start`` commands. Pass the name of the service as argument. You can check on the status of the application using ``supervisorctl status``.

```
(app-root)sh-4.2$ supervisorctl stop blog
blog: stopped
(app-root)sh-4.2$ supervisorctl start blog
blog: started
(app-root)sh-4.2$ supervisorctl status
blog                             RUNNING   pid 885, uptime 0:00:03
```

If you manage to make an error in the ``supervisord`` configuration, it will cause ``supervisord`` to abort and the pod will stop. If this occurs, scale down the deployment so there are no replicas, then use ``oc debug`` to run a debug instance and from the interactive shell, make any corrections to the configuration file necessary. Run the 'original command' down by ``oc debug`` to test your changes. Kill ``supervisord`` if all is good and exit the debug session. Then scale up the deployment again.

```
$ oc scale dc/pseudo-vps --replicas 0
deploymentconfig "pseudo-vps" scaled

$ oc debug dc/pseudo-vps
Debugging with pod/pseudo-vps-debug, original command: container-entrypoint /tmp/scripts/run
Waiting for pod to start ...
Pod IP: 172.17.0.4
If you don't see a command prompt, try pressing enter.

(app-root)sh-4.2$ vi /opt/app-root/etc/supervisord.conf
...

(app-root)sh-4.2$ exit
exit

Removing debug pod ...

$ oc scale dc/pseudo-vps --replicas 1
deploymentconfig "pseudo-vps" scaled
```

Running Multiple Applications
-----------------------------

As ``supervisord`` is a general purpose supervisor system, it is possible to run multiple applications. You could run a separate application which runs periodic tasks in support of a web application. You could also run a second web application.

In the case of running multiple web applications which you want to expose via a public route, only one can use the default route created using port 8080. The second web application will need to use a different port, and you will have to edit the existing _Service_ object to add an additional port reference, or create a new _Service_ object for the second port. You can then create an additional _Route_ object to expose it via a public URL.

Deleting the Deployment
-----------------------

To delete the complete deployment, including the persistent volume claim, use ``oc delete``, supply a label selector to match the application components.

```
oc delete all,pvc --selector app=pseudo-vps
```
