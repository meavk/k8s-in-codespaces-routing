# Kubernetes in Codespaces

> Setup a Kubernetes cluster using `k3d` running in [GitHub Codespaces](https://github.com/features/codespaces)

![License](https://img.shields.io/badge/license-MIT-green.svg)

## Overview

This repo is to demonstrate the use of Kubernetes in Codespaces in a production like scenario where an ingress is setup for routing.

> Refer to the original `Kubernetes in Codespaces` [repository](https://github.com/cse-labs/kubernetes-in-codespaces) to understand more about the setup and usage.

## Creating a codespace and running IMDB App

When a repository is created from Kubernetes in Codespaces template, it pulls a `dotnet` based application, *IMDB App*, from [here](https://github.com/cse-labs/imdb-app). While starting the codespace, this application is built and hosted in a `k3d` cluster running in Codespaces.

1. Use the template [here](https://github.com/cse-labs/kubernetes-in-codespaces) and create a Github repository.

2. Create a codespace on the main branch or a new branch from main.

3. Once the cocespace is created and configured, the IMDB App should be running. Check this either in the container creation logs or by running `kic pods` from a `zsh` terminal.

   > Wait for `onCreate`, `postCreate` and `postStart` commands to finish before checking.

4. Make sure that the IMDB App is running in Codespace by navingating to the nodeport exposed through Codespaces port `30080`. Swagger documentation should be visible on the homepage.
    ![image](https://user-images.githubusercontent.com/32096756/182039350-a84c910d-92ca-44e2-82c5-e2c9630202fd.png)

5. Navigate to `/api/movies` and check if returns a list of movies.

## Onboarding other applications

The following sections describe how another application can be added to this environment.

The first scenario is talking about *IMDB UI*, which consumes one of the APIs in *IMDB App*. In this scenario, the other app is being developed in a separate repository.

### Adding an application from another repository

> All changes mentioned in below steps are available in branch [external-app-onboarding](https://github.com/meavk/k8s-in-codespaces-routing/tree/external-app-onboarding) as part of commit [Onboard meavk/imdb-ui application](https://github.com/meavk/k8s-in-codespaces-routing/commit/0bb05f8b86a2f3b0f9b3bc0b3d7049eca89789c9) for reference.

1. First step is to clone that repository to Kubernetes in Codespaces repositoy. This can be done by adding few lines of code to `.devcontainer/on-create.sh` to clone `imdb-ui` and restore packages.

    ```bash
    # clone repos
    # -- clone other repos --
    git clone https://github.com/meavk/imdb-ui /workspaces/imdb-ui

    # restore the repos
    # -- restore other projects --
    dotnet restore /workspaces/imdb-ui/src/Imdb.BlazorWasm/Imdb.BlazorWasm.csproj
    ```

2. You'd also need to keep it up to date. Modify `.devcontainer/post-create.sh` for this.

    ```bash
    # update the repos
    # -- update other repos --
    git -C /workspaces/imdb-ui pull
    ```

3. As the new app is already dockerized, next step is to create a YAML file `imdb-ui.yaml` for IMDB UI `deployment` and `service` in a new folder `deploy/apps/imdb-ui` for the app. Note that the service listens on `node port` `30090`.

   ```YAML
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: imdb-ui
     namespace: imdb
     labels:
       app.kubernetes.io/name: imdb-ui
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: imdb-ui
     template:
       metadata:
         labels:
           app: imdb-ui
       spec:
         containers:
           - name: app
             image: k3d-registry.localhost:5500/imdb-ui:local
             imagePullPolicy: Always

             ports:
               - name: http
                 containerPort: 80
                 protocol: TCP

             resources:
               limits:
                 cpu: 1000m
                 memory: 256Mi
               requests:
                 cpu: 200m
                 memory: 64Mi

   ---

   apiVersion: v1
   kind: Service
   metadata:
     name: imdb-ui
     namespace: imdb
   spec:
     type: NodePort
     ports:
       - port: 9080
         nodePort: 30090
         targetPort: http
         protocol: TCP
         name: http
     selector:
       app: imdb-ui
   ```

4. The `kic` CLI has `build` commands for building and deploying the apps. Add one to build and deploy IMDB UI by creating new file `imdb-ui` in `cli/.kic/commands/build` folder. It can be seen that it's mostly a copy paste of the same for `imdb` except for a few changes.

    ```bash
    #!/bin/bash

    #name: imdb-ui
    #short: Build and deploy the IMDb UI to the local cluster

    cd "$REPO_BASE" || exit

    # validate directories
    if [ ! -d /workspaces/imdb-ui ]; then echo "/workspaces/imdb-ui directory not found. Please clone the imdb-ui repo to /workspaces"; exit 1; fi
    if [ ! -d deploy/apps/imdb-ui ]; then echo "$REPO_BASE/deploy/apps/imdb-ui directory not found. Please cd to an appropriate directory"; exit 1; fi

    # delete webv and imdb
    kubectl delete -f deploy/apps/imdb-ui --ignore-not-found=true --wait=false

    # build and push the local image for imdb-ui
    docker build /workspaces/imdb-ui/src/Imdb.BlazorWasm -t k3d-registry.localhost:5500/imdb-ui:local
    docker push k3d-registry.localhost:5500/imdb-ui:local

    # wait for delete to finish
    kubectl wait pod -l app=imdb-ui -n imdb --for delete --timeout=30s

    # deploy local app and re-deploy webv
    kubectl apply -f deploy/apps/imdb-ui
    kubectl wait pod -l app=imdb-ui -n imdb --for condition=ready --timeout=30s
    ```

5. This command needs to be added `kic build` scripts list by adding following lines to `root.yaml` in `cli/.kic` folder.

    ```yaml
    - name: imdb-ui
      short: Build the IMDb UI app
      path: build/imdb-ui
    ```

6. Add following lines to `.devcontainer/on-create.sh`, right after `kic build imdb`, to invoke `imdb-ui` build command, so that IMDB UI is built as the container gets created.

    ```bash
    echo "building IMDb UI"
    chmod +x /workspaces/k8s-in-codespaces-routing/cli/.kic/commands/build/imdb-ui
    kic build imdb-ui
    ```

7. Lastly, the app needs to be exposed to internet. Codespaces has port-forwarding to expose services that are running in Codespaces. In earlier step while creating a `service` for IMDB-UI, you might have noticed a `nodePort` being assigned. Since this is a `k3d` node, first step is to configure `k3d` to expose it. This can be done by adding a new entry to the `ports` section in `.devcontainer/k3d.yaml`.

    ```yaml
      - port: 30090:30090
        nodeFilters:
        - server[0]
    ```

8. Finally, enable port forwarding by adding port `30090` to respective sections in `.devcontainer/devcontainer.json`.

    ```json
      "forwardPorts": [
        30000,
        30080,
        30090,
        31080,
        32000
      ],
      "portsAttributes": {
        "30000": { "label": "Prometheus" },
        "30080": { "label": "IMDb App" },
        "30090": { "label": "IMDb UI" },
        "31080": { "label": "Heartbeat" },
        "32000": { "label": "Grafana" }
      },
    ```

9. Rebuild codespace by using the command `Codespaces: Rebuild Container` from Command Palette (`Ctrl+Shift+P`) for the changes to take effect.

10. Once the codespace is built and pods are running, `Ports` section should show a new entry `IMDb UI (30090)`. Click on the link to browse the application.
    > Use `kic pods` command or check creation logs as did earlier to verify pods, including `imdb-ui`, are running.
    
    ![image](https://user-images.githubusercontent.com/32096756/182041728-5c70f99b-d125-441b-bd25-0b9c53971f3d.png)



