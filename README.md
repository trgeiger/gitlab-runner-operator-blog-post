# Companion files for "Creating an Image Build Pipeline with the Certified GitLab Runners Operator and Buildah"

Template files for setting up a simple build pipeline for custom Fedora Silverblue images with the GitLab Runners Operator for OpenShift.

Included files:
- `gitlab-runner-secret.yml` for creating a Secret with GitLab registration token
- `config.toml` for creating a ConfigMap with your runner configuration
- `gitlab-runner.yml` Custom Resource Definition for declaring a GitLab Runner using our Secret and ConfigMap
- `gitlab-ci.yml` for defining our image build job with Buildah
- `Containerfile` for our custom Silverblue image

References:
- [GitLab Runners Documentation](https://docs.gitlab.com/ee/)
- [Fedora Silverblue Documentation](https://docs.fedoraproject.org/en-US/fedora-silverblue/)
- [Red Hat Certified GitLab Runner Operator](https://catalog.redhat.com/software/container-stacks/detail/5e9877e96c5dcb34dfbb1ac9)
- [Universal Blue: Community built OS images based on Fedora](https://github.com/ublue-os)
