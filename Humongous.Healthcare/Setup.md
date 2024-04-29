<h1> Review and publish the Humongous Healthcare Web API service</h1>

### Task 1:  Review the Humongous Healthcare Web API service

1. Open the hands-on lab in Visual Studio Code and use the terminal to navigate to the `Humongous.Healthcare` project.

2. Run the following commands in the terminal to restore and build the project, bringing in necessary packages.

    ```sh
    dotnet restore
    dotnet build
    ```

3. Review the `HealthCheckController.cs` controller to see the available endpoints.  There are two endpoints included in this controller:  `HealthCheck` and `HealthCheck/GetStatus`.  The `HealthCheck` endpoint accepts four REST API verbs:  `GET`, `PUT`, `POST`, and `DELETE`.  Furthermore, data retrieval using `GET` will either list health checks (when passing in zero parameters) or retrieve details on a single health check (passing in one parameter).  For this proof of concept, we do not implement logic to control results by patient ID but the code base could be extended to incorporate this.  The `GetStatus` method serves as a quick test method, ensuring that the service is up by returning data.

4. Review the `Startup.cs` file, particularly the `ConfigureServices()` and `Configure()` methods.  Because this project uses the `Swashbuckle.AspNetCore` NuGet package, we can build a Swagger API and user interface automatically from our Web API definition.  This will be important later on when integrating with API Management.

5. Open the `appsettings.json` file and replace the `Account` and `Key` with your Cosmos DB URI and primary key, respectively.

6. Try this project locally and ensure that it works as expected.  To do this, execute the following in the terminal:

    ```sh
    dotnet run
    ```

7. Use the Postman application to call the `HealthCheck/GetStatus` endpoint.  This should return one record.

8. Use the Postman application to post three status updates, one call at a time.  Use the following JSON blocks to execute `POST` operations against the `HealthCheck` endpoint.  Be sure to set the Body format to JSON.  You should get back a **201 Created** response with a copy of each created record.

    ```json
    {
        "patientid": 5,
        "date": "2021-09-01T00:41:49.9602636+00:00",
        "healthstatus": "I feel healthy",
        "symptoms": [
            "None"
        ]
    }
    ```

    ```json
    {
        "patientid": 5,
        "date": "2021-09-04T00:41:49.9602636+00:00",
        "healthstatus": "I feel unwell",
        "symptoms": [
            "Hair loss",
            "Internal bleeding",
            "Temporary blindness"
        ]
    }
    ```

    ```json
    {
        "patientid": 5,
        "date": "2021-09-08T00:41:49.9602636+00:00",
        "healthstatus": "I feel healthy",
        "symptoms": [
            "Hair regrowth",
            "Ennui"
        ]
    }
    ```

9. Use the `GET` verb on the `HealthCheck` endpoint to retrieve an array containing health check records.

10. When you are done, use `Ctrl+C` to stop the dotnet service.

### Task 2: Run the Humongos Healthcare Web API service in a container

1. Open the hands-on lab in Visual Studio Code and use the terminal to navigate to the `Humongous.Healthcare` project.

2. Create a file named `Dockerfile` and add the following contents.

    ```dockerfile
    FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build-env
    WORKDIR /app

    # Copy project file
    COPY ./Humongous.Healthcare.csproj .
    # Restore as distinct layer
    RUN dotnet restore

    # Copy everything else
    COPY . .

    # Build and publish a release
    RUN dotnet publish -c Release -o out

    # Build runtime image
    FROM mcr.microsoft.com/dotnet/aspnet:6.0
    WORKDIR /app
    COPY --from=build-env /app/out .
    ENTRYPOINT ["dotnet", "Humongous.Healthcare.dll"]
    ```

    This is a multi-stage Dockerfile with 2 stages, build and runtime.  The build stage uses the .net 6 SDK image which contains all the compilers and tooling needed to create the Web API service. In contrast, the runtime stage is based on the smaller `aspnet` image, which only contains the necesssary binaries to execute asp.net applications.

    The build stage:
    - Copies the project file into the Docker container and runs `dotnet restore` to retrieve nuget packages.  This is a stand-alone layer so that the packages will be cached if we need to run `docker build` again.
    - Copies the remaining project files to the container then runs `dotnet publish` to create the Web API service.

    The runtime stage starts from a clean `aspnet` image without compiler tooling or intermediate object files then:
    - Copies the Web API binaries from the build stage.
    - Configures the container image to run the Web API on startup.

3. Run the following command to build the container image.

    ```sh
    docker build -t api .
    ```

    **Info**
    
    If the image is being build on an ARM machine (for example a Macbook with M1 chip), you may need to use the following command to build the image:

    ```sh
    docker build --platform linux/amd64 -t api .    
    ```

4. Run the following command to run an instance of the container image.

    ```sh
    docker run -it -p 5000:80 api
    ```

5. Use postman (or your browser) to `GET` the following URL and confirm the Web API is functioning inside the container.

    `http://localhost:5000/HealthCheck`

6. When you are done, use `Ctrl+C` to stop the container instance.

### Task 3: Push the Humongous Healthcare Web API container image to ACR

1. Use the following command to tag your container image with your ACR login server name and a more descriptive repository name:

    ```sh
    docker tag api <replace with your login server>/humongous-healthcare-api:0.0.0
    ```

    Example:

    ```sh
    docker tag api taw.azurecr.io/humongous-healthcare-api:0.0.0
    ```

2. Login to the Azure CLI and the ACR instance using the following commands:

    ```sh
    az login
    az acr login --name <acrName>
    ```

    Example
    ```sh
    az login
    az acr login --name taw
    ```

3. Use the following command to push the container image to ACR.

    ```sh
    docker push <replace with your login server>/humongous-healthcare-api:0.0.0
    ```

    Example:

    ```sh
    docker push taw.azurecr.io/humongous-healthcare-api:0.0.0
    ```

### Task 4: Deploy the Humongous Healthcare Web API service to AKS

1. Open the hands-on lab in Visual Studio Code.

2. Create a folder named `manifests` and add a file named `deployment.yml` with the following content:

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: "humongous-healthcare-api"
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: "humongous-healthcare-api"
      template:
        metadata:
          labels:
            app: "humongous-healthcare-api"
        spec:
          containers:
          - name: "humongous-healthcare-api"
            image: "<replace with your login server>/humongous-healthcare-api:0.0.0"
            env:
            - name: CosmosDb__Account
              valueFrom:
                secretKeyRef:
                  name: cosmosdb
                  key: cosmosdb-account
                  optional: false
            - name: CosmosDb__Key
              valueFrom:
                secretKeyRef:
                  name: cosmosdb
                  key: cosmosdb-key
                  optional: false
            - name: CosmosDb__DatabaseName
              value: HealthCheckDB
            - name: CosmosDb__ContainerName
              value: HealthCheck
            imagePullPolicy: IfNotPresent
            ports:
              - name: http
                containerPort: 80
                protocol: TCP
            livenessProbe:
              httpGet:
                path: /HealthCheck
                port: http
            readinessProbe:
              httpGet:
                path: /HealthCheck
                port: http
            resources:
              limits:
                cpu: 200m
                memory: 256Mi
              requests:
                cpu: 200m
                memory: 256Mi
    ```

    This manifest contains the AKS deployment configuration for the Humongous Healthcare Web API container images.  Be sure to update `<replace with your login server>` with your ACR login server name.

3. Create another YAML file in the `manifest` folder called `service.yml` and add the following content:

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: "humongous-healthcare-api"
      labels:
        app: "humongous-healthcare-api"
    spec:
      type: LoadBalancer
      ports:
      - port: 80
        targetPort: 80
        protocol: TCP
        name: http
      selector:
        app: "humongous-healthcare-api"
    ```

    This manifest contains the AKS service configuration for the Humongous Healthcare Web API deployment.  Because it is a `LoadBalancer` type service, AKS will make the Web API service available to clients outside the cluster.

4. Login to your AKS cluster using the Azure CLI command shown below.

    ```sh
    az aks get-credentials --name <your cluster name> --resource-group <your resource group>
    ```

    Example

    ```
    az aks get-credentials --name taw-aks --resource-group taw
    ```

5. Create a Kubernetes namespace for the service using the following command.

    ```sh
    kubectl create namespace health-check
    ```

6. Create a Kubernetes secret to provide the CosmosDB configuration using the following command.

   ```sh
   kubectl create secret generic cosmosdb --namespace health-check --from-literal=cosmosdb-account='<Replace with your account URL>' --from-literal=cosmosdb-key='<Replace with your account key>'
   ```

    Example:

   ```sh
   kubectl create secret generic cosmosdb --namespace health-check --from-literal=cosmosdb-account='https://taw-cosmosdb.documents.azure.com:443/' --from-literal=cosmosdb-key='qwerty123456!@#$%==='
   ```

7. Create the AKS deployment and service using the following commands.

    ```sh
    kubectl apply --namespace health-check --filename manifests/deployment.yml --filename manifests/service.yml
    ```

8. Visit your AKS cluster in the Azure Portal and navigate to "Kubernetes resources > Services and ingresses".  Click the "External IP" link for `humonguous-healthcare-api`.  **Note**: You will initially recieve a 404 error when the new brower tab opens.  Continue to the next step.

    ![The portal AKS workloads is displayed.](media/aks-service-and-ingresses.png "Service External IP")

9. Add the `/HealthCheck` IP to the end of the URL to view the raw health check data.

    ![The Health Check data is displayed in the browser.](media/aks-health-checks.png "Health Check JSON")

7. Once this deployment succeeds, you should be able to navigate to your External IP address in Postman and perform the same `GET`, `POST`, `PUT`, and `DELETE` operations you could locally.

## Exercise 3:  Configure AKS continuous deployment with GitHub Actions

### Task 1:  Create a GitHub Actions Workflow

1. Open the hands-on lab in Visual Studio Code.

2. Create a folder named `.github/workflows` and add a file named `deployToAksCluster.yml` with the following content (remember to replace the placeholders with your ACR name where noted below):

    ```yaml
    name: API - AKS - Build and deploy

    on:
    push:
      branches:
      - main
      paths:
      - Humongous.Healthcare/**
      - manifests/**

    jobs:
      build-and-deploy:
        runs-on: ubuntu-latest
        env:
          ACR_LOGIN_SERVER: <replace with your ACR name>.azurecr.io
          AKS_NAMESPACE: health-check
          CONTAINER_IMAGE: <replace with your ACR name>.azurecr.io/humongous-healthcare-api:${{ github.sha }}
        steps:
        - uses: actions/checkout@master

        - uses: azure/docker-login@v1
          with:
            login-server: ${{ env.ACR_LOGIN_SERVER }}
            username: ${{ secrets.acr_username }}
            password: ${{ secrets.acr_password }}

        - name: Build and push image to ACR
          id: build-image
          run: |
            docker build "$GITHUB_WORKSPACE/Humongous.Healthcare" -f  "Humongous.Healthcare/Dockerfile" -t ${CONTAINER_IMAGE} --label dockerfile-path=Humongous.Healthcare/Dockerfile
            docker push ${CONTAINER_IMAGE}

        - uses: azure/k8s-set-context@v1
          id: login
          with:
            kubeconfig: ${{ secrets.aks_kubeConfig }}

        - name: Create namespace
          run: |
            namespacePresent=`kubectl get namespace | grep ${AKS_NAMESPACE} | wc -l`
            if [ $namespacePresent -eq 0 ]
            then
                echo `kubectl create namespace ${AKS_NAMESPACE}`
            fi

        - uses: azure/k8s-create-secret@v1
          name: dockerauth - create secret
          with:
            namespace: ${{ env.AKS_NAMESPACE }}
            container-registry-url: ${{ env.ACR_LOGIN_SERVER }}
            container-registry-username: ${{ secrets.acr_username }}
            container-registry-password: ${{ secrets.acr_password }}
            secret-name: dockerauth

        - uses: Azure/k8s-create-secret@v1
          name: cosmosdb - create secret
          with:
            namespace: ${{ env.AKS_NAMESPACE }}
            secret-type: 'generic'
            secret-name: cosmosdb
            arguments:
              --from-literal=cosmosdb-account=${{ secrets.COSMOSDB_ACCOUNT }}
              --from-literal=cosmosdb-key=${{ secrets.COSMOSDB_KEY }}

        - uses: azure/k8s-deploy@v1.2
          with:
            namespace: ${{ env.AKS_NAMESPACE }}
            manifests: |
              manifests/deployment.yml
              manifests/service.yml
            images: |
              ${{ env.CONTAINER_IMAGE }}
            imagepullsecrets: |
              dockerauth
    ```

    > Note: An example of this YAML file can be located in the `./actions` folder of this lab as `deployToAksCluster.yml`.

3. The workflow will fail on the intial run because it requires some secrets to be setup in the repository.  Navigate to the repository in GitHub then select "Settings > Secrets > Actions".  Finally use the "New Repository Secret" button to create secrets.

    Create the following secrets.

    - ACR_USERNAME - Copy from the "Access Keys" pane on your container registry.
    - ACR_PASSWORD - Copy from the "Access Keys" pane on your container registry.
    - AKS_KUBECONFIG - Use the following Azure CLI command to retrieve the configuration data and save it to a file, then use the contents of the file as your secret: `az aks get-credentials --name <your cluster name> --resource-group <your resource group> --file kube.config`
    - COSMOSDB_ACCOUNT - Use the CosmosDb account URI you have previously retrieved during this lab.
    - COSMOSDB_KEY - Use the CosmosDb account key you have previously retrieved during this lab.

4. Once you create all the secrets, navigate to the "Actions" tab, select your failed workflow run, and choose "Re-run all Jobs".

5.  Once this deployment succeeds, you should still be able to navigate to your App Service URL in Postman and perform the same `GET`, `POST`, `PUT`, and `DELETE` operations you could locally.
