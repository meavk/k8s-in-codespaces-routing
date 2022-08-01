# Kubernetes in Codespaces

> Setup a Kubernetes cluster using `k3d` running in [GitHub Codespaces](https://github.com/features/codespaces)

![License](https://img.shields.io/badge/license-MIT-green.svg)

## Overview

This repo is to demonstrate the use of Kubernetes in Codespaces in a production-like scenario where an `ingress` is set up for routing.

> Refer to the original `Kubernetes in Codespaces` [repository](https://github.com/cse-labs/kubernetes-in-codespaces) to understand more about the setup and usage.

## Creating a codespace and running IMDB App

When a repository is created from Kubernetes in Codespaces template, it pulls a `dotnet` based application, *IMDB App*, from [here](https://github.com/cse-labs/imdb-app). While starting the codespace, this application is built and hosted in a `k3d` cluster running in Codespaces.

1. Use the template [here](https://github.com/cse-labs/kubernetes-in-codespaces) and create a Github repository.

2. Create a codespace on the `main` branch or a new branch from `main`.

3. Once the codespace is created and configured, the IMDB App should be running. Check this either in the container creation logs or by running `kic pods` from a `zsh` terminal.

   > Wait for `onCreate`, `postCreate` and `postStart` commands to finish before checking.

4. Make sure that the IMDB App is running in Codespace by navigating to the nodeport exposed through Codespaces port `30080`. Swagger documentation should be visible on the homepage.
    ![image](https://user-images.githubusercontent.com/32096756/182039350-a84c910d-92ca-44e2-82c5-e2c9630202fd.png)

5. Navigate to `/api/movies` and check if returns a list of movies.

## Onboarding other applications

The following sections describe how another application can be added to this environment.

The first scenario is talking about *IMDB UI*, which consumes one of the APIs in *IMDB App*. In this scenario, the other app is being developed in a separate repository.

### Adding an application from another repository

> All changes mentioned in the below steps are available as part of commit [Onboard meavk/imdb-ui application](https://github.com/meavk/k8s-in-codespaces-routing/commit/0bb05f8b86a2f3b0f9b3bc0b3d7049eca89789c9) for reference.

1. The first step is to clone that repository to Kubernetes in Codespaces repository. This can be done by adding a few lines of code to `.devcontainer/on-create.sh` to clone `imdb-ui` and restore packages.

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

3. As the new app is already dockerized, the next step is to create a YAML file `imdb-ui.yaml` for IMDB UI `deployment` and `service` in a new folder `deploy/apps/imdb-ui` for the app. Note that the service listens on `node port` `30090`.

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

4. The `kic` CLI has `build` commands for building and deploying the apps. Add one to build and deploy IMDB UI by creating a new file `imdb-ui` in `cli/.kic/commands/build` folder. It can be seen that it's mostly a copy-paste of the same for `imdb` except for a few changes.

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

5. This command needs to be added `kic build` scripts list by adding the following lines to `root.yaml` in `cli/.kic` folder.

    ```yaml
    - name: imdb-ui
      short: Build the IMDb UI app
      path: build/imdb-ui
    ```

6. Add the following lines to `.devcontainer/on-create.sh`, right after `kic build imdb`, to invoke `imdb-ui` build command so that IMDB UI is built as the container gets created.

    ```bash
    echo "building IMDb UI"
    chmod +x /workspaces/k8s-in-codespaces-routing/cli/.kic/commands/build/imdb-ui
    kic build imdb-ui
    ```

7. The app needs to be exposed to the internet. Codespaces has port-forwarding to expose services that are running in Codespaces. In an earlier step, while creating a `service` for IMDB-UI, you might have noticed a `nodePort` being assigned. Since this is a `k3d` node, first step is to configure `k3d` to expose it. This can be done by adding a new entry to the `ports` section in `.devcontainer/k3d.yaml`.

    ```yaml
      - port: 30090:30090
        nodeFilters:
         - server[0]
    ```

8. Lastly, enable port forwarding by adding port `30090` to respective sections in `.devcontainer/devcontainer.json`.

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

11. Once the codespace is built and pods are running, `Ports` section should show a new entry `IMDb UI (30090)`. Click on the link to browse the application.
    > Use `kic pods` command or check creation logs as did earlier to verify pods, including `imdb-ui`, are running.
    
    ![image](https://user-images.githubusercontent.com/32096756/182041728-5c70f99b-d125-441b-bd25-0b9c53971f3d.png)

### About IMDB UI application

The UI for IMDB onboarded in the previous section is a [Blazor WebAssembly](https://docs.microsoft.com/en-us/aspnet/core/blazor/?view=aspnetcore-6.0#blazor-webassembly) application that runs completely in the browser. Navigate to the 'Movies' tab to see a list of movies.
![image](https://user-images.githubusercontent.com/32096756/182042339-c43dc404-6186-4513-bad1-365a05a98495.png)

You might remember from the earlier step that the response from `/api/movies` endpoint had a large number of movies in the list. However, the app is displaying only 2 movies. 

This is because the UI is showing a hardcoded list of movies as the request to `/api/movies` failed. 
![image](https://user-images.githubusercontent.com/32096756/182042527-4870f387-6e4e-4518-bc44-589de1c5b334.png)

And, application fetched a hard-coded list of movies from a `.json` file
   ```CSharp
        try
        {
            movies = await Http.GetFromJsonAsync<Movie[]>("api/movies");
        }
        catch (System.Exception)
        {
            movies = await Http.GetFromJsonAsync<Movie[]>("sample-data/movies.json");
        }
   ```
   
This is a possible real-world scenario. In the microservices world, subdomains and different paths on the same domain can be served by different applications. Kubernetes and other microservices frameworks provide such capabilities. It's only fair to expect Kubernetes in Codespaces also to provide such capability. The next section shows how Codespaces port-forwarding and a Kubernetes `ingress` can be leveraged to implement the same.

## Routing applications through an `ingress`

For codespaces, the `host` part of the URL will be dynamically generated by GitHub and it changes from Codespace to Codespace, and even for different ports in the same codespace. Look at the 'Ports' tab screenshot above, you'll see the port number being appended to the host address. For the scenario described in the previous section, requests to `/api/` should be routed to 'IMDB App' and the rest to 'IMDB UI`. Usually, this is easily doable by creating a [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) and adding respective rules. This alone wouldn't solve the problem here since the cluster is running on a virtual node inside Codespace. The ingress-managed loadbalancer is not directly bound to the Codespace host IP. This is where Codespaces' port-forwarding comes in handy. 

> All changes mentioned in the below steps are available as part of commit [Configure routing through ingress.](https://github.com/meavk/k8s-in-codespaces-routing/commit/3494e5945b0cacd64663a144d51d82f5ed85fdbb) for reference.

1. The first requirement for the solution is an ingress controller. `k3d` comes with [Traefik Ingress Controller](https://doc.traefik.io/traefik/providers/kubernetes-ingress/). Since Kubernetes in Codespaces has disabled this, a new ingress needs to be set up. Add the following code to `.devcontainer/on-create.sh` after `kic cluster deploy` to install the latest version of Traefik. 

   > This sample uses Traefik. Any other ingress, [Nginx](https://docs.nginx.com/nginx-ingress-controller/) for example, would also work. Just have to configure the rest accordingly

   ```bash
   echo "installing traefik"
   helm repo add traefik https://containous.github.io/traefik-helm-chart
   helm install traefik traefik/traefik
   ```

2. Next add a port mapping to `ports` section in `.devcontainer/k3d.yaml` to map port `8081` to Traefik `loadbalancer` port.

   ```yaml
     - port: 8081:80
       nodeFilters:
         - loadbalancer
   ```
   
   > The above configuration maps port `8081` from the host to port `80` on the container that matches the [nodefilter](https://k3d.io/v5.3.0/design/concepts/#nodefilters) `loadbalancer`. And it matches `traefik` LoadBalancer service.
   >  ```bash
   >  @meavk ➜ /workspaces/k8s-in-codespaces-routing (main) $ k get service traefik
   >  NAME      TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)                      AGE
   >  traefik   LoadBalancer   10.43.161.1   172.18.0.2    80:31244/TCP,443:30201/TCP   5d5h
   >  ```

3. Add port-forwarding to `.devcontainer/devcontainer.json`.
   
   ```json
   	// forward ports for the app
      "forwardPorts": [
         30000,
         30080,
         31080,
         32000,
         30090,
         8081
      ],

      // add labels
      "portsAttributes": {
         "30000": { "label": "Prometheus" },
         "30080": { "label": "IMDb App" },
         "31080": { "label": "Heartbeat" },
         "32000": { "label": "Grafana" },
         "30090": { "label": "IMDb UI" },
         "8081": { "label": "Traefik Ingress" }
      },
   ```
   
4. Lastly, to route the requests as described earlier, create `imdb-ingress.yaml` to create an ingress resource in a new folder `deploy/apps/ingress`.
   
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: imdb-ingress
     namespace: imdb
     annotations:
       ingress.kubernetes.io/ssl-redirect: "false"
   spec:
     rules:
     - http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: imdb-ui
               port:
                 number: 9080
         - path: /api
           pathType: Prefix
           backend:
             service:
               name: imdb
               port:
                 number: 8080
   ```
   
5. Rebuild Codespace to apply the changes. 
6. After the rebuild, once commands are run and the applications are up and running, navigate to 'Traefik Ingress' to load the UI. This time, the 'Movies' tab should load the full list of movies.

   ![image](https://user-images.githubusercontent.com/32096756/182047099-03ba4666-db54-4946-9241-e0c095f4334d.png)
   
   ![image](https://user-images.githubusercontent.com/32096756/182047141-e53edf28-90b2-4c3c-b31a-459bbeee2dec.png)
   
   ![image](https://user-images.githubusercontent.com/32096756/182047187-4ed83a2f-3a3f-4aca-b136-e749bd27b798.png)
