# prerequisites
First, [set up Docker](https://docs.docker.com/engine/installation/). Then,
make sure your user has permission to use Docker.
Typically this will involve being a member of a group named docker.

If you can successfully run `docker info`, things are probably okay.

# build an image in which to run the app

Make a settings file (and edit to taste):

```
$ cp -a settings.js.example settings.js
```

Build a new docker image using the `sudo-humans` source code:

```
$ docker build -t $LOGNAME/sudo-humans .
```

Note that `$LOGNAME` will expand to your username, so the `-t` flag, which
applies a repository name to the newly created image, will apply a repository
name that begins with your username. This is a common and recommended practice,
but of course you can call your image anything you like.

Note also that the `docker build` command ends in a dot, which is being used
to specify the current directory.

Building the container will take a while the first time you do it, since it
pulls in a base Debian image and several package dependencies. On subsequent
runs, it will reuse what has already been set up, so it shouldn't take nearly
as long.

If the build process completes successfully, a message like the following will
be displayed:

```
Successfully built d5a48c5c39d0
```

The last part of this message is the image ID, which will vary.

Once the image has been created, you should be able to see it in the image
list:

```
rcsheets@odin:~/sudo-humans$ docker images
REPOSITORY               TAG                 IMAGE ID            CREATED              VIRTUAL SIZE
rcsheets/sudo-humans     latest              d5a48c5c39d0        About a minute ago   534 MB
```

# running sudo-humans within a container

Now that your image has been created, you can start the application:

```
$ docker run -d -p 8080:80 $LOGNAME/sudo-humans
```

The immediate output of the `docker run` command will be a long hexadecimal
string. This is the container ID.

In this example, your app listens on real port 8080. Adjust the port number
to suit your preference. Within the container, the app listens on port 80.
To run the app in the foreground, omit the -d flag.

If all goes according to plan, you'll be able to see that the application
is running inside of its container:

```
rcsheets@odin:~/sudo-humans$ docker ps
CONTAINER ID        IMAGE                  COMMAND             CREATED             STATUS              PORTS                  NAMES
bcbccd48a5f4        rcsheets/sudo-humans   "npm start"         3 minutes ago       Up 3 minutes        0.0.0.0:8080->80/tcp   sick_colden
```

Note that your container can be referred to by its ID or name. The name is
usually easier to type. The ID is more suited for automation purposes.

Now that the application is running, you should be able to test it out with
your web browser. Be sure to access port 8080, as that's the real port opened
by Docker (refer to the the `PORTS` column in the output of `docker ps`).

Stop the application:

```
$ docker stop sick_colden
```

Your container name will vary. You can also specify the container by its ID.

# other possibilities

In addition to simply running the app, you can also get an interactive shell
inside the Docker container. To do this, supply the `-i` and `-t` flags to
the `docker run` command, and also specify a shell to run. As we invoked
`docker run` the first time, without specifying a command, the command
came from our Dockerfile. In the following example, we will get an interactive
bash shell:

```
$ docker run -it -p 8080:80 $LOGNAME/sudo-humans bash
root@ad488e91f56b:/usr/src/app#
```

Within the Docker container, `/usr/src/app` is a *copy* of our source code.
Once the container is destroyed, the copy is lost. If you want to attach the
container's `/usr/src/app` to your source code in such a way that changes will
persist, Docker can be configured to [mount a directory from the host within
the container][1] with the `-v` option:

[1]: https://docs.docker.com/engine/tutorials/dockervolumes/#/mount-a-host-directory-as-a-data-volume

```
$ docker run -it -p 8080:8000 --user $(id -u) -e HOME=/usr/src/app -v $(pwd):/usr/src/app $LOGNAME/sudo-humans bash
```

Now, in a shell within the container, changes you make will be preserved when
the container is stopped. For example, this could be used to add a new npm
development dependency:

```
I have no name!@89431f8e79ac:~$ npm install --save-dev mocha
sudo-humans@0.0.0 /usr/src/app
`-- mocha@3.1.2
```

A few details of the above bear explanation:
- We're now running our containerized process using the same user ID we are
  logged in as on the host. This is done so that write access to the file
  system works as expected. Otherwise, the containerized process would attempt
  to write files owned by root, which would fail under most circumstances.
- The prompt now begins with `I have no name!` because, within the container,
  your user ID does not map to a username. This is harmless.
- We are changing the environment slightly, setting our home directory
  (within the container) to `/tmp`. This is done so that programs like `npm`,
  which cache things in your home directory, work properly. Otherwise, the
  home directory would be `/`, which is not writable by your user.
- The example `npm install` command installs mocha into `/tmp`, which is
  discarded after the container is stopped; however the effect of `--save-dev`,
  which results in a change to `package.json`, is preserved.
- The port mapping (`-p 8080:8000`) is changed from the previous setting
  because the app is no longer running as root inside the container. Only root
  can bind to low port numbers (such as 80), so the app default to listening on
  port 8000 since it knows it isn't running as root. The "real" port (which
  would be used in your browser, for example) stays the same at 8080.

# fixing bugs

Let's say you've found a bug. For example, once you have the app up and running
inside a Docker container (at least as of the current revision as this document
is being written), you could do the following to see the first few lines of
HTML output when the default page is requested:

```
rcsheets@fireball:~/sudo-humans$ curl -s http://localhost:8080/ | head -3
<html>
  <head>
    <meta http-equiv="X-UA-Compatible" content="IE=Edge">
```

You might then notice that the HTML document doesn't begin with a [document
type declaration](https://en.wikipedia.org/wiki/Document_type_declaration).
If you wanted to add one, you could do the following:

1. Stop the container, as described above. You're stopping the container in
   order to restart it with a directory mounted, making it easier to preserve
   any changes made in pursuit of fixing the bug.
2. Start the container interactively with its directory mounted, as described
   above. Once the container is started, if you haven't done so already, you'll
   need to run `npm install` in order to get all of the npm dependencies ready.
   If you're not sure, it's fine to run a superfluous `npm install`.
3. Start the application using `npm start`.
4. Double-check that you can still observe the bug. In this example, you could
   run the curl command again and verify that there's still no document type
   declaration.
5. In this example, we can fix the bug by editing `static/layout.html` and
   adding the document type declaration at the top of the file.
6. Once the file has been saved, check that the bug has been fixed.
7. If your fix worked, you can now submit a pull request to get your fix into
   the official sudo-humans repository.

# troubleshooting

A recent re-architecting attempt may cause local instances of sudohumans to fail with an error message like "Error: EEXIST: File already exists, mkdir $PATH/sudoroom-data"

To fix this, run

```
$ rm sudoroom-data
$ mkdir sudoroom-data
```
