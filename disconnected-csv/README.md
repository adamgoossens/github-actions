# disconnected-csv

This GitHub Action processes a `ClusterServiceVersion` YAML file generated by `operator-sdk` tooling to make it suitable for disconnected environments.

Specifically, it will:

* Build `spec.relatedImages`.
* Convert tags to digest references, if configured.
* Add annotations required to the `ClusterServiceVersion` metadata.

## Selecting related images

The scripts will consider the following two sources for potential related images:

1. `spec.containers[*].image` for each container in each `Deployment`.
2. Any environment variable, in any container, with a prefix defined in the `RELATED_IMAGE_ENV_PREFIX` input.

If no environment variables referencing container image refs are found, the action will raise the following note:

```
*************
NOTE - no related images found injected as environment variables prefixed with ${RELATED_IMAGE_ENV_PREFIX}.

If your Operator deploys an application container, it is recommended that you inject the container image reference
as an environment variable, and define this environment variable in your Operator Deployment definition 
in your ClusterServiceVersion.

Example:

containers:
  - name: operator
    image: quay.io/yourrepo/your-operator:latest
    env:
    - name: ${RELATED_IMAGE_ENV_PREFIX}APPLICATION
      value: quay.io/anotherrepo/application-image:latest

That will allow this Action to pick up your injected container reference and process it as a related image.

If your Operator does not deploy any application containers, you can safely ignore this message.
*************
```

## Configuration

The following inputs can be configured:

| Action Input             | Description                                                                               | Required? (Y/N)| Default        |
| ------------------------ | ----------------------------------------------------------------------------------------- | :-------------:| -------------- |
| CSV_FILE                 | A path to the `ClusterServiceVersion` YAML file to be processed.                          | Y              | -              |
| RELATED_IMAGE_ENV_PREFIX | A prefix to use to select container environment variables for related images.             | N              | RELATED_IMAGE_ |
| TAGS_TO_DIGESTS          | If set to any value, tag references will be converted in-place to digest references. Omit to leave tag refs unchanged.      | N              | Unset          |
| OPERATOR_CONTAINER_NAME  | Which container refers to the Operator.                                                   | N              | manager        |

## Example usage

Examples for using this within a GitHub Actions workflow:

```
jobs:
  release-without-digests:
    runs-on: ubuntu-latest
    steps:
      - name: process bundle for disconnected support
        uses: adamgoossens/github-actions/disconnected-csv@main
        with:
          CSV_FILE: bundle/manifests/my-operator.clusterserviceversion.yaml

  bundle-with-digests:
    runs-on: ubuntu-latest
    steps:
      - name: process bundle for disconnected support
        uses: adamgoossens/github-actions/disconnected-csv@main
        with:
          CSV_FILE: bundle/manifests/my-operator.clusterserviceversion.yaml
          TAGS_TO_DIGESTS: 1

  custom-env-prefix:
    runs-on: ubuntu-latest
    steps:
      - name: process bundle for disconnected support
        uses: adamgoossens/github-actions/disconnected-csv@main
        with:
          CSV_FILE: bundle/manifests/my-operator.clusterserviceversion.yaml
          RELATED_IMAGE_ENV_PREFIX: "my_custom_prefix_"
```
## Example Output

```
Finding image references from container definitions...
Finding additional relatedImages as environment variables prefixed with "RELATED_IMAGE_"...

The following refs were found:
   docker.io/memcached:1.4.36-alpine
   gcr.io/kubebuilder/kube-rbac-proxy:v0.8.0
   quay.io/agoossen/memcached-operator:latest

Processing docker.io/memcached:1.4.36-alpine...
  Adding relatedImage...
  Processing tag to digest conversion...
  Digest is docker.io/memcached@sha256:00b68b00139155817a8b1d69d74865563def06b3af1e6fc79ac541a1b2f6b961
Processing gcr.io/kubebuilder/kube-rbac-proxy:v0.8.0...
  Adding relatedImage...
  Processing tag to digest conversion...
  Digest is gcr.io/kubebuilder/kube-rbac-proxy@sha256:db06cc4c084dd0253134f156dddaaf53ef1c3fb3cc809e5d81711baa4029ea4c
Processing quay.io/agoossen/memcached-operator:latest...
  Adding relatedImage...
  Processing tag to digest conversion...
  Digest is quay.io/agoossen/memcached-operator@sha256:fe71cd9b17dfe8c7b07c452b2184a520435b59ed838ff52ca69014d6837f22d9

relatedImages completed. Adding annotations...
Done! Resulting CSV:

apiVersion: operators.coreos.com/v1alpha1
kind: ClusterServiceVersion
metadata:
  annotations:
...
     createdAt: "2021-05-08T06:21:50Z"
    containerImage: quay.io/agoossen/memcached-operator@sha256:fe71cd9b17dfe8c7b07c452b2184a520435b59ed838ff52ca69014d6837f22d9
  name: memcached-operator.v0.0.1
  namespace: placeholder
spec:
...
  relatedImages:
    - name: docker.io/memcached
      image: docker.io/memcached@sha256:00b68b00139155817a8b1d69d74865563def06b3af1e6fc79ac541a1b2f6b961
    - name: gcr.io/kubebuilder/kube-rbac-proxy
      image: gcr.io/kubebuilder/kube-rbac-proxy@sha256:db06cc4c084dd0253134f156dddaaf53ef1c3fb3cc809e5d81711baa4029ea4c
    - name: quay.io/agoossen/memcached-operator
      image: quay.io/agoossen/memcached-operator@sha256:fe71cd9b17dfe8c7b07c452b2184a520435b59ed838ff52ca69014d6837f22d9
```