Folder Structure (Example)


project-root/
├── backend/
│   ├── app.py
│   ├── requirements.txt
│   ├── .env
│   ├── Dockerfile
│
├── frontend/
│   ├── public/
│   ├── src/
│   ├── package.json
│   ├── Dockerfile.frontend
│
├── k8s
    frontend-deploy.yaml
    backend-deploy.yaml
    argocd.yaml




Let's go step-by-step to build your frontend and backend Docker images and push them to Azure Container Registry (ACR).

✅ STEP 1: Login to Azure & ACR

az login
az acr login --name <your-acr-name>



Replace <your-acr-name> with your actual ACR name (not the login server). You can find your login server using:


az acr show --name <your-acr-name> --query "loginServer" --output tsv




✅ STEP 2: Tag & Build Images


📦 Backend


cd backend

# Build the backend image
docker build -t my-backend .

# Tag it for ACR
docker tag my-backend <acr-login-server>/my-backend:v1

# Push it to ACR
docker push <acr-login-server>/my-backend:v1







📦 Frontend


cd ../frontend

# Build the frontend image
docker build -f Dockerfile.frontend -t my-frontend .

# Tag it for ACR
docker tag my-frontend <acr-login-server>/my-frontend:v1

# Push it to ACR
docker push <acr-login-server>/my-frontend:v1





✅ STEP 3: Verify Images in ACR

az acr repository list --name <your-acr-name> --output table





✅ Final Tips

| Task            | Example                                |
| --------------- | -------------------------------------- |
| ACR Name        | `myregistry`                           |
| ACR LoginServer | `myregistry.azurecr.io`                |
| Frontend Image  | `myregistry.azurecr.io/my-frontend:v1` |
| Backend Image   | `myregistry.azurecr.io/my-backend:v1`  |


✅ 7. Verify Deployment


kubectl get pods -n app
kubectl get svc -n app
kubectl get ingress -n app

Check the Ingress or LoadBalancer IP to access the frontend.





You're running your app successfully in the app namespace on AKS 🎉

Now let's add Argo CD into the same AKS cluster — but in a separate namespace (argocd) using local YAMLs or Helm.






✅ What We'll Do
Create argocd namespace

Install Argo CD using official YAML (from local or remote)

Access Argo CD UI

Configure port-forward or LoadBalancer for web access

(Optional) Add Argo CD app config to sync your app repo





✅ STEP 1: Create Argo CD Namespace


kubectl create namespace argocd





✅ STEP 2: Install Argo CD Using YAML (Recommended for Local Dev)
You can apply the official install manifest directly (if your cluster has internet access):


kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml








✅ STEP 3: Verify Argo CD is Running


kubectl get pods -n argocd


Wait for them to reach Running state.




✅ STEP 4: Expose Argo CD UI
Option 1: Port Forward (for local testing)




kubectl port-forward svc/argocd-server -n argocd 8080:443







Then open your browser at: https://localhost:8080

⚠️ It's HTTPS, so ignore the certificate warning.






Option 2: Expose with LoadBalancer
Create a YAML to expose Argo CD externally (optional):




# argocd-server-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: argocd-server
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-server
spec:
  type: LoadBalancer
  ports:
    - name: https
      port: 443
      targetPort: 8080
  selector:
    app.kubernetes.io/name: argocd-server





Apply it:


kubectl apply -f argocd-server-service.yaml





Then get the external IP:


kubectl get svc -n argocd






✅ STEP 5: Login to Argo CD
Get the admin password:

bash
Copy
Edit




kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo






Then login with:

Username: admin

Password: (output above)




✅ TL;DR

| Task                     | Command/Status                                              |
| ------------------------ | ----------------------------------------------------------- |
| Create namespace         | `kubectl create ns argocd`                                  |
| Install Argo CD          | `kubectl apply -n argocd -f <yaml>`                         |
| Access UI (port-forward) | `kubectl port-forward svc/argocd-server -n argocd 8080:443` |
| Get login password       | `kubectl get secret ...`                                    |
| Login                    | `admin / <password>`                                        |
| Deploy app via Argo CD   | ✅ (You can use GUI or App YAML)                             |














 your backend and frontend apps are already containerized and pushed to ACR, and your Kubernetes YAMLs (like backend-deployment.yaml, frontend-deployment.yaml) are stored locally or committed in Git, you can let Argo CD deploy and manage them via GitOps.




 ✅ Prerequisites
You must push your k8s/ folder (which contains your Kubernetes manifests) to a Git repo like GitHub.
Example structure in Git:




repo-root/
└── k8s/
    ├── backend-deployment.yaml
    ├── frontend-deployment.yaml




You should know:

Git repo URL: https://github.com/<username>/<repo>.git

Branch name (e.g., main)

Path to YAMLs (e.g., k8s)





✅ Argo CD Application YAML (for both backend and frontend)


apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: todo-app
  namespace: argocd
spec:
  project: default

  source:
    repoURL: 'https://github.com/<your-username>/<your-repo>.git'
    targetRevision: main
    path: k8s   # folder where backend-deployment.yaml & frontend-deployment.yaml exist

  destination:
    server: 'https://kubernetes.default.svc'
    namespace: app

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true





✅ How to Apply It
Save the above YAML as todo-app.yaml

Apply it:


kubectl apply -f todo-app.yaml -n argocd




✅ Result
Argo CD will watch your Git repo.

When anything changes inside k8s/, Argo CD will auto-sync and deploy the updated manifests.

The app will run in the app namespace.




✅ Access Argo CD UI
You can now open the UI (https://localhost:8080 via port-forward or public IP) and:

See the app todo-app listed

View live status: Sync, Healthy, Out-of-Sync

Roll back or manually sync changes












 Your app is running in AKS, but your Kubernetes YAML files are local. To connect Argo CD to your app, you need to:




 ✅ Step-by-Step: Push Your Local App to GitHub for Argo CD
🔹 Step 1: Prepare Local Git Repo
From your local directory (~/Desktop/AKS/app/k8s):





cd ~/Desktop/AKS/app
git init
git add .
git commit -m "Initial commit for backend and frontend K8s manifests"





 Step 2: Create a New Repo on GitHub
Go to GitHub → New Repository

Name: todo-app (or anything)

Keep it public or private

Don’t add README or license (you already have code locally)



🔹 Step 3: Push Local Code to GitHub
After creating the repo, GitHub will show you commands like:



git remote add origin https://github.com/<your-username>/<your-repo>.git
git branch -M main
git push -u origin main




🔹 Step 4: Confirm YAML Path
In GitHub, verify that your folder k8s/ exists and contains:

backend-deployment.yaml

frontend-deployment.yaml





🔹 Step 5: Create Argo CD Application YAML
Now create an Application manifest like this (save it as todo-app.yaml):




apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: todo-app
  namespace: argocd
spec:
  project: default

  source:
    repoURL: 'https://github.com/<your-username>/<your-repo>.git'
    targetRevision: main
    path: k8s

  destination:
    server: 'https://kubernetes.default.svc'
    namespace: app

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true




Replace the GitHub repo URL and ensure path: k8s is correct.



🔹 Step 6: Apply to AKS



kubectl apply -f todo-app.yaml -n argocd




✅ DONE: Argo CD Will Now
Fetch YAML from GitHub

Deploy to app namespace

Sync automatically whenever Git changes




✅ Summary

| Task                        | Tool                   | Status |
| --------------------------- | ---------------------- | ------ |
| Local code pushed to GitHub | `git push`             | ✅      |
| Argo CD config from GitHub  | `Application.yaml`     | ✅      |
| Argo CD watches repo        | Auto-deploys on change | ✅      |











let’s create your first Application from your GitHub repo.

✅ Create Argo CD Application via UI
Click “+ NEW APP” in the Argo CD dashboard and fill out the form like this:





GENERAL
Application Name: todo-app

Project: default

Sync Policy: Leave as Manual for now

SOURCE
Repository URL:


https://github.com/imapsingh007/todo.git




Revision: main

Path:

.



This assumes both backend-deployment.yaml and frontend-deployment.yaml are in the root of the repo (which they are, per your screenshot).




DESTINATION
Cluster URL:

https://kubernetes.default.svc





Namespace:

app




DIRECTORY SETTINGS
Leave everything default.




✅ Then:
Click Create

You’ll see the app todo-app listed

Click on it and hit “Sync” to deploy the manifests to your AKS cluster









