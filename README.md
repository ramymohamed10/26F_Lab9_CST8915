# Lab 9 — Automate the Algonquin Pet Store with GitHub Actions

In Lab 8, you pushed six application images to Docker Hub and deployed them to Azure Kubernetes Service (AKS) by hand. In Lab 9, you will automate that work with a continuous integration and continuous deployment (CI/CD) pipeline. Each push to a service's `main` branch builds your changes. A successful release updates the same Docker Hub repository and tag you used in Lab 8, then rolls that image out to AKS.

Start with Store Front so you can see one release go all the way through. Then repeat for the other five application services.

## Step 1. Connect to your lab cluster

Use your Lab 8 cluster if it is still running. If you cleaned it up, create a cluster with the Lab 8 node configuration and the **Local accounts with Kubernetes RBAC** setting from Lab 6. Connect to it using `az aks get-credentials`. [Lab 6 cluster setup](../Lab6_CST8915/README.md#step-2-create-an-azure-kubernetes-cluster-aks) · [Lab 8 cluster setup](../Lab8_CST8915/Lab8_Instructions.md#step-2-set-up-the-aks-cluster)

Run these commands in the terminal where you connected to AKS. Check that the context names your lab cluster and the nodes show `Ready`. Use this same context and namespace throughout the lab.

```bash
kubectl config current-context
kubectl get nodes
```

## Step 2. Deploy the Lab 8 application

The workflow updates an existing Kubernetes Deployment. Set up all six Deployments before adding the workflow to any fork.

1. From the root of this course repository, open the Lab 9 deployment folder.

   ```bash
   cd "Lab9_CST8915/Deployment Files"
   ```

2. Copy your completed Lab 8 Task 1 manifest into this folder as `aps-all-in-one-Task1.yaml`. If you need to use the supplied `aps-all-in-one.yaml`, change the six application `image:` values to the images you already pushed to your Docker Hub account in Lab 8. Keep `imagePullPolicy: Always` on all six application containers so AKS pulls the updated tag when a pod restarts. [Lab 8 Task 1](../Lab8_CST8915/Lab8_Instructions.md#task-1-build-push-and-deploy-your-own-docker-images)

3. Prepare `secrets.yaml` in this folder using the same AI provider you used in Lab 8. Git ignores the populated file. Keep its contents out of commits and workflow logs.

   | Your Lab 8 provider | What to do |
   | --- | --- |
   | Direct OpenAI through Brightspace | Save the current Brightspace `secrets.yaml` here as supplied. In your application manifest, leave `USE_AZURE_OPENAI` set to `"False"`. [Student guide](../Lab8_CST8915/Student_Guide_OpenAI.md) |
   | Azure OpenAI | Reuse your Lab 8 `secrets.yaml`, or run `cp secrets.example.yaml secrets.yaml` and fill in the text and image resource keys. In your application manifest, set `USE_AZURE_OPENAI` to `"True"` and check both endpoints and deployment names. [Lab 8 Azure settings](../Lab8_CST8915/Lab8_Instructions.md#step-3-set-up-the-ai-backing-services) |

4. Apply the RabbitMQ ConfigMap and both AI Secrets. The last two commands confirm their names.

   ```bash
   kubectl apply -f config-maps.yaml
   kubectl apply -f secrets.yaml
   kubectl get configmap rabbitmq-enabled-plugins
   kubectl get secret openai-gpt-api-secret openai-dalle-api-secret
   ```

5. Apply your Task 1 manifest. If you chose the supplied starter, use `kubectl apply -f aps-all-in-one.yaml` for this step.

   ```bash
   kubectl apply -f aps-all-in-one-Task1.yaml
   ```

6. Check all six Deployments. Run the command again after a minute if pods are still starting. Continue when every row shows all replicas ready, for example `1/1`.

   ```bash
   kubectl get deployments store-front store-admin order-service product-service makeline-service ai-service
   ```

## Step 3. Configure your Store Front fork

Use your own `store-front-L8` fork from Lab 8. If you need to fork it, use the source link in the table below. In your fork, open **Actions** and enable workflows if GitHub prompts you.

In the same terminal you used for Step 2, run the block for your shell. It saves the encoded kubeconfig for your current lab context as `lab9-kubeconfig.b64` in your home folder. GitHub Actions uses this file's contents to connect to your cluster.

**Linux, macOS, or Windows with WSL (Bash or zsh).** Run `umask 077` on its own line, then run the pipeline.

```bash
umask 077
kubectl config view --minify --flatten --raw | base64 | tr -d '\n' > "$HOME/lab9-kubeconfig.b64"
```

**Windows PowerShell.** Open PowerShell in the Windows account where you configured `kubectl`.

```powershell
$kubeConfig = (kubectl config view --minify --flatten --raw) -join "`n"
if ($LASTEXITCODE -ne 0) { throw "Check your kubectl context and try again." }
$encoded = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes($kubeConfig))
Set-Content -LiteralPath "$HOME/lab9-kubeconfig.b64" -Value $encoded -Encoding Ascii -NoNewline
```

### Values for your fork

Choose either the [GitHub website](#option-a-use-the-github-website) or the [GitHub CLI](#option-b-use-the-github-cli) to save the settings below. Use the same three **repository secrets** in all six forks.

| Secret name | Value |
| --- | --- |
| `DOCKER_USERNAME` | The Docker Hub username you used to push your Lab 8 images. |
| `DOCKER_PASSWORD` | A Docker Hub personal access token with read and write access. [Docker access tokens](https://docs.docker.com/security/access-tokens/) |
| `KUBE_CONFIG_DATA` | The contents of `lab9-kubeconfig.b64` in your home folder. Treat this file like a password. |

The three **repository variables** depend on the service. Use the repository and tag from your Lab 8 image for `DOCKER_IMAGE_NAME`. For example, if your image is `alex/store-front:lab8`, enter `store-front:lab8`. The workflow gets `alex` from `DOCKER_USERNAME`. Use the last column for both `DEPLOYMENT_NAME` and `CONTAINER_NAME`.

| Source repository to fork | Example `DOCKER_IMAGE_NAME` from Lab 8 | `DEPLOYMENT_NAME` and `CONTAINER_NAME` |
| --- | --- | --- |
| [store-front-L8](https://github.com/ramymohamed10/store-front-L8) | `store-front:lab8` | `store-front` |
| [store-admin-L8](https://github.com/ramymohamed10/store-admin-L8) | `store-admin:lab8` | `store-admin` |
| [order-service-L8](https://github.com/ramymohamed10/order-service-L8) | `order-service:lab8` | `order-service` |
| [product-service-L8](https://github.com/ramymohamed10/product-service-L8) | `product-service:lab8` | `product-service` |
| [makeline-service-L8](https://github.com/ramymohamed10/makeline-service-L8) | `makeline-service:lab8` | `makeline-service` |
| [ai-service-L8](https://github.com/ramymohamed10/ai-service-L8) | `ai-service:lab8` | `ai-service` |

### Option A. Use the GitHub website

1. In your Store Front fork, open **Settings → Secrets and variables → Actions**.
2. On the **Secrets** tab, choose **New repository secret**. Enter `DOCKER_USERNAME` in **Name** and your Docker Hub username in **Secret**, then click **Add secret**. Repeat for `DOCKER_PASSWORD` and `KUBE_CONFIG_DATA` using the secrets table above.
3. On the **Variables** tab, choose **New repository variable**. Enter `DOCKER_IMAGE_NAME` in **Name** and your Lab 8 repository and tag in **Value**, then click **Add variable**. Add `DEPLOYMENT_NAME` and `CONTAINER_NAME` as separate variables. Both values are `store-front` for this fork.
4. Check that the three secret names appear under **Repository secrets** and the three variable names and values appear under **Repository variables**. If an entry already exists, use its edit button to update it.

### Option B. Use the GitHub CLI

Install the [GitHub CLI](https://cli.github.com/) if needed. Run `gh auth status` to check which GitHub account is active. If you need to sign in, run `gh auth login --hostname github.com --web` and choose the account that owns your fork.

1. Use the part of your fork's URL after `github.com/`. The examples below show the course repository. Replace `YOUR-GITHUB-USERNAME` with the owner of your own fork, then run the block for your shell to upload `KUBE_CONFIG_DATA`.

   **Bash or zsh (Linux, macOS, or WSL)**

   ```bash
   # Course example: labRepo="ramymohamed10/store-front-L8"
   labRepo="YOUR-GITHUB-USERNAME/store-front-L8"
   gh secret set KUBE_CONFIG_DATA --repo "$labRepo" < "$HOME/lab9-kubeconfig.b64"
   ```

   **Windows PowerShell**

   ```powershell
   # Course example: $labRepo = "ramymohamed10/store-front-L8"
   $labRepo = "YOUR-GITHUB-USERNAME/store-front-L8"
   Get-Content -Raw -LiteralPath "$HOME/lab9-kubeconfig.b64" | gh secret set KUBE_CONFIG_DATA --repo "$labRepo"
   ```

2. In the same terminal, run these commands one at a time. At the first prompt, enter your Docker Hub username. At the second, paste your Docker Hub access token. GitHub CLI hides what you type. [GitHub CLI secret commands](https://cli.github.com/manual/gh_secret_set)

   ```sh
   gh secret set DOCKER_USERNAME --repo "$labRepo"
   gh secret set DOCKER_PASSWORD --repo "$labRepo"
   ```

3. Set the three Store Front variables. If your Lab 8 image uses a different repository or tag, change `store-front:lab8` in the first command. These commands also update existing entries. [GitHub CLI variable commands](https://cli.github.com/manual/gh_variable_set)

   ```sh
   gh variable set DOCKER_IMAGE_NAME --body "store-front:lab8" --repo "$labRepo"
   gh variable set DEPLOYMENT_NAME --body "store-front" --repo "$labRepo"
   gh variable set CONTAINER_NAME --body "store-front" --repo "$labRepo"
   ```

4. Verify the three secret names and the three variable values in your fork.

   ```sh
   gh secret list --repo "$labRepo"
   gh variable list --repo "$labRepo" --json name,value
   ```

## Step 4. Run and verify your first pipeline

1. Copy the [workflow template](./Workflow%20Files/ci_cd.yaml). In your Store Front fork, open `.github/workflows/ci_cd.yaml` for editing, or use **Add file → Create new file** to create it at that path. Paste the template and commit to `main` to trigger the workflow.
2. Open **Actions → Build, Test, and Deploy** and select your run. The workflow has four jobs. GitHub starts each job on a runner (a virtual machine) after the previous job succeeds. Select a job to see the log for each step.

   | Job | What to look for in the log |
   | --- | --- |
   | Build | The runner pulls your Lab 8 image, builds the code in your fork, and pushes the build with a tag such as `:test-12345`. The number identifies the workflow run. The unit-test step prints a simulated result. |
   | Test | The integration-test step also prints a simulated result. |
   | Release | The runner publishes the tested build under your original Lab 8 tag. That tag now points to the new build. |
   | Deploy | The runner restarts the Store Front Deployment so its pods pull the updated image, then waits for the rollout. |

3. Verify the result from your terminal.

   ```bash
   kubectl rollout status deployment/store-front
   kubectl get deployment store-front -o 'custom-columns=IMAGE:.spec.template.spec.containers[0].image'
   kubectl get service store-front
   ```

   Look for a successful rollout and the same image name and tag you used in Lab 8, for example `alex/store-front:lab8`. Open the service's external IP in a browser using `http://` and check that the storefront loads.

## Step 5. Run the other five pipelines

For each remaining fork, enable Actions and repeat your chosen setup method from Step 3. With the CLI, change `labRepo` to the next fork before running any `gh` commands. Reuse the same secrets and kubeconfig file, and set the three variables using that service's row. Commit the workflow template to `main` at `.github/workflows/ci_cd.yaml` and check the run in **Actions**.

Once all six runs succeed, list the deployed images together.

```bash
kubectl get deployments store-front store-admin order-service product-service makeline-service ai-service -o 'custom-columns=SERVICE:.metadata.name,READY:.status.readyReplicas,IMAGE:.spec.template.spec.containers[0].image'
```

Each row should show ready replicas and the matching image from your Lab 8 manifest. If Build fails at `docker pull`, compare `DOCKER_USERNAME/DOCKER_IMAGE_NAME` with the full image reference in Docker Hub. If Deploy cannot find a Deployment or container, check the fork's two Kubernetes variables against the service table and confirm you used the Step 2 cluster.

## Submit and clean up

Submit links to your **six service forks** in Brightspace. The instructor will use the GitHub Actions run in each fork for marking.

After verifying your runs, remove the temporary `lab9-kubeconfig.b64` file from your home folder. Clean up the Azure resources you created for this lab to stop further charges. The course estimate for the lab is about $3 of Azure credit.
