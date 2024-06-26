# Manual Operations

To better understand the `Infrastructure as Code` (`IaC`) concept, we will first define the problem we are facing and deal with it with manually to get our hands dirty and see how things work overall.

## Intro

Imagine you have developed a new cool application called [integration](https://github.com/GaabsDev/layers).

You want to run your application on a dedicated server and make it available to the Internet users.

You heard about the `public cloud` thing, which allows you to provision compute resources and pay only for what you use. You believe it's a great way to test your idea of an application and see if people like it.

You've signed up for a free tier of [Google Cloud Platform](https://cloud.google.com/) (GCP) and are about to start deploying your application.

## Provision Compute Resources

First thing we will do is to provision a virtual machine (VM) inside GCP for running the application.

Use the following gcloud command in your terminal to launch a VM with Ubuntu 16.04 distro:

```bash
$ gcloud compute instances create integration-instance-2 \
    --image-family ubuntu-1604-lts \
    --image-project ubuntu-os-cloud \
    --boot-disk-size 10GB \
    --machine-type n1-standard-1
```

## Create an SSH key pair

Generate an SSH key pair for future connections to the VM instances (run the command exactly as it is):

```bash
$ ssh-keygen -t rsa -f ~/.ssh/integration-user -C integration-user -P ""
```

Create an SSH public key for your project:

```bash
$ gcloud compute project-info add-metadata \
    --metadata ssh-keys="integration-user:$(cat ~/.ssh/integration-user.pub)"
```

Add the SSH private key to the ssh-agent:

```
$ ssh-add ~/.ssh/integration-user
```

Verify that the key was added to the ssh-agent:

```bash
$ ssh-add -l
```

## Install Application Dependencies

To start the application, you need to first configure the environment for running it.

Connect to the started VM via SSH:

```bash
$ INSTANCE_IP=$(gcloud --format="value(networkInterfaces[0].accessConfigs[0].natIP)" compute instances describe integration-instance-2)
$ ssh integration-user@${INSTANCE_IP}
```

Install Ruby:

```bash
$ sudo apt-get update
$ sudo apt-get install -y ruby-full build-essential
```

Check the installed version of Ruby:

```bash
$ ruby -v
```

Install Bundler:

```bash
$ sudo gem install --no-rdoc --no-ri bundler
$ bundle version
```

Clone the [application repo](https://github.com/GaabsDev/layers), but first make sure `git` is installed:
```bash
$ git version
```

At the time of writing the latest image of Ubuntu 16.04 which GCP provides has `git` preinstalled, so we can skip this step.

Clone the application repo into the home directory of `integration-user` user:

```bash
$ git clone https://github.com/GaabsDev/layers.git
```

Install application dependencies using Bundler:

```bash
$ cd ./integration
$ sudo bundle install
```

## Prepare Database

Install MongoDB which your application uses:

```bash
$ sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv EA312927
$ echo "deb http://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.2 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-3.2.list
$ sudo apt-get update
$ sudo apt-get install -y mongodb-org
```

Start MongoDB and enable autostart:

```bash
$ sudo systemctl start mongod
$ sudo systemctl enable mongod
```

Verify that MongoDB is running:

```bash
$ sudo systemctl status mongod
```

## Start the Application

Download a systemd unit file for starting the application from a gist:

```bash
$ wget https://gist.githubusercontent.com/GaabsDev/ce82397cfc69d912df9cd648a8d69bec/raw/7193a36c9661c6b90e7e482d256865f085a853f2/integration.service
```

Move it to the systemd directory

```bash
$ sudo mv integration.service /etc/systemd/system/integration.service
```

Now start the application and enable autostart:

```bash
$ sudo systemctl start integration
$ sudo systemctl enable integration
```

Verify that it's running:

```bash
$ sudo systemctl status integration
```

## Access the Application

Open a firewall port the application is listening on (note that the following command should be run on your local machine):

```bash
$ gcloud compute firewall-rules create allow-integration-tcp-9292 \
    --network default \
    --action allow \
    --direction ingress \
    --rules tcp:9292 \
    --source-ranges 0.0.0.0/0
```

Get the public IP of the VM:

```bash
$ gcloud --format="value(networkInterfaces[0].accessConfigs[0].natIP)" compute instances describe integration-instance-2
```

Now open your browser and try to reach the application at the public IP and port 9292.

For example, I put in my browser the following URL http://104.155.1.152:9292, but note that you'll have your own IP address.

## Conclusion

Congrats! You've just deployed your application. It is running on a dedicated set of compute resources in the cloud and is accessible by a public IP. Now Internet users can enjoy using your application.

Now that you've got the idea of what sort of steps you have to take to deploy your code from your local machine to a virtual server running in the cloud, let's see how we can do it more efficiently.

Destroy the current VM and move to the next step:

```bash
$ gcloud compute instances delete integration-instance-2
```

Next: [Scripts](03-scripts.md)
