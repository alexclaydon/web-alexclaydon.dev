Recently, as part of a telehealth services rollout, I went through the process of customising and deploying an instance of Jitsi-Meet to Docker on an Ubuntu 20.04 LTS host. The following are learnings from that experience. This is an early draft; I'll continue updating with additional content and clarifications in future. This article covers a development deployment; in a future piece I'll cover production deployments.

First of all, don't forget to drop by the Jitsi-Meet [community forums](https://community.jitsi.org/); they are an excellent source of discussion on problems that you may face. However, most of the participants seem to be developing and deploying using the standard "quick install" or full install method - not using Docker. For reasons that are beyond the scope of this article, I wanted to develop against, and deploy to, Docker containers. I know I'm not alone in this: see, eg, [here](https://community.jitsi.org/t/dev-workflow-question-how-to-build-a-custom-docker-image-of-jitsi-meet-web-for-use-in-our-dockerized-setup/73527), but the answers I found were not entirely on point for my use case.

So, the following covers a range of issues that might come up from developing and deploying a Dockerized Jitsi-Meet instance, including:
- How to duplicate (not fork) the relevant Git repos, to allow you to develop against your own private repo, protecting any potentially sensitive commercial information while still benefiting from updates to the main Jitsi-Meet repos;
- How to set up your back and front-end contexts to allow for a convenient development workflow; and
- At a later date, instructions on deploying to production using Docker.

A disclaimer: I'm setting out herein what's worked for me to try to help anyone that might be interested to avoid the pain points that I came up against; you may think you can improve on the below - if so, feel free to write down your thoughts and add them to the discussion on the community forums. The more information the better!

## Getting the code

First things first: you need to get the code. The relevant GitHub repos are as follows: [`jitsi-meet`](https://github.com/jitsi/jitsi-meet), and [`docker-jitsi-meet`](https://github.com/jitsi/docker-jitsi-meet).

You'll only need a couple of files from the `docker-jitsi-meet` repo, so we'll deal with those later, but if you want to make heavy customisations to the front-end web app, you'll be making a large number of changes to the `jisti-meet` repo. Although your first instinct may be to fork it, note that GitHub doesn't permit forks of public repos to be made private. Accordingly, if you think there might be a possibility that commercially sensitive information ends up in your codebase, you'll want instead to _duplicate_ the `jitsi-meet` repo. This will allow you to work on your own private copy of the repos but also to pull in changes made to `master` using `git pull` or `git rebase` (depending on your needs).

Do the following:

1.  Make a new empty, private repository on GitHub
2.  Bare clone the public `jitsi-meet` repo to your local machine: `git clone --bare https://github.com/jitsi/jitsi-meet.git`
3.  Mirror push the local bare repo to your new empty private repo; e.g.: `cd jitsi-meet && git push --mirror https://github.com/exampleuser/private-jitsi-repo.git`
4.  Delete the local bare copy of the public repo, which you no longer need: `cd .. && rm -rf jitsi-meet`
5.  Create a local clone of your private repo: `git clone https://github.com/exampleuser/private-jitsi-repo.git`
6.  Add the public upstream repo to the private repo as a remote: `git remote add upstream https://github.com/jitsi/jitsi-meet.git`
7.  Disable pushing to the public upstream repo (you don't have push priveleges anyway): `git remote set-url --push upstream DISABLE`
8.  Each time you want to pull the latest commits made to the public upstream repo, use: `git pull upstream master` and then push those new commits to your private repo `git push origin master`. Alternatively, you might want to do a `git rebase` instead, depending on your needs.

Your private duplicate of the `jitsi-meet` repo contains all of the front-end files that you might like to customise - the React components, `.html` and `.css` files, etc. Here's where you'll do most of your work.

You can use `make dev` from the project root (see the included `Makefile` for details) to install your `node_modules`, compile the `SCSS`, etc., and spin up the `webpack-dev-server` to see your changes to the front-end reflected immediately in your browser using `webpack` live-reload. However, when first doing this I noted that some of the interface changes I had made in the repo itself - in fact, notably, all of the settings in `interface_config.js` file, as opposed to changes made to, for example, `.html` and `.css` files directly - weren't displaying correctly in my `localhost` front-end. For example, the Jitsi Meet watermark would display on the front page and in-call, instead of my custom watermark, even though I had specified one in `interface_config.js`.

Long story short, the settings in `interface_config.js` actually configure the back-end service, not the front end - notwithstanding the file name. Accordingly - and perhaps unsurprisingly - to get the complete picture of how your production user interface will look, you will need to also bring up a local back-end.

## Setting up a local development back-end

This is probably the topic on which there's the least Docker-specific advice out there. To set up your back-end using Docker, you'll need a couple of files from the [`docker-jitsi-meet`](https://github.com/jitsi/docker-jitsi-meet.git) repo. You can either `git clone` or download this repo - no need to fork or duplicate it. The files you want are the `docker-compose.yml` file in the root directory, and the `Dockerfile` and `rootfs` directory (and its contents) from the `web` subfolder. Personally, I've saved these files into a custom provisioning repo that I set up to serve the particular client I'm working with. This private repo also contains development notes, some of the `bash` provisioning scripts set out below, various Ansible playbooks for setting up infrastructure, etc, and the `.env` files used by Docker when bringing up the container stack. This keeps all of the provisioning and deployment code in one place.

If you take a look at the `docker-compose.yml` file mentioned above, you can see that where you need to get to is a custom-built `jitsi/web` image that incorporates all of the custom UI changes that you've made to your private `jitsi-meet` repo. If you're familiar with Docker you'll know that there are a few different ways you can go about doing that. I have seen some discussion about `SSH`-ing into running containers (or using `docker cp` from the host) and making your changes directly to the contents of the `/usr/share` target `jitsi-meet` deploy directory on the container, then committing those changes to a new Docker image based on the existing one. That might be okay if you're making only minor changes to your front-end, but in general that practice is discouraged. You want to make any required changes in the relevant `Dockerfile` you're using to build your `jitsi/web` container instead.

Let's take a look at the `Dockerfile` for the `jitsi/web` image:

```Dockerfile
ARG JITSI_REPO=jitsi
FROM ${JITSI_REPO}/base

ADD https://dl.eff.org/certbot-auto /usr/local/bin/

COPY rootfs/ /

RUN \
	apt-dpkg-wrap apt-get update && \
	apt-dpkg-wrap apt-get install -y cron nginx-extras jitsi-meet-web && \
	apt-dpkg-wrap apt-get -d install -y jitsi-meet-web-config && \
    dpkg -x /var/cache/apt/archives/jitsi-meet-web-config*.deb /tmp/pkg && \
    mv /tmp/pkg/usr/share/jitsi-meet-web-config/config.js /defaults && \
	mv /usr/share/jitsi-meet/interface_config.js /defaults && \
	apt-cleanup && \
	rm -f /etc/nginx/conf.d/default.conf && \
	rm -rf /tmp/pkg /var/cache/apt

RUN \
	chmod a+x /usr/local/bin/certbot-auto && \
	certbot-auto --noninteractive --install-only

EXPOSE 80 443

VOLUME ["/config", "/etc/letsencrypt", "/usr/share/jitsi-meet/transcripts"]
```

The first thing you'll notice is that `jitsi-meet-web` is installed as an `apt` package. Initially, I had thought that where I wanted to get to was packing up my own custom `jitis-meet` repo into an `apt` package and then replacing the existing `apt` package in the `Dockerfile`. That approach, though, comes with its own overhead - and thankfully, there's an easier way.

If you're interested to read some of the discussion, [this](https://community.jitsi.org/t/change-welcome-page-text-after-quick-install/15279/3), [this](https://community.jitsi.org/t/how-to-how-to-build-jitsi-meet-from-source-a-developers-guide/75422), [this](https://community.jitsi.org/t/development-in-localhost-jitsi-meet/17773/11) and [this](https://community.jitsi.org/t/dev-workflow-question-how-to-build-a-custom-docker-image-of-jitsi-meet-web-for-use-in-our-dockerized-setup/73527/2) got me oriented in the right direction.

Here's how I'm doing it:

- `cd` into your custom `jitsi-meet` repo and call `make && make source-package`. The zip archive outputted as a result of this operation (`jitsi-meet.tar.bz2`) will contain all of your user interface customisations.
- Extract the archive into your `jitsi-web` Docker image build context (being the directory that also contains your `Dockerfile` and the `rootfs` directory taken from the `docker-jisti-meet` repo above). If you've just cloned the entire `docker-jitsi-meet` repo from GitHub (instead of setting up your own personal provisioning repo), the Docker build context will be the `web` subdirectory of that repo. The contents of the archive will be extracted into a subdirectory called `jitsi-meet`.
- `cd` into your build context, open the `Dockerfile` for editing, and add the following two lines just above the `EXPOSE` command:
```Dockerfile
COPY jitsi-meet/ /usr/share/jitsi-meet
COPY jitsi-meet/interface_config.js /defaults/interface_config.js
```
- This will tell Docker to copy your own custom code on top of the existing `jitsi-meet` web app installed using the pre-built `apt` package. 
- You now need to build your custom image based on this `Dockerfile`. From the build context, call `docker build -t jitsi/web .` (note: don't exclude that trailing period, which tells Docker that this directory is the build context). Note that if you already have a local copy of the `jitsi/web` image downloaded from the DockerHub container registry (for example, as part of a previous Docker installation of `jitsi-meet`), you'll need to add the `--no-cache` flag to your build command to ensure that image is overwritten (`docker build --no-cache -t jitsi/web .`).

Once all of your images are built, follow the usual instructions [here](https://jitsi.github.io/handbook/docs/devops-guide/devops-guide-docker) to bring up the entire Jitsi-Meet container stack, which now includes a web app container built from your custom `jitsi/web` image. Note that you will likely want to have different contents in your `.env` file depending on whether you are doing a development or production deployment. See the Docker devops guide for more information on all of the environment variables. Specifically for the purposes of setting up a development back-end, you will need to ensure both that Let's Encrypt is disabled, and that you uncomment the `DOCKER_HOST_ADDRESS=` env and insert the IP address of the host machine. We'll look at this in more detail later.

I've packaged up all of the above into a simple bash script that I can run from the command line every time I want to re-compile and re-deploy the back-end into a new Docker container after making changes. Note that the script, however, doesn't touch your firewall ports - you'll need to do that manually - and that it assumes that you don't already have a locally-cached version of the `jitsi/web` image downloaded from DockerHub (in which case, you'll need to pass the `--no-cache` flag to your build command).

Here it is; feel free to use and to modify to fit your needs:

```bash
#!/bin/bash

REPO_ROOT=~/dev/projects/telehealth-provisioning
TELEHEALTH_JITSI_MEET_DIR=~/dev/projects/telehealth-jitsi-meet
JITSI_MEET_CONFIG_DIR=~/.jitsi-meet-cfg

echo "Building Jitsi-Meet packages from source ..." && sleep 1

cd $TELEHEALTH_JITSI_MEET_DIR || { echo "Failed to find $TELEHEALTH_JITSI_MEET_DIR"; exit 1; }
make || { echo "Make failed"; exit 1; }
make source-package || { echo "Make source-package failed"; exit 1; }

echo "Moving and unzipping Jitsi-Meet package archive into Docker build context" && sleep 1

mv $TELEHEALTH_JITSI_MEET_DIR/jitsi-meet.tar.bz2 $REPO_ROOT/Docker/web || { echo "Failed to move packages to destination"; exit 1; }
cd $REPO_ROOT/Docker/web || { echo "Failed to find $REPO_ROOT/Docker/Web"; exit 1; }
tar -xvf jitsi-meet* || { echo "Failed to unzip archive"; exit 1; }
rm -rf $REPO_ROOT/Docker/web/jitsi-meet.tar.bz2

echo "Rebuilding Jitsi-Meet web Docker image" && sleep 1

docker build -t jitsi/web . || { echo "Docker operation failed"; exit 1; }
rm -rf $REPO_ROOT/Docker/web/jitsi-meet 

echo "Taking down the Docker container stack" && sleep 1

cd $REPO_ROOT/Docker || { echo "Failed to find $REPO_ROOT/Docker"; exit 1; }
docker-compose down 

echo "Removing and recreating Jitsi-Meet config directories" && sleep 1

sudo rm -rf $JITSI_MEET_CONFIG_DIR || { echo "Failed to remove $JITSI_MEET_CONFIG_DIR"; exit 1; }
mkdir -p $JITSI_MEET_CONFIG_DIR/{web/letsencrypt,transcripts,prosody/config,prosody/prosody-plugins-custom,jicofo,jvb,jigasi,jibri} || { echo "Failed to create required config directories at $JITSI_MEET_CONFIG_DIR root"; exit 1; }

echo "Rebuilding Jitsi-Meet web Docker container and bringing up Docker container stack" && sleep 1

cd $REPO_ROOT/Docker || { echo "Failed to find $REPO_ROOT/Docker"; exit 1; }
./gen-passwords.sh
docker-compose up -d || { echo "Docker operation failed"; exit 1; }

echo "Back-end is up"
```

## Setting up a local development front-end

So, we've got our custom front-end UI code, we've got a custom back-end development server hosted using Docker. Now we want to bring up the front-end, connect it to the back-end and enable live-reloading with `webpack`, so you can see your UI changes in realtime.

Take a look [here](https://jitsi.github.io/handbook/docs/dev-guide/dev-guide-web). Note that if you bring up your front-end with `make dev` but without explicitly specifying your own back-end in the `WEBPACK_DEV_SERVER_PROXY_TARGET` environment variable, Jitsi-Meet will by default connect to the publicly available back-end at `alpha.jitsi.net`. That might be fine if you just want to test some changes that you know don't depend on which back-end you're connected to, but it won't give you any sense of what your instance will look like in production.

So, `cd` into your custom `jitsi-meet` repo and then start the webpack server as follows, passing your custom back-end as an environment variable:

`WEBPACK_DEV_SERVER_PROXY_TARGET="https://$HOST-MACHINE-IP-ADDRESS:8443" make dev`

where `$HOST-MACHINE-IP-ADDRESS` is your host machine IP, and the number after the colon is the `HTTPS` port specified in your `.env` file (by default, 8443). Your development front and back ends are now up and connected.

Don't forget to open the required ports in the firewall on your host machine; for example:

```bash
sudo ufw allow 8000/tcp
sudo ufw allow 8443/tcp
sudo ufw allow 4443/tcp
sudo ufw allow in 10000:20000/udp
```

That's it. Open a browser and navigate to `https://localhost:8080/` (default port 8080 is set in your `.env` file) for a complete picture of how your stack looks. Any changes made to the front-end will be cause `webpack` to re-compile and re-display the output in your browser automatically, although any changes you make to files that affect the back-end will require you to re-run the container build process detailed above (preferrably using that `bash` script or something like it for convenience.)

I'll document my production deployment process in a future article.

Thanks for reading and I hope that's of some use to somebody.