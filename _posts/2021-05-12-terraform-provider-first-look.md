---
title: "OpenFaaS Terraform provider - First look"
description: "We take a look at using the new terraform provider for openfaas to deploy our stack in one command"
date: 2021-05-12
image: /images/2021-tf-provider/tf-banner.jpg
categories:
- kubernetes
- terraform
- openfaas
author_staff_member: alistair
dark_background: true

---

In this post we are going to walk through how to use [Hashicorp Terraform](https://www.terraform.io/) to deploy an OpenFaaS application in one command. 
We will define all of our infrastructure and application as code to allow us to make changes with confidence.


We are going to use the following technology and providers:

* [OpenFaaS](https://openfaas.com) as our serverless framework 
* [Civo](https://civo.com) for hosted Kubernetes
* [Terraform](https://www.terraform.io/) for defining our resources and deploying our infrastructure


## What is a `Terraform Provider`
A `terraform provider` is a layer that converts the complex logic of resource "lifecycle management" into a plugin that
a provider author can distribute to users of their service, so for OpenFaaS we can work out how to update your already running 
function with your new label or new function name. We know that a new function name would mean destroying the old function, and creating a 
new deployment.

We then write the required API calls to manage this resource lifecycle and embed the logic into our provider code so the users
of the `openfaas terraform provider` can focus on building their products and projects and we can focus on the complexities of 
translating your "intent" into api calls that bring the state of the system in-line with your configuration.

So a user then writes some `terraform code` to define the desred state of the system, and our provider code goes off and checks
the current state of the system and issues a set of api calls to bring the current state in-line with your desired state.


## Why Terraform?

We already provide OpenFaaS users with a CLI, API Spec and there is also the [openfaas-operator](https://github.com/openfaas/faas-netes/blob/master/README-OPERATOR.md) 
which allows our users to define their functions using a Kubernetes CRD. What does a terraform provider bring to the table that
the CLI/API or operator don't? 

As you will see from this post there are terraform providers for everything! Our cloud provider, Kubernetes, Helm and many more. This
allows us to define our entire system using terraform, and then deploy and update our services from one place. 

There is even a terraform provider for [ordering Pizza](https://github.com/ndmckinley/terraform-provider-dominos). If it has an API, you can 
write a terraform provider to interact with it. 

It's not unheard of for a company to manage everything from onboarding users to github, their Cloud Computing estate and even on-premese compute 
using one single github repo and one `terraform apply` command. So we want to offer OpenFaaS users the ability to manage their functions in the same way too.


## Installing Terraform

We are going to use Terraform version 0.15.3, so please install this version. We can use [arkade](get-arkade.dev) to download this version and it will get the correct binary for your operating system.

```shell
# Install arkade
# Note: you can also run without `sudo` and move the binary yourself
curl -sLS https://dl.get-arkade.dev | sudo sh

# Install terraform at the required version
arkade get terraform --version 0.15.3

# Add (terraform) to your PATH variable
export PATH=$PATH:$HOME/.arkade/bin/

# Test the binary:
$HOME/.arkade/bin/terraform

# Or install with:
sudo mv /$HOME/.arkade/bin/terraform /usr/local/bin/
```

## Provisioning a Kubernetes cluster

Sign up for a [Civo account](https://civo.com), at time of writing new users get $250 credit to use on their new platform, which is more than 
enough to run this infrastructure for a few months for free. 

Once you have an account, navigate to [Account/Security](https://www.civo.com/account/security) where you should be able 
to get a hold of your API Key. 

Copy this key and set it as an environment variable.

```shell
# Just in this shell
export TF_VAR_civo_token=<your api key>

#In every shell from now onwards (this is for bash shell only!)
echo "export TF_VAR_civo_token=<your api key>" >> ~/.bashrc
source ~/.bashrc
```

Now we can start writing our Terraform code to spin up a kubernetes cluster on the Civo platform

create a new directory on your computer called `openfaas-terraform` and create a file called `providers.tf`

```hcl
variable "civo_token" {}

terraform {
  required_providers {
    civo = {
      source  = "civo/civo"
      version = "0.10.0"
    }
  }
}

# Configure the Civo Provider
provider "civo" {
  token  = var.civo_token
  region = "LON1"
}

```

You can see from the above configuration that I have selected the `LON1` region, feel free to select another region if
you prefer.

We are now going to use the configured Civo provider to configure a kubernetes cluster. 

Create a file called `kubernetes.tf` with the following contents

```hcl 
resource "civo_kubernetes_cluster" "this" {
    name = "openfaas-cluster"
    applications = ""
    num_target_nodes = 3
    target_nodes_size = "g3.k3s.small"
}
```

This terraform code will launch a new Civo kubernetes cluster using 3 "g3.k3s.small" nodes, you can increase the number of nodes
or size of node by changing the config above. However, for our application the settings should be ok.



## Install OpenFaaS with Helm & Kubernetes

Next up we are going to use the helm terraform provider to install OpenFaaS on Kubernetes. We also need to create some 
secrets to use to secure the OpenFaaS API endpoint, we use the `kubernetes provider` for this.

We are going to add some more `providers` to our `providers.tf` file, so we can keep them all in one place. 
So open the `providers.tf` file and replace the contents with the configuration below.

```hcl 
variable "civo_token" {}

terraform {
  required_providers {
    civo = {
      source  = "civo/civo"
      version = "0.10.0"
    }
    helm = {
      source = "hashicorp/helm"
      version = "2.1.2"
    }
    kubernetes = {
      source = "hashicorp/kubernetes"
      version = "2.1.0"
      }
  }
}

# Configure the Civo Provider
provider "civo" {
  token  = var.civo_token
  region = "LON1"
}

provider "helm" {
  kubernetes {
    host  = civo_kubernetes_cluster.my-cluster.api_endpoint
    username = yamldecode(civo_kubernetes_cluster.my-cluster.kubeconfig).users[0].user.username
    password = yamldecode(civo_kubernetes_cluster.my-cluster.kubeconfig).users[0].user.password
    cluster_ca_certificate = base64decode(
      yamldecode(civo_kubernetes_cluster.my-cluster.kubeconfig).clusters[0].cluster.certificate-authority-data
    )
  }
}

provider "kubernetes" {
    host  = civo_kubernetes_cluster.my-cluster.api_endpoint
    username = yamldecode(civo_kubernetes_cluster.my-cluster.kubeconfig).users[0].user.username
    password = yamldecode(civo_kubernetes_cluster.my-cluster.kubeconfig).users[0].user.password
    cluster_ca_certificate = base64decode(
      yamldecode(civo_kubernetes_cluster.my-cluster.kubeconfig).clusters[0].cluster.certificate-authority-data
    )
}
```

Now we have the helm and kubernetes providers configured, we can create helm releases in our cluster.

Create a new file called `helm.tf` 
```hcl 


```