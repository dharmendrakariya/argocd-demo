## This document depicts the complete CI/CD workflow.

Tools we are using here are CiecleCI as a CI to build the docker image and push it to the DockerHub and ArgoCD as CD to deploy it in k8s Cluster.


Fisrt Part:

- We have [circleci-demo](https://github.com/dharmendrakariya/circleci-demo) named repo where we are having application code and Dockerfile

- on every commit on master triggers the Circle Ci and builds the docker image with semver like 1.0.$(circle_build_number)

Second Part:

- We are having [helm-chart](https://github.com/dharmendrakariya/helm-charts) repo and we are storing our chart here.

    - The chart we are using is ```demo7```

- We are having [argo-cd](https://github.com/dharmendrakariya/argocd-demo) repo where we are storing helm release to get it synced in our cluster.

    - To install argocd in your cluster follow official [documentation](https://argo-cd.readthedocs.io/en/stable/getting_started/).

    - After that you can create ingress resource for your deployed argocd

    You can use below template, store it in ```argo-ingress.yaml``` and apply with ```kubectl apply -f argo-ingress.yaml```

    ```
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
    annotations:
        kubernetes.io/ingress.class: nginx
        nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
        nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    name: argocd-server-ingress
    namespace: argocd
    spec:
    rules:
    - host: argo.yourDomain.xyz
        http:
        paths:
        - backend:
            service:
                name: argocd-server
                port:
                name: https
            path: /
            pathType: Prefix
    ```

    You would be able to see the ingress in ```argocd``` namespace.

    make the dns entry and try to access it, how to retrieve argo password is given in above mentioned official documentation.

- Create a application in argocd dashboard with given attributes.

    repo url: https://github.com/dharmendrakariya/argocd-demo

    branch: hook

- Install argocd-image-updater with [this](https://argocd-image-updater.readthedocs.io/en/stable/install/start/#apply-the-installation-manifests) crds

    We will take help from [this](https://www.padok.fr/en/blog/argocd-image-updater) doc for auto image update feature.

    Things we need to consider first here, is secret ```Git credentials``` with PAT and if you have private registry then ```Image Registry credentials``` as well.

    Then we need to add the required annotations in our deployed application, mind here we are using helm chart so check out ```The Helm Umbrella Charts``` section.

    Note: to get the deployed application manifest, we can get this by using below commands.

    - ``` kc get app -n argocd circle-app -oyaml > annotated-app.yaml ```
    
    I have deployed app with circle-app name in ui.

    - then add the below annotations.

        ```
        argocd-image-updater.argoproj.io/demo7.helm.image-name: demo7.image.name
        argocd-image-updater.argoproj.io/demo7.helm.image-tag: demo7.image.tag
        argocd-image-updater.argoproj.io/image-list: demo7=dharmendrakariya/circle-app
        argocd-image-updater.argoproj.io/write-back-method: git:secret:argocd/argo-secret
        ```

    - againg apply the modified annotated-app.yaml with given command

        ```kc apply -f annotated-app.yaml```

- Go to ```circleci-demo``` repo, update the index.html and check the circleci dashboard or imageRepository(in my case dockerHub), it will push the     updated image.  

- Check the ```argo-cd``` repo, you should see the commit with the updated image. something like below commit

    ``` argocd-image-updater build: automatic update of circle-app ```

- Check the ```argo.yourDomain.xyz ``` to see if that gets synced with the newly commited changes, similarly go through the cluster and watch out for the newly deployed application.


- Bye-Bye