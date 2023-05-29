# GitLab event integration example

The repository contains examples of how to integrate GitLab events into Direktiv for pipeline extensions or CI/CD+ workflows.

## Pre-requisites

To allow GitLab events to be received by Direktiv, the GitLab listener would need to be installed. The receiver/listener provided by Kantive can also be used, but this requires a users' Kubernetes API to be exposed and reachable from GitLab. Alternatively, the GitLab receiver provided by Direktiv will listen on the Direktiv URL with a custom path (example `prod.direktiv.io/gitlab/`). The GitLab listener is available at https://github.com/direktiv/direktiv-listeners/tree/main/gitlab-receiver. To install the GitLab listener, follow the steps below:

1. Create the following route for the apisix gateway (`gitlab-apisix-route.yaml`):

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: gitlab-receiver-route
spec:
  http:
  - name: gitlab-receiver
    match:
      hosts:
      - <direktiv-url i.e prod.direktiv.io> # direktiv url
      paths:
      - "/gitlab/*" # path for the webhook endpoint
    backends:
    - serviceName: gitlab-receiver-service
      servicePort: 8080
```

2. Apply the GitLab APISIX route on the ingress control to enable access to the GitLab running receiver:

```sh
root@direktiv-1:~/# kubectl apply -f gitlab-apisix-route.yaml
apisixroute.apisix.apache.org/gitlab-receiver-route created
root@direktiv-1:~/#
```

3. Create the following 

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gitlab-receiver-cfg-cm
data:
  config.yaml: |
    server:
      bind: ":8080"
      tls: false

    gitlab:
      token: "glpat-MaY937VkLyzgz-cV2Wbq" # this is the shared token created in GitLab

    direktiv:
      endpoint: https://<url>/api/namespaces/<namespace>/broadcast # specify the hostname and namespace
      insecureSkipVerify: true
      token: "v4.local.token"
      event-on-error: true
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitlab-receiver
  labels:
    app: gitlab-receiver
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gitlab-receiver
  template:
    metadata:
      annotations:
        linkerd.io/inject: disabled
      labels:
        app: gitlab-receiver
    spec:
      volumes:
      - name: gitlabconf
        configMap:
          name: gitlab-receiver-cfg-cm
      securityContext:
        runAsNonRoot: true
        runAsUser: 65532
        runAsGroup: 65532
      containers:
        - name: gitlab-receiver
          image: gcr.io/direktiv/listeners/gitlab-receiver:1.0
          imagePullPolicy: Always
          ports:
          - containerPort: 8080
          volumeMounts:
          - name: gitlabconf
            mountPath: "/config"
            readOnly: false
---
apiVersion: v1
kind: Service
metadata:
  name: gitlab-receiver-service
spec:
  selector:
    app: gitlab-receiver
  ports:
    - port: 8080
```

4. Install the service on the Kubernetes platform using the file create or the `kuberetes/install.yaml`:

```sh
kubectl apply -f kubernetes/install.yaml
```

## Workflow overview

The workflow will clone a GitLab repository whenever a "Push Hook" event is received. The workflow has the following steps after receiving the event:

1. Clone the GitLab repository into a Direktiv instance variable using the `git` operation
2. Run the `yq` command to convert the YAML configuration for the GitLab pipeline into a JSON object which can be used to drive the next steps in the workflow
3. Compile and execute the Python scripts in the repository we cloned in step 1
4. Generate an event to notify any messaging workflows that they need to publish the output of the Python script
5. Wait for the completion events and close the workflow.

## Variables

 - None

## Secrets

 - GITLAB_PAT: GitLab Personal Access Token (PAT)

## Namespace Services

 - None

## Input examples

```json
Context Attributes,
  specversion: 1.0
  type: Push Hook
  source: https://gitlab.com/wwonigkeit/dcs-poc
  id: 929e33e0-dc47-40e7-9ce0-6f74bd8b3134
  time: 2023-05-11T20:38:33.31799776Z
  datacontenttype: application/json
Extensions,
  traceparent: 00-be263658dccd5ece7bf8b5c8006a2802-7a3f8155e0e96cac-00
Data,
  {
    "ref": "refs/heads/main",
    "after": "ab8d5a6d80730cd40315a1e61d6027fb60dc3e50",
    "before": "65bc52d802a441705e42a2d9786939dcc64a37f4",
    "commits": [
      {
        "id": "ab8d5a6d80730cd40315a1e61d6027fb60dc3e50",
        "url": "https://gitlab.com/wwonigkeit/dcs-poc/-/commit/ab8d5a6d80730cd40315a1e61d6027fb60dc3e50",
        "added": [],
        "title": "Test event integration",
        "author": {
          "name": "Wilhelm Wonigkeit",
          "email": "wilhelm.wonigkeit@direktiv.io"
        },
        "message": "Test event integration\n",
        "removed": [],
        "modified": [
          "README.md"
        ],
        "timestamp": "2023-03-13T16:11:44+11:00"
      },
      {
        "id": "2fbf3bc7a61a9effce49f34ed5f5617a15d3aacd",
        "url": "https://gitlab.com/wwonigkeit/dcs-poc/-/commit/2fbf3bc7a61a9effce49f34ed5f5617a15d3aacd",
        "added": [
          "dbo/FUNCTIONS/.gitkeep"
        ],
        "title": "Merge remote-tracking branch 'refs/remotes/origin/main'",
        "author": {
          "name": "Wilhelm Wonigkeit",
          "email": "wilhelm.wonigkeit@direktiv.io"
        },
        "message": "Merge remote-tracking branch 'refs/remotes/origin/main'\n",
        "removed": [],
        "modified": [],
        "timestamp": "2023-03-13T14:11:31+11:00"
      },
      {
        "id": "65bc52d802a441705e42a2d9786939dcc64a37f4",
        "url": "https://gitlab.com/wwonigkeit/dcs-poc/-/commit/65bc52d802a441705e42a2d9786939dcc64a37f4",
        "added": [],
        "title": "Commit 13/03/2023",
        "author": {
          "name": "Wilhelm Wonigkeit",
          "email": "wilhelm.wonigkeit@direktiv.io"
        },
        "message": "Commit 13/03/2023\n",
        "removed": [],
        "modified": [
          "README.md"
        ],
        "timestamp": "2023-03-13T14:11:02+11:00"
      }
    ],
    "message": null,
    "project": {
      "id": 44199177,
      "url": "git@gitlab.com:wwonigkeit/dcs-poc.git",
      "name": "dcs-poc",
      "ssh_url": "git@gitlab.com:wwonigkeit/dcs-poc.git",
      "web_url": "https://gitlab.com/wwonigkeit/dcs-poc",
      "homepage": "https://gitlab.com/wwonigkeit/dcs-poc",
      "http_url": "https://gitlab.com/wwonigkeit/dcs-poc.git",
      "namespace": "wwonigkeit",
      "avatar_url": null,
      "description": null,
      "git_ssh_url": "git@gitlab.com:wwonigkeit/dcs-poc.git",
      "git_http_url": "https://gitlab.com/wwonigkeit/dcs-poc.git",
      "ci_config_path": "",
      "default_branch": "main",
      "visibility_level": 0,
      "path_with_namespace": "wwonigkeit/dcs-poc"
    },
    "user_id": 13633436,
    "user_name": "Wilhelm Wonigkeit",
    "event_name": "push",
    "project_id": 44199177,
    "repository": {
      "url": "git@gitlab.com:wwonigkeit/dcs-poc.git",
      "name": "dcs-poc",
      "homepage": "https://gitlab.com/wwonigkeit/dcs-poc",
      "description": null,
      "git_ssh_url": "git@gitlab.com:wwonigkeit/dcs-poc.git",
      "git_http_url": "https://gitlab.com/wwonigkeit/dcs-poc.git",
      "visibility_level": 0
    },
    "user_email": null,
    "object_kind": "push",
    "user_avatar": "https://secure.gravatar.com/avatar/ed86c6b6063b39405f37de55d54d2f07?s=80&d=identicon",
    "checkout_sha": "ab8d5a6d80730cd40315a1e61d6027fb60dc3e50",
    "push_options": {},
    "user_username": "wilhelm.wonigkeit",
    "total_commits_count": 3
  }
```

## External links

The following blog has more information on this repository:

https://blog.direktiv.io/extending-or-replacing-gitlab-github-pipelines-with-event-driven-direktiv-workflows-fabbfb418fc


