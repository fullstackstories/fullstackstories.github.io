---
layout: post
title:  "Running the script youtube-dl inside a docker container"
date:   2020-10-18 09:30:00 +0000
categories:
  - "Docker"
excerpt: >-
  youtube-dl is a python script that lets you download YouTube videos. In this post I'll tell what I did to run the script from inside a docker container.
cover_image_alt: "containers"  
cover_image: "post_headers/touch_play_button.jpg"
image: "/assets/images/cards/touch_play_button.jpg"  

---

## TL;DR

You don't really need docker to run `youtube-dl`. You can just install it using `brew install youtube-dl` or 
`apt install youtube-dl` in Debian based distributions and you are ready to go. However, as I mentioned in a 
[previous post]({% link _posts/2020-10-13-about-this-blog-docker.md %}), I will use docker to run the command.

** Update 2020-10-21:** Upgraded the syntax highlight styles of the post. Improve the text to make the article
more readable. 

## Docker concepts involved

Running this `youtube-dl` inside docker, instead of installing it using our package manager, gives us a great opportunity to
learn and practice a few docker concepts. I've already talked about why I'm using docker as much as possible, even when it's
overkill (like in this example). If you want to know about it, you can 
[read about it in this post]({% link _posts/2020-10-13-about-this-blog-docker.md %}).


If you follow the example, you will need to dig deeper into the following docker concepts:

* Build custom docker images using a `Dockerfile`
* Build arguments
* Differences between images and containers
* Run containers
* Bind volumes
* Bind volumes and file ownership/permissions

Explaining these docker concepts is out of the scope of this post, so I will not go into the details of any of them.

## Creating a Dockerfile for Docker for Mac: how we want to build our image

As mentioned in the [INSTALLATION section of the readme of the `youtube-dl` respository](https://github.com/ytdl-org/youtube-dl),
installing the script is as simple as download it with `wget` or `curl`, set the execute permission and run it. 
All you need is a python interpreter and `ffmpeg` for certain options.

Instead, We will create a docker image and later use that image to run the command inside a container.

Let's start by creating our working directory, `youtube-dl-docker`, in which will store all our files:

```bash
> mkdir youtube-dl-docker
> cd youtube-dl-docker
```

Make sure you are inside that folder. Open your favorite editor and create the following `Dockerfile`:

{% highlight docker linenos %}
FROM python:3-alpine

RUN  apk add --no-cache ffmpeg curl && \
     curl -L https://yt-dl.org/downloads/latest/youtube-dl -o /usr/local/bin/youtube-dl && \
     chmod a+rx /usr/local/bin/youtube-dl

VOLUME /downloads
WORKDIR /downloads
ENTRYPOINT ["youtube-dl"]
{% endhighlight %}

Let's go through the `Dockerfile`:

```docker
FROM python:3-alpine
```

Since `youtube-dl` needs python to run, we will create our image using the official python-3 image as the starting point. 

```docker
RUN  apk add --no-cache ffmpeg curl && \
     curl -L https://yt-dl.org/downloads/latest/youtube-dl -o /usr/local/bin/youtube-dl && \
     chmod a+rx /usr/local/bin/youtube-dl
```

The next step is to install the dependencies that we will need to run the script:
* `curl` to actually download the script
* `ffmpeg` if we want to use certain options of `youtube-dl`

```docker
VOLUME /downloads
```

We declare the directory `/downloads` inside the container as a volume. We will use that mount point
to [bind mount](https://docs.docker.com/storage/bind-mounts/) a directory from the operating system to store the 
downloaded video.

```docker
WORKDIR /downloads
```

`/downloads` will be our default working directory when we run any command inside a container created from this image.

```docker
ENTRYPOINT ["youtube-dl"]
```

We declare an [ENTRYPOINT](https://docs.docker.com/engine/reference/builder/#entrypoint) so the container can be run as
an executable in case we need it.

## Build the image

Once we have the `Dockerfile` in place, let's build the image. We will build it using 
[Builkit](https://docs.docker.com/develop/develop-images/build_enhancements/), which is the enhanced way
to build images with docker.

Placed inside the `youtube-dl-docker` directory, we run the following commands to build the image:
```bash
youtube-dl-docker > export DOCKER_BUILDKIT = 1
youtube-dl-docker > docker image build -t youtube-dl .
```

The first command will tell docker to build the image using 
[`buildkit`](https://docs.docker.com/develop/develop-images/build_enhancements/).

The second command is the one that will actually build the image. Depending on you internet connection, this command
may take a few minutes to run.

Once it finishes, and if everything went well, you should see a message like this:

```bash
youtube-dl-docker > docker image build -t youtube-dl .
 ...
 ...
 => => exporting layers
 => => writing image sha256:9174386f3568131240a7c336cba55b6409f46398d549d26f3e131590f3ed37d6
 => => naming to docker.io/library/youtube-dl
```

With the option `-t` we set the tag of the image (`youtube-dl` in our case). This tag name will be used later to
run the container.

## Running the command

Great! We have our image built and ready to be used. Let's try it! 

Go to [www.youtube.com](https://www.youtube.com) and select a video. Once you have the URL of the video,
let's download it by running the following command:

```bash
youtube-dl-docker > docker run \
                    --rm \ 
                    -v $(pwd):/downloads \
                    youtube-dl \
                    "https://www.youtube.com/watch?v=Q_F9CxSmGOM"
```

This command creates a container based on the image with tag name `youtube-dl` (Remember? that is the one we created above). 
Docker will create a process, isolate it, assign resources like memory and CPU to it, make all the files that are in the image
available to this process, and run it.

The process will execute the command we defined in the `ENTRYPOINT` inside the `Dockerfile` and pass the right arguments to it.
The result is that the container will run the `youtube-dl` script and will download the video. One of the keys to understand
docker is to know the difference between a container and an image. There are plenty of resources on the internet which you can
[search and browse](https://www.google.com/search?q=containers+vs+images&rlz=1C1CHBF_enES920ES920&oq=containers+vs+images) 
to understand the difference.

Once the download process finishes, you will end up with the downloaded video in the folder where you run the 
command (`youtube-dl-docker` in my case). And that's it!! 

Before continuing, let's go through the different options of the `docker run` command:

* `--rm` Remove the container after the execution finishes
* `-v $(pwd):/downloads` use a bind mount, so the downloaded video ends up in the file system of the host computer
  and not inside the container
* `youtube-dl` is the tag name of the image we use to run the container. This tag name is the same one 
  we passed to the `-t` option above when we built the image. Remember the 
  command? If not, I will copy it here for your convenience: `docker image build -t youtube-dl .`


## The problem with Linux

If you run this command in OSX after installing Docker for Mac, the owner of the downloaded file will be your system user.
However, if you follow the steps above in a Linux computer or inside WSL2 in Windows 10, you will end up with the 
downloaded files belonging to the root user:

```bash
youtube-dl-docker > ls -l
-rw-r--r-- 1 root root 3097969115 Jan  7  2020 the_video.mp4
```

This is due to how the filesystem is handled in Docker for Mac vs how it is handled in Linux.

## Fixing the file permissions in Linux

To fix this issue, we are going to change the `Dockerfile` a little bit. We will pass the user id at build time 
as a build argument and create a user inside the container with the same user id as our system user. Then, we will run
the process inside the container as that user instead of root.

Let's modify the `Dockerfile`:

```docker
FROM python:3-alpine

ARG USERID

RUN  apk add --no-cache ffmpeg curl && \
     curl -L https://yt-dl.org/downloads/latest/youtube-dl -o /usr/local/bin/youtube-dl && \
     chmod a+rx /usr/local/bin/youtube-dl && \
     adduser --uid $USERID --gecos "" --disabled-password youtubedl

VOLUME /downloads
WORKDIR /downloads

USER youtubedl

ENTRYPOINT ["youtube-dl"]
```

The differences, compared to the previous `Dockerfile` are the following:

```docker
ARG USERID
```
We define a build argument that will be the user id in the host system. We want to create a user inside the container
with the same user id as the system user and runs the process using that user.

```docker
RUN  apk add --no-cache ffmpeg curl && \
     curl -L https://yt-dl.org/downloads/latest/youtube-dl -o /usr/local/bin/youtube-dl && \
     chmod a+rx /usr/local/bin/youtube-dl && \
     adduser --uid $USERID --gecos "" --disabled-password youtubedl
```

Here, we add a command to create a user called `youtubedl` with the same id as our system user.

```docker
USER youtubedl
```

We include in our image a small "annotation" saying that by default, the process that run using this image, should run
as user `youtubedl`.

Let`s build the image passing the user id as a build argument:

```bash
youtube-dl-docker > export DOCKER_BUILDKIT = 1
youtube-dl-docker > docker image build -t youtube-dl --build-arg USERID=$(id -u) .
```

After doing this, running the same command we run before will download the files inside the container
as the user `youtubedl`. Since that user has the same user id as our system user, the files
will end up belonging to us:

```bash
youtube-dl-docker > docker run --rm -v $(pwd):/downloads youtube-dl "https://www.youtube.com/watch?v=Q_F9CxSmGOM"
youtube-dl-docker > ls -l
-rw-r--r-- 1 youruser youruser 3097969115 Jan  7  2020 the_video.mp4
```

## Improvements

### Create a `makefile` to help you run the commands

We can make our life easier by creating a `Makefile` to build the image or remove it:

```Makefile
USERID := $(shell id -u)

build: export DOCKER_BUILDKIT = 1
build:
        docker image build --build-arg USERID=$(USERID) -t youtube-dl .

clean:
        docker image rm youtube-dl
```

To build the container, we can just run: 

```bash 
youtube-dl-docker > make build
```

We can remove the image by running:

```bash 
youtube-dl-docker > make clean
```

### Create an alias

We can also create an alias to avoid writing this very long command every time we want to :

```bash
> alias youtube-dl="docker run --rm -v /Users/aalba/MyStuff/youtube-dl:/downloads youtube-dl"
```

Once you define this alias, running the command `youtube-dl "https://www.youtube.com/watch?v=Q_F9CxSmGOM"` will
download the file to the current folder.

## Limitations

Since we are running the command inside a container, those options from the `youtube-dl` command that
read or write files to the host file system, might require the definition of extra bind volumes to work properly.

## References

The code of the post can be found in [this git repository](https://github.com/alfonsoalba/youtube-dl-docker).