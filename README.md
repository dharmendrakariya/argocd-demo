## This document depicts the complete CI/CD workflow.

Tools we are using here are ```CiecleCI``` as a ```CI``` to build the docker image and push it to the DockerHub and ```ArgoCD``` as ```CD``` to deploy application in k8s Cluster.


#### Fisrt Part:

- We have [circleci-demo](https://github.com/dharmendrakariya/circleci-demo) named repo where we are having application code and Dockerfile

- on every commit on master triggers the Circle CI and builds the docker image with semver like 1.0.$(circle_build_number)

#### Second Part:

- We are having [helm-chart](https://github.com/dharmendrakariya/helm-charts) repo and we are storing our chart here.

    - How to create helm-chart repo in github is different topic, but you can refer the official [documentation](https://helm.sh/docs/howto/chart_releaser_action/). its very simple with the github actions.

    - The chart we are using is ```demo7```

#### Third Part: 

- We are having [argo-cd](https://github.com/dharmendrakariya/argocd-demo) repo where we are storing helm release (in ```hook``` branch) to get it synced in our cluster.

    
    - First, To install argocd in your cluster follow official [documentation](https://argo-cd.readthedocs.io/en/stable/getting_started/).

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

#### Fourth Part:

- Create a application in argocd dashboard with given attributes.

    - We will be installing helm chart with the subchart deploymenr method, us can refer [this](https://cloud.redhat.com/blog/continuous-delivery-with-helm-and-argo-cd) document. 

    Refer argo-cd repo, hook branch, I have stored the manifests which will be working as a subchart deployment method.

    repo url: https://github.com/dharmendrakariya/argocd-demo

    branch: hook

#### Fifth Part:

- Install argocd-image-updater with [this](https://argocd-image-updater.readthedocs.io/en/stable/install/start/#apply-the-installation-manifests) crds

    We will take help from [this](https://www.padok.fr/en/blog/argocd-image-updater) doc for auto image update feature.

    Things we need to consider first here, is secret ```Git credentials``` with PAT and if you have private registry then ```Image Registry credentials``` as well.

    I will name my git credentials secret ```argo-secret``` and it will be under argocd namespace, we will use it in our annotations

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

    - again apply the modified annotated-app.yaml with given command

        ```kc apply -f annotated-app.yaml```

#### Sixth Part:


- Go to ```circleci-demo``` repo, update the index.html, it will trigger the ci, and check the circleci dashboard or imageRepository(in my case dockerHub), it will push the updated image.  

- Check the ```argo-cd``` repo, you should see the commit with the updated image after couple of minutes. something like below commit

    ``` argocd-image-updater build: automatic update of circle-app ```

- Check the ```argo.yourDomain.xyz ``` to see if that gets synced with the newly commited changes, similarly go through the cluster, wait for five minutes, and watch out for the newly deployed application.

    the commit should be there, only after new commit it will deploy the updated manifest.

- Bye-Bye
