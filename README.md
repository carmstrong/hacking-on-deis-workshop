# Hacking on Deis workshop

In this lab, we will set up a local development environment, deploy Deis 1.5.1 to AWS, and customize
the router component to respond to a new endpoint.

## Workstation setup

To deploy Deis, we'll only need the `deisctl` and `deis` clients. To hack on Deis, however, we'll
need an appropriate development environment.

### Install Docker

#### OS X
* Install [boot2docker v1.5.0](https://github.com/boot2docker/osx-installer/releases/tag/v1.5.0)
* Run `boot2docker init`
* Run `boot2docker start`
* Run `eval "$(boot2docker shellinit)"`
* Confirm Docker is working properly with `docker info`

#### Linux
* Use the official installation shell script: `curl -sSL https://get.docker.com/ | sh`
 * To see what the script does, view [https://get.docker.com/](https://get.docker.com/)
 * Most distros have a much-too-old version of Docker in their package repositories - using this script should give us the latest
* Confirm Docker is working properly with `docker info`

### Install clients

We'll need both `deis` (the Deis CLI) and `deisctl` (the Deis provision tool). Both clients are
automatically built when we ship new releases of Deis, and be grabbed with our install scripts:

```console
$ curl -sSL http://deis.io/deisctl/install.sh | sh -s 1.5.1
$ curl -sSL http://deis.io/deis-cli/install.sh | sh
```

Both scripts will dump the binaries in the current working directory. We probably want them somewhere
else:
```console
$ mv ./deis* /usr/local/bin/
```

### Clone Deis

* [Fork deis](https://github.com/deis/deis/fork) to your account
* Clone Deis: `git clone git@github.com:<username>/deis.git && cd deis`

Since we're going to be working with Deis v1.5.1, we should make sure we're using that code:
```console
$ git fetch --tags
$ git checkout v1.5.1
```

## Deploying the cluster

The Deis project has community-contributed provision scripts for various providers in the `contrib`
directory. These provision scripts provision a cluster of CoreOS machines (3 by default) with some
system tweaks for Deis. We also configure things like EBS volumes and their mount points, install
some helper scripts, etc.

For EC2, install the AWS CLI tools and configure them with AWS account credentials:

```console
$ sudo pip install awscli pyyaml
$ aws configure
AWS Access Key ID [None]: ***************
AWS Secret Access Key [None]: ************************
Default region name [None]: us-west-1
Default output format [None]:
```

If you get an error because you don't already have `pip`, you'll need to install it first:
```console
$ sudo easy_install pip
```

Create a new keypair for Deis and upload it to EC2:
```console
$ ssh-keygen -q -t rsa -f ~/.ssh/deis-carmstrong -N '' -C deis-carmstrong
$ aws ec2 import-key-pair --key-name deis --public-key-material file://~/.ssh/deis-carmstrong.pub
```

We need to tell Deis to use our key. Edit `contrib/ec2/cloudformation.json` to specify the key:
```
[
    {
        "ParameterKey":     "KeyPair",
        "ParameterValue":   "deis-carmstrong"
    }
]
```

We should also add it to our local SSH agent so it's offered when we try to log into the machines:
```console
$ ssh-add ~/.ssh/deis-carmstrong
```

Generate a new discovery URL and deploy:
```console
$ make discovery-url
$ cd contrib/ec2
$ ./provision-ec2-cluster.sh deis-carmstrong
Creating CloudFormation stack deis
{
    "StackId": "arn:aws:cloudformation:us-east-1:69326027886:stack/deis/1e9916b0-d7ea-11e4-a0be-50d2020578e0"
}
Waiting for instances to be created...
Waiting for instances to be created...
Waiting for instances to pass initial health checks...
Waiting for instances to pass initial health checks...
Waiting for instances to pass initial health checks...
Instances are available:
i-5c3c91aa  203.0.113.91    m3.large        us-east-1a      running
i-403c91b6  203.0.113.20    m3.large        us-east-1a      running
i-e36fc6ee  203.0.113.31    m3.large        us-east-1b      running
Using ELB deis-DeisWebE-17PGCR3KPJC54 at deis-DeisWebE-17PGCR3KPJC54-1499385382.us-east-1.elb.amazonaws.com
Your Deis cluster has been successfully deployed to AWS CloudFormation and is started.
Please continue to follow the instructions in the documentation.
```

## Configure DNS

We'll need to configure DNS to point to our Deis cluster.

We can either use a real domain name and create an appropriate DNS record for Deis, or we can
fake it by using [xip.io](http://xip.io) and one of the IP addresses for the ELB.

In either event, we'll need the ELB hostname. Mine is `deis-deiswebelb-1n78wvhpoyarq-1945623935.us-east-1.elb.amazonaws.com`.

### Using a real domain name

We simply configure a CNAME record for `*.domain.tld` to point to the ELB name. For example:
```console
$ dig some-app.atribecalledchris.com CNAME +short
deis-deiswebelb-1n78wvhpoyarq-1945623935.us-east-1.elb.amazonaws.com.
```

### Using xip.io

This is super hacky, as it's not recommended to create A records for AWS ELBs, as the actual IPs can
change. However, for short-lived clusters, this should work just fine. First, we need to get an
IP for our ELB:

```console
$ host deis-deiswebelb-1n78wvhpoyarq-1945623935.us-east-1.elb.amazonaws.com
deis-deiswebelb-1n78wvhpoyarq-1945623935.us-east-1.elb.amazonaws.com has address 54.164.177.68
deis-deiswebelb-1n78wvhpoyarq-1945623935.us-east-1.elb.amazonaws.com has address 54.209.130.62
```

Then, we can compose a domain as `some-string.<IP>.xip.io`. For example:
```console
$ dig some-app.54.164.177.68.xip.io A +short
54.164.177.68
```

## Installing Deis

Now, we have the running CoreOS machines and a domain we can use to access the cluster. It's time
to actually stand up Deis on the hosts.

We'll need one of the public IPs of our instances (these were output by the provision script). Any
one will do. Then, we set an environment variable so our local `deisctl` knows who to talk to:

```console
$ export DEISCTL_TUNNEL=52.4.161.101
```

Now, we need to have `deisctl` tell our CoreOS cluster what our domain will be. Note that this is
the domain one level above what the apps will be called (i.e. the domain we set here is
`atribecalledchris.com`, and our apps will be `app-name.atribecalledchris.com`).

```console
$ deisctl config platform set domain=54.164.177.68.xip.io
```

We also provide the SSH key we used to provision the hosts. This is used by the `deis run` command
to log into the host machine and schedule one-off tasks:

```console
$ deisctl config platform set sshPrivateKey=~/.ssh/deis-carmstrong
```

*Finally*, we can install and start the platform. This will take about 20 minutes for the platform
to fully start, as it's pulling each component's Docker image from Docker Hub.

```console
$ deisctl install platform
$ deisctl start platform
```

What exactly did this do? `deisctl` talked to [fleet](https://github.com/coreos/fleet), the CoreOS cluster scheduler, and asked it to run the Deis components on the cluster. We can use `deisctl list` to see on which hosts  the components ended up.

## Deploying apps

We'll need to register a Deis user on our new cluster. The `deis` CLI interacts with the Deis
`controller` component, which is our API server. The routers know to forward requests to `deis.*`
along to the controller. So, we log into `deis.DOMAIN`:

```console
$ deis register http://deis.54.164.177.68.xip.io
username: chris
password:
password (confirm):
email: carmstrong@engineyard.com
Registered chris
Logged in as chris
```

Now, let's see Deis in action by deploying a sample Ruby app. Clone our `example-ruby-sinatra`
somewhere:

```console
$ git clone git@github.com:deis/example-ruby-sinatra.git && cd example-ruby-sinatra
```

We create an app for Deis:
```console
$ deis create
Creating application... done, created usable-riverbed
Git remote deis added
```

We see that there's a new remote for pushing our app:
```console
$ git remote -v
deis  ssh://git@deis.54.164.177.68.xip.io:2222/usable-riverbed.git (fetch)
deis  ssh://git@deis.54.164.177.68.xip.io:2222/usable-riverbed.git (push)
origin  git@github.com:deis/example-ruby-sinatra.git (fetch)
origin  git@github.com:carmstrong/example-ruby-sinatra.git (push)
```

Before we can push the app, we need to tell Deis who we are by uploading our pubkey.
```console
$ deis keys:add
Found the following SSH public keys:
1) deis-carmstrong.pub deis
2) id_rsa.pub carmstrong@carmstrong-mbp.local
0) Enter path to pubfile (or use keys:add <key_path>)
Which would you like to use with Deis? 1
Uploading deis to Deis...done
```

Let's try a `git push`:
```console
$ git push deis master
Counting objects: 105, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (55/55), done.
Writing objects: 100% (105/105), 22.42 KiB | 0 bytes/s, done.
Total 105 (delta 44), reused 101 (delta 43)
-----> Ruby app detected
-----> Compiling Ruby/Rack
-----> Using Ruby version: ruby-1.9.3
-----> Installing dependencies using 1.7.12
       Running: bundle install --without development:test --path vendor/bundle --binstubs vendor/bundle/bin -j4 --deployment
       Fetching gem metadata from http://rubygems.org/..........
       Using bundler 1.7.12
       Installing tilt 1.3.6
       Installing rack 1.5.2
       Installing rack-protection 1.5.0
       Installing sinatra 1.4.2
       Your bundle is complete!
       Gems in the groups development and test were not installed.
       It was installed into ./vendor/bundle
       Bundle completed (3.49s)
       Cleaning up the bundler cache.

-----> Discovering process types
       Procfile declares types -> web
       Default process types for Ruby -> rake, console, web
-----> Compiled slug size is 15M

-----> Building Docker image
remote: Sending build context to Docker daemon 15.09 MB
remote: build context to Docker daemon
Step 0 : FROM deis/slugrunner
 ---> e7d9c5776fa8
Step 1 : RUN mkdir -p /app
 ---> Running in 94c539c82c87
 ---> 0911f4cab93f
Removing intermediate container 94c539c82c87
Step 2 : WORKDIR /app
 ---> Running in 2fb75cc26175
 ---> 55e5e656dd4f
Removing intermediate container 2fb75cc26175
Step 3 : ENTRYPOINT /runner/init
 ---> Running in 29a92b4125e5
 ---> bdc4a79bb99f
Removing intermediate container 29a92b4125e5
Step 4 : ADD slug.tgz /app
 ---> e5e5e4d6325e
Removing intermediate container d5b6c0cdd1d9
Step 5 : ENV GIT_SHA 284de691058c28dce482f25ec5e2b9c6b7c632c4
 ---> Running in 050e39eb7e73
 ---> dee5296a994b
Removing intermediate container 050e39eb7e73
Successfully built dee5296a994b
-----> Pushing image to private registry

-----> Launching...

       done, usable-riverbed:v2 deployed to Deis

       http://usable-riverbed.54.164.177.68.xip.io

       To learn more, use `deis help` or visit http://deis.io

To ssh://git@deis.54.164.177.68.xip.io:2222/usable-riverbed.git
 * [new branch]      master -> master
```

If we run `deis open`, we see that the deploy was successful.

Let's try scaling up more containers of our app:
```console
$ deis scale web=3
Scaling processes... but first, coffee!
done in 133s
=== usable-riverbed Processes

--- web:
web.1 up (v2)
web.2 up (v2)
web.3 up (v2)
```

If we refresh our browser, we see three containers responding to traffic.

There are a lot more commands we could play with - `deis config:set` to set
environment variables, `deis logs` to see request and audit logs for the app,
`deis rollback` to roll back to a previous release, and
many more. Use `deis help` to see all the commands.

## Customizing Deis

We're going to modify the `router` component to impement part of the
[Hyper Text Coffee Pot Control Protocol](http://en.wikipedia.org/wiki/Hyper_Text_Coffee_Pot_Control_Protocol).

In the Deis code, edit `router/image/templates/nginx.conf` in your favorite code editor. After line 276,
insert the following code:
```
        location /teapot {
            default_type 'text/plain';
            access_log off;
            return 418 Teapot;
        }
```

The block should now look like the following:
```
    # healthcheck
    server {
        listen 80 default_server;
        location /health-check {
            default_type 'text/plain';
            access_log off;
            return 200;
        }
        location /router-nginx-status {
            stub_status on;
        }
        location /teapot {
            default_type 'text/plain';
            access_log off;
            return 418 Teapot;
        }
    }
```

Commit your awesome changes. Great job!
<img src="http://allthingsxbox.com/wp-content/uploads/2014/06/Obama-Thumbs-Up.jpg">


Now, we build the container:
```console
$ make -C router build
```

The Makefile built a container named `deis/router` that was tagged with a shortened version of the git SHA of your
latest commit (for example, `git-f66ae3b`). We can ask git for this:
```console
$ IMAGE=$(echo deis/router:git-$(git rev-parse --short HEAD))
$ echo $IMAGE
deis/router:git-f66ae3b
```

Let's log into Docker Hub so we can upload the image there:
```console
$ docker login
Username: carmstrong
Password:
Email: chris@chrisarmstrong.me
Login Succeeded
```

We need to re-tag the image before we can upload it to our Docker Hub account:
```console
$ docker tag $IMAGE carmstrong/deis-router:teapot
$ docker push carmstrong/deis-router:teapot
```

Now, we can tell the Deis cluster we'd like to use a custom router image:
```console
$ deisctl config router set image=carmstrong/deis-router:teapot
```

Whenever a Deis component starts, it runs a shell script which generates the name of the Docker image
that is used for the container. By setting this etcd key with `deisctl`, we told the script to use
our custom image instead of the default (which is `deis/<component>:<version>`, or `deis/router:v1.5.1`).

All that's left is to restart the routers. When they start again, they'll pull down our custom image and
start that instead of the stock router:

```console
$ deisctl restart router@*
```

We can confirm that our custom router is running by hitting the endpoint:

```console
$ curl http://foo-bar.54.164.177.68.xip.io/teapot
Teapot
```

I'm a little teapot, short and stout...

Happy Hacking!

## Cleaning up

You'll need to delete the CloudFormation stack when you're finished. Terminating the instances doesn't do anything, as the autocsaling group will launch new instances to replace them. You can do this through the AWS web UI, or by using the AWS CLI:
```console
$ aws cloudformation delete-stack <stackname>
```

## References

This workshop may fall out of date. The official [Deis documentation](http://docs.deis.io/en/latest/)
should be considered the source of truth whenever it conflicts with these instructions.

This workshop follows the following sections of the official documentation:
* [Hacking on Deis](http://docs.deis.io/en/latest/contributing/hacking/)
* [Provisioning Deis on AWS](http://docs.deis.io/en/latest/installing_deis/aws/)
* [Install deisctl](http://docs.deis.io/en/latest/installing_deis/install-deisctl/)
* [Install the Deis Platform](http://docs.deis.io/en/latest/installing_deis/install-platform/)
* [Install the Client](http://docs.deis.io/en/latest/using_deis/install-client/)

