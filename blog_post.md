![A metal pipeline junction in front of a blue sky](./pipeline.jpg)

Tayler Geiger

Continuous integration and continuous deployment (CI/CD) pipelines have become a crucial part of modern software development, allowing developers to build, test, and deploy code changes quickly and efficiently. By automating the process of building and deploying applications, CI/CD pipelines can help teams improve code quality, reduce errors, and speed up the time-to-market for new features and applications.

GitLab Runners are the applications that process CI/CD jobs on GitLab.com and self-hosted GitLab instances. GitLab.com provides their own hosted runners that are shared amongst users of the site, but you can also set up dedicated private runners for your projects through several different installation types. For enterprises or individuals that like to manage their own pipeline infrastructure in the cloud, the GitLab Runner Operator, available as a Red Hat Certified Operator in Openshift, provides a fast and easy cloud-native installation of GitLab Runners.

As a way to showcase the simplicity and functionality of the GitLab Runner Operator, I thought it would be fun to also highlight some exciting developments in Fedora Linux and use the GitLab Runner Operator to build customized base images for Fedora Silverblue.

Fedora Silverblue is a cutting-edge immutable distribution of Fedora Linux. Recently, its hybrid image and package management system `rpm-ostree` has gained the ability to boot from OCI containers. This allows users or enterprises to build their own customized base images with the familiar workflow of Dockerfiles and Containerfiles.

In the following tutorial, we will set up a fully automated build system for our Fedora Silverblue images using the GitLab Runner Operator.

### Prerequisites

You will need a few previously configured and installed resources that we will not be covering in the tutorial:
- An OpenShift cluster
- An existing Fedora Silverblue installation (if you want to actually use the custom images)
- A GitLab.com account

## Installing and Configuring the GitLab Runner Operator

Open the Operators heading in the sidebar of your Openshift cluster and click on __OperatorHub__. Search for the GitLab Runner operator and click the Certified option (the Community version of the operator is also available but this tutorial will stick to the Certified on Openshift variant). Before clicking __Install__, note the Prerequisites section of the operator’s description. The GitLab Runner Operator requires you first install `cert-manager`:

```bash
oc apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.7.1/cert-manager.yaml
```

After doing so, click __Install__ to install the GitLab Runner Operator.

On the next screen you will be prompted to change any of the default installation options. If you need or want to scope to a specific namespace do so now, otherwise continue with the default options. I also suggest leaving __Update approval__ set to __Automatic__ since that's one key benefit of using an operator.

Allow the installation a moment to finish up and then navigate to your __Installed Operators__ and then the GitLab Runner Operator. Here you will see the same info text from earlier, as well as a link under __Provided APIs__ where you can create a Runner instance. In the info text below are instructions for linking your Runners to your GitLab repositories, which we will follow now.

## Creating and registering our Runner instance

### 1. Creating the registration token Secret

First, we must create a Secret with a registration token from GitLab which our new Runner will use to register with our repository. Open GitLab.com or your private GitLab instance and then open the repository you want to register. Navigate to Settings and then CI/CD. Expand the section titled __Runners__ and then look to the section __Project runners__. This is where we will find our registration token and the URL you will use to register your Runner instance.

Also note the __Shared runners__ section if you're using GitLab.com. These are public runners provided by GitLab.com. Disable shared runners for this project since we will be using our own private runners.

Create your Secret, inserting your repository's registration token into the `runner-registration-token` field. You can do this in the web console or through the terminal interface. I created a file named `gitlab-runner-secret.yml` and then added it to my cluster:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-runner-secret
type: Opaque
stringData:
  runner-registration-token: YOUR_TOKEN_HERE
```

```bash
oc apply -f gitlab-runner-secret.yml
```

### 2. Configuring the runner with a ConfigMap

Before we create our runner, we also need to create a ConfigMap with some customization options. GitLab Runners can normally be configured with a `config.toml` file. In a Kubernetes context we can add those [customizations](https://docs.gitlab.com/runner/configuration/configuring_runner_operator.html#customize-configtoml-with-a-configuration-template) with a ConfigMap that contains our `config.toml` file.

Since we are running our build containers in the environment of an OpenShift cluster, we need to ensure the build containers run without escalated privileges and [as a non-root user](https://docs.gitlab.com/runner/configuration/configuring_runner_operator.html#root-vs-non-root) (Note: [there are ways around](https://docs.gitlab.com/runner/configuration/configuring_runner_operator.html#root-vs-non-root) this if you really know you need a privileged build environment, but we will be sticking with a non-root setup). Here's a simple `config.toml` that specifies the runner pod will run as non-root user 1000:
```toml
[[runners]]
  name = "gitlab-runner"
  url = "https://gitlab.com"
  executor = "kubernetes"
  [runners.kubernetes]
    [runners.kubernetes.pod_security_context]
      run_as_non_root = true
      run_as_user = 1000
```

To add this to our cluster as a ConfigMap:
```bash
oc create configmap my-runner-config --from-file=config.toml
```

### 3. Starting the Runner instance

Now we can create our actual Runner instance. Again, you can do this with the web console by clicking __Create instance__ on the GitLab Runner Operator page, or through the terminal. Either way, we want to ensure that our Custom Resource Definition includes the correct GitLab URL, the name of our registration token Secret, the name of our ConfigMap, and that it includes "openshift" in the tags (this last item of the "openshift" tag is required for jobs to be passed to your cluster). Here is a basic CRD named `gitlab-runner.yml` which fulfills all our criteria:
```yaml
apiVersion: apps.gitlab.com/v1beta2
kind: Runner
metadata:
 name: gitlab-runner
spec:
 gitlabUrl: https://gitlab.com
 token: gitlab-runner-secret
 tags: openshift
 config: my-runner-config
```

To install to our cluster:
```bash
oc apply -f gitlab-runner.yml
```

You can now check the status of your new runner in the web console or through the terminal with `oc get runners`. We should also check in our GitLab project's CI/CD settings to ensure the runner properly linked to our repository. You should now see a runner under the heading __Assigned project runners__ with the same name as the CRD we created and installed.

## Using our Runner to Build Silverblue Images

### 1. Defining the GitLab CI/CD pipeline jobs

Now that our runner is installed, configured, and linked to our project, we can the write the [GitLab CI](https://docs.gitlab.com/ee/ci/quick_start/index.html) file that will define our image build. GitLab provides many [examples](https://docs.gitlab.com/ee/ci/examples/index.html) of their `gitlab-ci.yml` file structure for different project types. We will be writing our own, utilizing [`buildah`](https://buildah.io/) to build our Silverblue images.

The `gitlab-ci.yml` we will use looks as follows:
```yml
stages:
  - build

buildah-build:
  stage: build
  tags:
    - openshift
  image: quay.io/buildah/stable
  variables:
    STORAGE_DRIVER: vfs
  script:
    - export HOME=/tmp
    - buildah login --username "$CI_REGISTRY_USER" --password "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
    - buildah build -t "$CI_REGISTRY_IMAGE:latest" .
    - buildah push "$CI_REGISTRY_IMAGE:latest"
    - buildah logout "$CI_REGISTRY"
```

There are several important elements of this file to note.
- We need to include at least the `openshift` tag, as previously mentioned, for our jobs to be picked up by the GitLab Runner Operator
- We are using the official stable Buildah image hosted in Quay.io as our build image.
- We set the storage driver to `vfs` due to [issues](https://github.com/containers/buildah/issues/1496) with overlay-on-overlay filesystems. Using `vfs` is the simplest solution for now.
- We change `$HOME` to `/tmp` to ensure we are able to write, since most container filesystems in OpenShift will be read-only.

The last section, `script`, is a list of the commands our build job will run. Here we utilize GitLab CI's [predefined variables](https://docs.gitlab.com/ee/ci/variables/predefined_variables.html) to sign into the GitLab container registry, build and tag our image, and finally push the image to the registry before logging out.

### 2. Writing the Containerfile for our custom Silverblue image

After adding the `gitlab-ci.yml` to our repository, we also need to add the `Containerfile` that we want to build. For our custom Fedora Silverblue image, I relied on the great work being done in the [Universal Blue](https://github.com/ublue-os) community. They have been working on various versions of Silverblue for different usecases and are an excellent resource if you or your team are interested in creating customized, immutable base images for Fedora Silverblue systems.

I thought it would be helpful to create a base image that includes the most recent releases of all the OpenShift and Operator tooling I use on a daily basis. Since we will be setting this to build daily, not only will my base image be updated with the most recent Fedora packages, but I will also never fall behind on the latest versions of these OpenShift and Operator tools that normally would require manual updates.

```Dockerfile
ARG FEDORA_MAJOR_VERSION=38

FROM quay.io/fedora-ostree-desktops/silverblue:${FEDORA_MAJOR_VERSION}
# Starting with Fedora 39, the new official location for these images is quay.io/fedora/fedora-silverblue
# See https://pagure.io/releng/issue/11047 for more information

# Install Openshift tools -- oc, opm, kubectl, operator-sdk, odo, helm, crc from official OpenShift sources
RUN curl -SL https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/latest/opm-linux.tar.gz | tar xvzf - -C /usr/bin
RUN curl -SL https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/operator-sdk/latest/operator-sdk-linux-x86_64.tar.gz | tar xvzf - --strip-components 2 -C /usr/bin
RUN curl -SL https://mirror.openshift.com/pub/openshift-v4/clients/oc/latest/linux/oc.tar.gz | tar xvzf - -C /usr/bin
RUN curl -SL https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/helm/latest/helm-linux-amd64 -o /usr/bin/helm && chmod +x /usr/bin/helm
RUN curl -SL https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/odo/latest/odo-linux-amd64 -o /usr/bin/odo && chmod +x /usr/bin/odo
RUN curl -SL https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/crc/latest/crc-linux-amd64.tar.xz | tar xfJ - --strip-components 1 -C /usr/bin

# Install awscli
RUN curl -SL https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip -o awscliv2.zip && unzip awscliv2.zip && ./aws/install --bin-dir /usr/bin --install-dir /usr/bin

# Install overrides and additions, remove lingering files
RUN rpm-ostree install podman-docker
RUN rm -rf aws && \
    rm -f awscliv2.zip && \
    rm -f /usr/bin/README.md && \
    rm -f /usr/bin/LICENSE
```

A short breakdown of what's happening in this Containerfile:
- To start, we specify the version of Fedora we want to build, in this case the current stable version 38
- Next, we specify the base image to use for our build, which is `silverblue:38` after our `FEDORA_MAJOR_VERSION` substitution (please see the comments in the Containerfile on the location of these images)
- In the largest section we run several `curl` commands to download all the OpenShift programs I want to install, making sure to put their binaries into the `/usr/bin` directory
- We also install `awscli` which involves unpacking the `.zip` file and running the installation script
- Lastly, we use `rpm-ostree` to install `podman-docker` from the Fedora package repositories (this essentially aliases any `docker` commands to `podman` for those of us used to running `docker`), and then delete some lingering files from the `awscli` extraction

As I mentioned before, for more Silverblue customization inspiration, please check out the community at [Universal Blue](https://github.com/ublue-os). I relied heavily on their work in learning this workflow and they're already working on many new exciting applications of the bootable OCI capability in Silverblue.

### 3. Using and monitoring the pipeline jobs

We should add this Containerfile, or whatever Containerfile or Dockerfile you would rather build, to the root of our project repository since our `gitlab-ci.yml` specifies that `buildah` use the current working directory as the Containerfile argument. Depending on how you have commited all of the changes in this repository, you might already have had notifications about failed pipeline jobs, or if you're commiting all of this at once, you will see the first attempt at running the build job start now. You can watch the logs for your build by navigating to the __CI/CD__ heading on your GitLab.com project page and then clicking either __Pipelines__ or __Jobs__ and clicking through until you see the log output for our job named `buildah-build`.

If we set everything up properly, you should see logs describing each step of the `gitlab-ci.yml` and Containerfile we wrote, concluding with "Job succeeded" when finished. If our job succeeds, then we should also be able to see our finished container in our project registry. On the left-hand navigation bar click __Packages and registries__ and then __Container Registry__. You should see a single image named after your project with a single tag "latest." This image will be located at `registry.gitlab.com/{YOUR_USERNAME}/{YOUR_REPOSITORY_NAME}`. 

The job as we wrote it will run with every commit to the `main` branch, but you can customize this behavior in the `gitlab-ci.yml` file. If we want to schedule regular builds, we can do so on our repository page in the CI/CD settings under the __Schedule__ section. Click __New schedule__ to configure the timing and frequency of your pipeline. You can learn more about GitLab.com pipeline scheduling [here](https://gitlab.com/help/ci/pipelines/schedules). For a custom Silverblue image you will probably want to build at least daily to match the build cadence of the official images.

## Using our Custom Images

To actually use this image as a bootable base for Silverblue, you first need to install Fedora Silverblue from the [official images](https://silverblue.fedoraproject.org/download) if you don't have an existing installation. After your installation is finished and you've logged in, we can rebase our installation onto our custom image.

*Please keep in mind* that booting from OCI images is still an experimental feature, so use your own best judgement and discretion if you're using this on personal or production machines. Silverblue is built to be resilient to breakage, though, so if your custom image doesn't work as expected you should be able to rollback to the original base image. To ensure we don't remove that original base, we can run `sudo ostree admin pin 0` to pin the current image. This ensures the reference is not lost on subsequent updates, as Silverblue normally only keeps the current and previous image references. To actually rebase to our custom image, we run `rpm-ostree rebase ostree-unverified-registry:registry.gitlab.com/{YOUR_USERNAME}/{YOUR_REPOSITORY_NAME}:latest` and then reboot.

After rebooting, we can verify we are running our custom image by looking at the output of `rpm-ostree status`. Your current deployment, identified by the circle/pip on the left-hand side, should show the `ostree-unverified-registry` URI of our custom image. You can also try running any of the OpenShift tools we added in a terminal, like `oc`, `operator-sdk`, or `helm`.

You should also see our pinned deployment with the old base reference in the output of `rpm-ostree status`. If you wish to rollback, just run `rpm-ostree rollback` and reboot. For more on Fedora Silverblue administration, please see the [documentation](https://docs.fedoraproject.org/en-US/fedora-silverblue/)

## Outcomes

Assuming everything went without a hitch, we now have a self-hosted CI/CD pipeline running on our own OpenShift cluster that regularly creates new custom Silverblue images. Our runners should require no manual intervention unless we wish to reconfigure build job tags or create more runners to handle more concurrent jobs. Builds will start at our scheduled intervals and when we commit new code to the `main` branch, and if we are running our custom image on a Silverblue installation we simple need to run `rpm-ostree update` to pull in our daily updates.

This tutorial example is a simple illustration of the capabilities of the GitLab Runner Operator and the GitLab CI/CD system. Both are able to manage much more sophisticated CI/CD needs, and by running the Certified GitLab Runner Operator on OpenShift you're able to cut out much of the manual set-up and maintenance of the runners themselves, freeing up your time and effort for the actual contents and configuration of your builds. 

I have created a reference repository with all the text files used in this tutorial [here](https://gitlab.com/trgeiger/gitlab-runner-image-build-examples). Please also see a list of useful references below.

- [GitLab Runners Documentation](https://docs.gitlab.com/ee/)
- [Fedora Silverblue Documentation](https://docs.fedoraproject.org/en-US/fedora-silverblue/)
- [Red Hat Certified GitLab Runner Operator](https://catalog.redhat.com/software/container-stacks/detail/5e9877e96c5dcb34dfbb1ac9)
- [Universal Blue: Community built OS images based on Fedora](https://github.com/ublue-os)
