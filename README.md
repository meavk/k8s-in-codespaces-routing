# Kubernetes in Codespaces

> Setup a Kubernetes cluster using `k3d` running in [GitHub Codespaces](https://github.com/features/codespaces)

![License](https://img.shields.io/badge/license-MIT-green.svg)

## Overview

This repo is to demonstrate the use of Kubernetes in Codespaces in a production like scenario where an ingress is setup for routing.

> Refer to the original `Kubernetes in Codespaces` [repository](https://github.com/cse-labs/kubernetes-in-codespaces) to understand more about the setup and usage.

## Onboarding another application

A repository created from Kubernetes in Codespaces template pulls a `dotnet` based application, *IMDB App*, that's pulled from [here](https://github.com/cse-labs/imdb-app). While starting the codespace, this application is built and hosted in a `k3d` cluster running in Codespaces.

The following sections describe how a new application, which consumes one of the APIs in *IMDB App* can be onboarded into this setup.

### Creating a codespace and running IMDB App

1. Use the template [here](https://github.com/cse-labs/kubernetes-in-codespaces) and create a Github repository.

2. Create a codespace on the main branch.

3. Once the cocespace is created and configured, the IMDB App should be running. Check this either in the container creation logs or by running `kic pods`.

4. Make sure that the IMDB App is running in Codespace by navingating to the nodeport exposed throigh port `30080`. Swagger documentation should be visible on the homepage.

5. Navigate to `/api/movies` and check if returns a list of movies.

### Adding IMDB UI application

1. Modify `.devcontainer/on-create.sh` to clone `imdb-ui` and restore packages by adding following lines to respective sections.

    ```bash
    # clone repos
    # -- clone other repos --
    git clone https://github.com/meavk/imdb-ui /workspaces/imdb-ui

    # restore the repos
    # -- restore other projects --
    dotnet restore /workspaces/imdb-ui/src/Imdb.BlazorWasm/Imdb.BlazorWasm.csproj
    ```

2. Modify `.devcontainer/post-create.sh` to update the repo

    ```bash
    # update the repos
    # -- update other repos --
    git -C /workspaces/imdb-ui pull
    ```

3. Add a new command to build and deploy IMDB UI by creating new file `imdb-ui` in `cli/.kic/commands/build` folder.

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

4. Add this command to `kic build` scripts list by adding following lines to  `root.yaml` in `cli/.kic` folder.

    ```yaml
    - name: imdb-ui
      short: Build the IMDb UI app
      path: build/imdb-ui
    ```

5. Configure `.devcontainer/on-create.sh`to invoke IMDB build command.

    ```bash
    echo "bilding IMDb UI"
    kic build imdb-ui
    ```
