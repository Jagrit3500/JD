# HireFlow: your complete step-by-step video guide

Start at Step 1 and follow this document in order. You do not need another recording guide. This is one scrolling page, with preparation first and the actual video second.

**How to read this page**

- **DO** means perform the action. Do not read the instruction aloud.
- **RUN** means paste the command into the specified PowerShell window and press Enter.
- **CHECK** means look at the result before moving on.
- **SAY** means read the quoted paragraph aloud while recording.
- Copy only the text inside command boxes. Do not type `PS C:\...>` or `>>`; these are PowerShell prompts.
- Run command blocks only when you reach them. Do not paste the entire guide into PowerShell.
- The rough recording time is 15–25 minutes, depending on the pipeline. This is a planning estimate.

This guide stays on your computer under `.local-notes/`. Git ignores it. The required presentation and finished video belong under the normal `demo/` folder, outside `.local-notes/`.

---

## BEFORE RECORDING — complete Steps 1–9 first

### Step 1. Prepare your workspace and notes

**DO**

1. Connect your laptop to power and check your internet connection.
2. Close unrelated applications and enable Do Not Disturb to reduce notification pop-ups.
3. Open File Explorer and go to `C:\Users\jagri\Documents\DevOps_Project`.
4. Open the project folder in your code editor. If using VS Code, choose **File > Open Folder**, select this folder and open it.
5. Open `demo\HireFlow_Presentation.pdf` in a PDF viewer. Set it to slide/page 1. Use one page at a time and fit the page to the window.
6. Put this guide on your phone, paper, or a second screen outside the recording area. Keep it available throughout the video.
7. Increase editor and PowerShell text size until it is easy to read in a recording. Use a browser zoom level that shows both labels and results clearly.

You will switch between four things: the PDF presentation, your editor, your browser, and Main PowerShell. Use **Alt+Tab** to switch applications and click browser tabs to change pages.

### Step 2. Open three separate PowerShell windows

**DO** Open PowerShell from the Windows Start menu. Open it three times so you have three windows. Existing windows can be reused.

| Window | Purpose | What happens to it |
|---|---|---|
| Main | All Git, Terraform and Kubernetes demonstration commands | Keep open for the entire recording |
| Prometheus | Keeps localhost:9090 connected to Prometheus | Leave forwarding; do not type demo commands here |
| Grafana | Keeps localhost:3001 connected to Grafana | Leave forwarding; do not type demo commands here |

**RUN in Main:**

```powershell
$Host.UI.RawUI.WindowTitle = 'HireFlow - MAIN'
Set-Location 'C:\Users\jagri\Documents\DevOps_Project'
$env:AWS_PROFILE = 'hireflow-admin'
git status --short
git pull --ff-only
kubectl get nodes
```

**CHECK**

- `git status --short` should normally print nothing: the tracked files have no local changes.
- Git pull should succeed or say Already up to date. If it fails, fix it before recording. Do not use reset or discard your changes.
- Kubernetes should list two nodes with STATUS **Ready**.

**Only if AWS reports an expired session, RUN in Main:**

```powershell
aws login --profile hireflow-admin --region ap-south-1
```

Complete the browser login, return to Main, and rerun `kubectl get nodes`. Do all sign-ins before recording.

### Step 3. Check the cluster and live application

**RUN in Main:**

```powershell
kubectl get pods -n hireflow
kubectl get application hireflow -n argocd -o wide
$lb = kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
Write-Output "Application URL: http://$lb"
./scripts/verify-app.ps1 -BaseUrl "http://$lb"
```

**CHECK**

- Backend and frontend each have at least two ready/running Pods. PostgreSQL is ready/running. At minimum replicas this is five Pods total.
- Argo CD says **Synced** and **Healthy**.
- The script prints PASS results for frontend, health, database readiness, metrics and candidate operations, including deletion. It creates a fictional candidate and removes it afterward.

**DO** Copy the application URL printed above. Paste it into a browser tab and check that HireFlow loads. This is the public AWS application URL for your video. Use the freshly printed address, since load balancer names can change.

**Important: do not type `$lb` or `http://$lb` in Chrome.** Chrome cannot expand PowerShell variables. To copy the actual address automatically, run this in Main PowerShell:

```powershell
"http://$lb" | Set-Clipboard
```

Then switch to Chrome, press **Ctrl+L**, **Ctrl+V**, then **Enter**. The address must contain the real hostname ending in `elb.ap-south-1.amazonaws.com`, not the text `$lb`.

`$lb` is a variable holding that hostname. It exists only in this Main PowerShell session. Keep this window open.

### Step 4. Connect Prometheus

If a Prometheus window already shows `Forwarding from 127.0.0.1:9090`, leave it running and skip the command below.

Otherwise, **RUN in the Prometheus window:**

```powershell
$Host.UI.RawUI.WindowTitle = 'HireFlow - PROMETHEUS - leave running'
$env:AWS_PROFILE = 'hireflow-admin'
kubectl port-forward -n monitoring svc/prometheus-server 9090:80
```

**CHECK** It prints `Forwarding from 127.0.0.1:9090` and continues running. This is expected; the command does not return to a normal prompt while forwarding.

**DO** Open `http://localhost:9090/targets` in your browser. Find the target/job named **hireflow-backend**. Expand it if needed. Confirm it says **UP**. Leave that tab open.

### Step 5. Connect Grafana and sign in

If the Grafana window already shows `Forwarding from 127.0.0.1:3001`, leave it running and skip the command below.

Otherwise, **RUN in the Grafana window:**

```powershell
$Host.UI.RawUI.WindowTitle = 'HireFlow - GRAFANA - leave running'
$env:AWS_PROFILE = 'hireflow-admin'
kubectl port-forward -n monitoring svc/grafana 3001:80
```

The output may show `3001 -> 3000`. That is correct: local port 3001 forwards to Grafana's internal port.

**DO** Open `http://localhost:3001`. If already signed in, go directly to the dashboard.

If a password is needed, **RUN in Main, before recording:**

```powershell
$grafanaEncoded = kubectl get secret grafana -n monitoring -o jsonpath='{.data.admin-password}'
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($grafanaEncoded)) | Set-Clipboard
Remove-Variable grafanaEncoded
```

**DO** In Grafana, enter username `admin` and paste the clipboard into the password field. Click **Log in**. The commands copy the password without displaying it in the terminal.

**RUN in Main after login:**

```powershell
Set-Clipboard -Value ''
```

**DO** Open Grafana's dashboard search/navigation, search **HireFlow**, and open **HireFlow — Application Overview**. The separator in its displayed title may look like a dash; searching just HireFlow avoids confusion.

Select **Last 15 minutes** in the dashboard time picker. Refresh the dashboard.

If the panels need traffic, **RUN in Main:**

```powershell
1..60 | ForEach-Object {
    Invoke-WebRequest "http://$lb/health" -UseBasicParsing | Out-Null
    Start-Sleep -Milliseconds 200
}
```

Wait about 30–60 seconds, then refresh Grafana again. Prometheus is configured to scrape every 15 seconds.

**CHECK** The **HTTP Requests / sec**, **Request Latency p95**, and **Error Rate** panels contain data. Zero errors is fine. **No data** needs investigation before recording. Leave both monitoring terminals open.

### Step 6. Rehearse Terraform

**RUN in Main:**

```powershell
$env:AWS_PROFILE = 'hireflow-terraform'
terraform -chdir=terraform fmt -check
terraform -chdir=terraform validate
terraform -chdir=terraform plan -no-color
```

**CHECK** Formatting exits without error, validation says the configuration is valid, and the plan says **No changes. Your infrastructure matches the configuration.** Silent output from fmt is normal when it passes.

If this profile's login expired, run `aws login --profile hireflow-terraform --region ap-south-1`, complete login, then repeat the failed check.

**RUN after the check:**

```powershell
$env:AWS_PROFILE = 'hireflow-admin'
```

You will repeat fmt, validate and plan during the video. Do not run apply or destroy for this demonstration.

### Step 7. Arrange your browser tabs in this order

**DO** Open these pages now. Sign in wherever necessary before recording.

| Tab | Open this page | Prepare this view |
|---|---|---|
| 1 | https://github.com/Jagrit3500/DevOps_Capstone_Project | Repository Code page |
| 2 | https://github.com/Jagrit3500/DevOps_Capstone_Project/actions | Actions workflow list |
| 3 | https://github.com/Jagrit3500/DevOps_Capstone_Project/pkgs/container/hireflow-backend | Backend image versions/tags |
| 4 | https://github.com/Jagrit3500/DevOps_Capstone_Project/pkgs/container/hireflow-frontend | Frontend image versions/tags |
| 5 | AWS Console, VPC service | Your HireFlow VPC details |
| 6 | AWS Console, VPC service > Subnets | Filter to the HireFlow VPC; four subnets visible |
| 7 | AWS Console, EKS service | `hireflow-eks` cluster details |
| 8 | AWS EKS > `hireflow-eks` > Compute > `hireflow-workers` | Node group status/configuration |
| 9 | Application URL printed in Step 3 | HireFlow dashboard |
| 10 | Same application URL followed by `/metrics` | Prometheus-format metric text |
| 11 | http://localhost:9090/targets | hireflow-backend target UP |
| 12 | http://localhost:3001 | HireFlow dashboard with populated panels |

For AWS tabs, use the Console's top search box to open **VPC** or **EKS**. Select **Asia Pacific (Mumbai), ap-south-1** in the region selector. In VPC, select the HireFlow VPC and note its VPC ID; use that VPC ID to filter subnets. In EKS, select `hireflow-eks`, then use **Compute** to find the managed node group.

These pages are prepared tabs; do not try to show them all at once. You will visit them when the recording steps below say to.

### Step 8. Prepare the editor and Helm command

**DO** Confirm your editor shows the project folder tree. In VS Code, use **Ctrl+P**, type a file path, and press Enter to open that file quickly. Use **Ctrl+G** only if you need a specific line. No need to memorize line numbers; search inside the file for the named setting.

**RUN in Main:**

```powershell
$helmPath = Join-Path $env:TEMP 'helm-v3.22.1\windows-amd64\helm.exe'
Test-Path $helmPath
& $helmPath list -n hireflow
```

**CHECK** The path test prints True and Helm lists the `hireflow` release. If the test prints False but Helm is installed on PATH, use `$helmPath = (Get-Command helm -ErrorAction Stop).Source` and repeat the list command. If neither works, resolve Helm availability before recording.

Do not edit README or push your demonstration commit yet. That action must appear in the actual video.

### Step 9. Test the recorder, then return to slide 1

**DO**

1. Open **Snipping Tool** from Start and choose its **Record** mode, then **New**. On systems supporting it, **Windows+Shift+R** opens screen recording.
2. Select the screen area containing the applications you will show. Use the same area for slides, editor, browser and terminal.
3. Enable the microphone in the recording toolbar and select your microphone. If asked, grant microphone permission. Check these controls before the real take.
4. Make a 15-second test: say a sentence, switch from browser to Main PowerShell, then stop.
5. Play the test. Confirm your voice is audible and command text is readable. Adjust volume or font size if needed.
6. Close the test playback. Return to the PDF at slide 1. Ensure no password prompts or credentials are visible.

**Ready check:** app works; Argo Synced/Healthy; Terraform passed; Prometheus UP; Grafana panels populated; browser tabs signed in; Main and both forwarding windows still open.

---

## ACTUAL RECORDING — start here once setup is ready

### Step 10. Press Record and give the introduction on slide 1

**DO** Start the real screen recording with microphone enabled. Show slide 1. Allow a short moment after recording starts, then speak.

**SAY**

> Hello, my name is Jagrit. This is HireFlow, my DevSecOps capstone project. HireFlow tracks candidates through a recruitment pipeline. I will explain the application and architecture, then demonstrate a new Git change passing through tests, security scans, image publication and deployment to AWS. I will finish with the live application and monitoring.

### Step 11. Present slides 2–10

**DO** Move forward one PDF page at a time using the viewer's next-page control or Page Down. Keep each slide visible while speaking its paragraph.

**Slide 2 — Project overview. SAY**

> HireFlow lets a team create, view, update and remove candidates. Candidates move through applied, screening, interview, offer, hired and rejected stages. The dashboard retrieves its records and statistics from the backend API, and PostgreSQL stores the data. I use fictional candidate information for this demonstration.

**Slide 3 — Technology stack. SAY**

> The frontend uses React and Vite, with Nginx serving the application. The backend uses FastAPI, SQLAlchemy and Alembic, with PostgreSQL for storage. Docker packages the application. GitHub Actions runs CI, GHCR stores container images, Terraform provisions AWS, Helm defines Kubernetes resources, and Argo CD handles GitOps reconciliation. Prometheus and Grafana provide monitoring.

**Slide 4 — Architecture. DO** Point first to delivery, then to the request path. **SAY**

> There are two flows here. For delivery, a push triggers tests, image builds and security scans. CI publishes SHA-tagged images and commits their tags to production Helm values. Argo CD watches Git and applies the configuration to EKS. For user requests, traffic reaches the AWS load balancer and ingress-nginx. The root path serves the frontend, slash-api reaches the backend, and the backend connects to PostgreSQL.

**Slide 5 — CI/CD pipeline. SAY**

> This slide shows an earlier successful pipeline. In the live demonstration I will trigger a new run and show it completing. Backend tests and frontend checks precede image builds. Both image scans must pass before publication and promotion. Git commit SHA tags make the running version traceable to its source.

**Slide 6 — DevSecOps gates. SAY**

> Trivy scans operating-system and installed application packages in both runtime images. HIGH or CRITICAL findings fail the pipeline. The frontend also runs npm audit. The runtime images use non-root users. Passwords come from environment configuration locally and Kubernetes Secrets in the cluster. GitHub Actions uses AWS OIDC instead of a stored long-lived AWS access key. Scan results reflect the scanner database at the time of the run.

**Slide 7 — Infrastructure. SAY**

> Terraform provisions the network and EKS infrastructure in Mumbai. The VPC has two public and two private subnets across two Availability Zones. Worker nodes run in private subnets, with a NAT gateway for outbound connectivity. The stack includes IAM roles, the EBS CSI driver for persistent storage, and Metrics Server. The worker node group has a desired size of two.

**Slide 8 — Kubernetes and GitOps. SAY**

> Backend and frontend each have a minimum of two replicas. Readiness probes check whether the application can serve requests, while liveness probes check that it is alive. The application Services use ClusterIP, and Ingress supplies public routing. Helm defines the resources, and Argo CD renders the chart and reconciles it from Git. I will show the current sync state and image tags shortly.

**Slide 9 — Observability. SAY**

> The backend exposes Prometheus metrics. Prometheus scrapes them, and Grafana displays HTTP request rate, p95 latency and server error rate. These help observe traffic, response times and failures after deployment.

**Slide 10 — Demo and lessons. SAY**

> One issue was PostgreSQL initialization on an EBS volume containing lost plus found. Setting PGDATA to a subdirectory fixed it. Another issue was frontend startup before the backend was ready; Compose now waits for database readiness. My main lesson is to verify each stage with tests, scans, plans, rollout status and live metrics. Production improvements would include authentication, TLS, backups and stronger database availability.

**SAY to transition**

> I will now demonstrate the live project, starting with the codebase.

### Step 12. Show the codebase in your editor

**DO** Switch to your editor. Show the folder tree first.

**SAY**

> The repository separates application code, infrastructure, deployment configuration and monitoring. Backend and frontend contain the application. Terraform provisions AWS. Helm and Kubernetes files define workloads and networking. The workflow automates delivery, and Argo CD manages reconciliation from Git.

**DO** Open each file using Ctrl+P. Keep the code visible for several seconds while speaking. Do not open `.env`, credentials, Terraform state or live Secret values.

| Open in this order | What to point at | SAY |
|---|---|---|
| `backend/app/routes.py` | Candidate routes and statistics | “These routes implement candidate create, read, update and delete operations, filtering and pipeline statistics.” |
| `backend/app/models.py` | Candidate model fields | “SQLAlchemy defines the stored candidate fields.” |
| `backend/alembic/versions/001_initial.py` | Table creation | “Alembic versions the database schema through migrations.” |
| `backend/conftest.py` | Test database/session fixtures | “Tests use an isolated SQLite database rather than the cloud database.” |
| `backend/tests/test_candidates.py` | Example CRUD and validation tests | “The current backend suite has twenty-two cases covering health, candidate operations, validation and other application behavior.” |
| `frontend/src/api.js` | Relative `/api` calls and error handling | “The frontend uses relative API paths instead of embedding an AWS hostname and reports failed responses.” |
| `backend/Dockerfile` | Runtime user | “The backend image runs as a non-root user.” |
| `frontend/Dockerfile` | Build/runtime stages and user | “The frontend uses a multi-stage Node build and Nginx runtime, also as a non-root user.” |
| `docker-compose.yml` | Health checks and dependencies | “Compose starts PostgreSQL first and waits for database and backend readiness before the frontend starts.” |
| `.github/workflows/ci.yml` | Jobs and dependencies | “The workflow connects tests, frontend checks, image builds, scans, publication and GitOps promotion. I will trigger it now.” |

### Step 13. Make the live README edit

**DO** Stay in the editor. Open `README.md`. Find the **Quick Start / Run Locally** section. After the startup command block, before the service URL table, add this sentence on its own line:

> During local startup, Compose waits for the backend database readiness check before starting the frontend.

Press **Ctrl+S** to save. This sentence was absent when this guide was prepared. If you already added it in an earlier take, make a different useful documentation edit; do not add a duplicate just to create a change.

**SAY**

> I am adding a documentation note explaining local startup readiness. I will commit and push this change to demonstrate the real pipeline trigger. The workflow runs for documentation pushes to main too.

### Step 14. Commit and push in Main PowerShell

**DO** Switch to the window titled **HireFlow - MAIN**. Run the commands below one at a time, waiting for each to finish.

```powershell
git diff -- README.md
git add -- README.md
git commit -m "Clarify local startup readiness for the walkthrough"
$demoSha = git rev-parse HEAD
git push origin main
Write-Output "Application source SHA for this video: $demoSha"
```

**CHECK** The diff shows your sentence, the commit succeeds, and the push succeeds. Do not put `[skip ci]` in this demonstration commit message.

`$demoSha` now stores the application source commit. Keep this Main window open until the end; you will compare it with the deployed image tags.

### Step 15. Show the new GitHub Actions run completing

**DO** Switch to browser tab 2, **Actions**, and refresh. Open the new **HireFlow CI/CD Pipeline** run with your commit message and source commit. Do not open the cleanup commit or an older successful run.

Older red crosses in the Actions history mean those particular earlier runs failed. They are historical records, not a statement that the current app is down. Scroll to the top to find the newest run and match its source commit. Leave the history intact; the recording should show the new run completing successfully.

**SAY**

> This is the run triggered by my new push. Backend tests run first, followed by frontend dependency audit, build and tests. Both container images are then built and scanned. Publication happens only after both security gates pass. The final job promotes their tags in Git and waits for Argo CD to deploy the expected images.

**DO** Visit the jobs and steps below as they complete. Click a job in the run summary, then expand the named log step. Leave each relevant result visible long enough to read.

| Job | Open these steps | What you need to show |
|---|---|---|
| Backend tests | Run tests | Passing pytest result |
| Build React frontend | Audit frontend dependencies, Build frontend, Test frontend API pagination | Audit, build and tests succeeded |
| Build, scan and publish images | Build backend image, Build frontend image, Scan backend image, Scan frontend image, Publish scanned images | Both builds and both Trivy gates passed; images published |
| Promote image tags through GitOps | Promote scanned images in Git, Wait for Argo CD deployment | Promotion and rollouts succeeded |

While waiting, you can explain the visible workflow settings. Keep the recording running so the new trigger and completion are included. The deployment step can wait several minutes for reconciliation and rollout.

**CHECK** The run finishes green and all four jobs succeed. A skipped deployment is not evidence of a completed deployment. If it fails, use the problem-help section at the bottom of this same page.

**Only after success, SAY**

> The new pipeline has completed successfully. Tests, frontend checks and both image security gates passed. Both images were published, and the promotion job confirmed the expected deployments completed their rollouts.

### Step 16. Show published images and the promotion commit

**DO** Open backend package tab 3 and refresh. Find the full tag equal to `$demoSha`. Show it. Repeat in frontend package tab 4. If useful, return briefly to Main and run `Write-Output $demoSha` to make the comparison readable.

**SAY**

> Both image tags identify the source commit I just pushed. Using the same SHA for both images makes the deployed application traceable.

**DO** Open repository tab 1, refresh, and navigate to `helm` > `hireflow` > `values-prod.yaml`. Show the backend and frontend tags matching `$demoSha`. Show the bot's promotion commit through the file's latest-commit link or repository history.

**SAY**

> CI committed the scanned image tags to production Helm values. The promotion commit has its own Git SHA because it records a configuration change. The image tags still identify the original application source commit.

Use the GitHub copy of production values here. The local file may still show the previous version because the bot pushed after your local commit.

### Step 17. Show Terraform code, then run the checks

**DO** Switch to your editor. Open `terraform/variables.tf`, then `terraform/vpc.tf`, then `terraform/eks.tf`. Point to the region/capacity variables, subnets and cluster/node group resources.

**SAY**

> Terraform variables configure deployment settings such as region, network ranges and worker capacity. These resources define the VPC, subnets, EKS cluster and node group. Infrastructure changes are reviewed through a plan before application.

**DO** Switch to Main. Run each command separately.

```powershell
$env:AWS_PROFILE = 'hireflow-terraform'
terraform -chdir=terraform fmt -check
terraform -chdir=terraform validate
terraform -chdir=terraform plan -no-color
```

**CHECK** Wait for the final plan summary. It should say **No changes**.

**Only after successful checks, SAY**

> Formatting and validation passed. The current plan reports no changes because the AWS infrastructure is already provisioned and matches the configuration. This is a drift check, not the original creation plan.

Do not run Terraform apply or destroy.

### Step 18. Show AWS Console proof

**DO** Switch to the prepared AWS tabs. Confirm the region selector shows Mumbai.

1. **VPC tab:** select the HireFlow VPC and show its name, VPC ID and CIDR.
2. **Subnets tab:** show the four subnets belonging to that VPC: two public and two private, across two Availability Zones.
3. **EKS tab:** show `hireflow-eks`, status **Active**.
4. **Node group tab:** show `hireflow-workers`, status **Active**, desired size **2** and the configured instance type. The current project config uses `c7i-flex.large`; read the live value if you mention it.

**SAY as you visit the tabs**

> These Console pages confirm the Terraform-managed resources exist in Mumbai. Here is the HireFlow VPC and its public and private subnets. The EKS cluster is Active, and its worker node group is Active with a desired size of two. This is the deployed infrastructure defined in the repository.

### Step 19. Show Argo CD sync and the deployed SHA

**DO** Return to Main. Switch the profile back before Kubernetes commands.

```powershell
$env:AWS_PROFILE = 'hireflow-admin'
kubectl get application hireflow -n argocd -o wide
kubectl get deployment backend frontend -n hireflow -o wide
Write-Output "Expected application source SHA: $demoSha"
```

**CHECK** Argo CD says **Synced / Healthy**. Backend and frontend images end with the same SHA printed above, and their replicas are available.

If the wide image columns wrap too much, run this clearer version as well:

```powershell
kubectl get deployment backend frontend -n hireflow -o 'custom-columns=NAME:.metadata.name,IMAGE:.spec.template.spec.containers[0].image,AVAILABLE:.status.availableReplicas'
```

**SAY**

> Argo CD reports Synced and Healthy. Synced means the managed resources match the desired Git configuration, and Healthy means Argo's resource health checks pass. Both deployments use the application source SHA from the pipeline I just triggered, and their replicas are available. Argo's revision identifies the Git configuration commit, so it can differ from the image tag.

**DO** Switch to your editor and open `argocd/hireflow-application.yaml`.

**SAY**

> This Application follows main and uses the HireFlow Helm chart and production values. Automated reconciliation and self-healing keep resources aligned with Git. CI changes the image tags in Git, and Argo CD applies the resulting manifests.

The terminal output is your Argo CD sync demonstration. No extra Argo CD browser login or port-forward is needed for this flow.

### Step 20. Show Kubernetes resources one command at a time

**DO** Return to Main. Run a command, explain its output, then continue to the next command. You do not need to run local Docker Compose during this cloud walkthrough.

**RUN**

```powershell
kubectl get nodes
```

**CHECK:** Ready. **SAY:** “The Kubernetes worker nodes are Ready.”

**RUN**

```powershell
kubectl get pods -n hireflow
```

**CHECK:** Pods ready/running. **SAY:** “Backend and frontend each have at least two ready Pods, and PostgreSQL is running.”

**RUN**

```powershell
kubectl get svc -n hireflow
```

**CHECK:** backend/frontend ClusterIP. **SAY:** “The application Services are internal ClusterIP Services.”

**RUN**

```powershell
kubectl get ingress -n hireflow
```

**CHECK:** hireflow Ingress with address. **SAY:** “Ingress supplies the application's public routing through ingress-nginx.”

**RUN**

```powershell
kubectl get pvc -n hireflow
```

**CHECK:** database claim Bound. **SAY:** “PostgreSQL's volume claim is Bound, so its data uses persistent storage.”

**RUN**

```powershell
kubectl get hpa -n hireflow
```

**CHECK:** CPU metrics available. **SAY:** “The autoscalers have CPU metrics. At light traffic they can remain at their minimum of two replicas.”

**RUN**

```powershell
& $helmPath list -n hireflow
```

**CHECK:** hireflow release listed. **SAY:** “Helm lists the original chart release. Argo CD now manages subsequent manifest reconciliation.”

**DO** In the editor, open `helm/hireflow/templates/ingress.yaml`, then `helm/hireflow/templates/backend-deployment.yaml`. Find `readinessProbe` in the second file.

**SAY**

> The ingress template routes the root path to the frontend and slash-api to the backend. The backend readiness probe calls slash-ready to check database connectivity.

### Step 21. Demonstrate the live application

**DO** Return to Main and refresh the hostname:

```powershell
$lb = kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
Write-Output "http://$lb"
```

Open that address in application browser tab 9 and refresh. Clear any old search text or status/priority filters so all records are visible.

**SAY**

> This is HireFlow running through the public Ingress. I will create a fictional candidate, update their stage, search for them and remove the record. These operations use the live API and PostgreSQL database.

**DO: create the candidate**

1. Click **Add Candidate** near the top of the app.
2. Fill these fields:

| Field | Enter/select |
|---|---|
| Full Name | Demo Candidate |
| Email | demo.video@example.com |
| Position | DevOps Engineer |
| Department | Platform |
| Status | applied |
| Priority | medium |
| Resume URL | Leave empty |
| Notes | Fictional candidate for the capstone video. |

3. Click **Add Candidate** in the form.
4. If the email already exists, change it to `demo.video2@example.com` and submit again.
5. Show the new row and the Total/Applied statistics. Pause briefly so the viewer can see them.

**SAY:** “The candidate was created and appears in the list and statistics.”

**DO: update the candidate**

1. Find the Demo Candidate row. You may type `Demo Candidate` into **Search candidates** to locate it.
2. Click its pencil button. Hovering it shows **Edit**.
3. Set **Status** to `interview` and **Priority** to `high`.
4. Click **Save Changes**.
5. Show the updated stage and priority in the row.

**SAY:** “The update changed the candidate's stage to interview and priority to high.”

**DO: search and remove**

1. If not already entered, type `Demo Candidate` in the search box.
2. Show the matching row.
3. Click its trash button, titled **Remove**.
4. In the confirmation dialog, verify the candidate name, then click **Remove**.
5. Show that the searched record is gone. Clear the search afterward to restore the full list.

**SAY**

> Search located the candidate, and removal deleted the demonstration record. The dashboard uses API responses rather than fixed candidate counts.

### Step 22. Show metrics and Prometheus

**DO** Open metrics browser tab 10, using the current application hostname followed by `/metrics`. Show a few lines of Prometheus metric text. Use Ctrl+F and search `http_requests_total` if helpful.

If the browser text is hard to read, show this in Main instead:

```powershell
$metricsText = (Invoke-WebRequest "http://$lb/metrics" -UseBasicParsing).Content
$metricsText -split '\r?\n' | Select-String '^http_requests_total' | Select-Object -First 8
```

**SAY:** “The backend exposes Prometheus-formatted HTTP request metrics.”

**DO** Open Prometheus tab 11, `http://localhost:9090/targets`. Refresh and show **hireflow-backend UP**. Expand the target section if necessary.

**SAY**

> Prometheus reports the hireflow-backend target as UP, showing that scraping is successful. These collected metrics supply the Grafana dashboard. The candidate operations and readiness checks separately demonstrate application and database behavior.

### Step 23. Show Grafana

**DO** In Main, generate a little recent traffic:

```powershell
1..60 | ForEach-Object {
    Invoke-WebRequest "http://$lb/health" -UseBasicParsing | Out-Null
    Start-Sleep -Milliseconds 200
}
```

**SAY while it runs**

> I am sending sixty health requests to make a small amount of demonstration traffic visible. This demonstrates request metrics; it is not a stress test or evidence of autoscaler scaling.

**DO** Open Grafana tab 12. Confirm you are on the HireFlow application dashboard. Set **Last 15 minutes**, wait for a scrape interval, and click refresh. If needed, wait 30 seconds and refresh again.

Point to **HTTP Requests / sec**, then **Request Latency p95**, then **Error Rate**.

**SAY**

> Grafana queries Prometheus for these panels. HTTP Requests per second shows request rate. The p95 latency is a response-time threshold below which approximately ninety-five percent of observed requests fall. Error Rate shows server errors, and zero is a valid result. These panels help monitor the application after delivery.

Use only numbers actually visible if you choose to quote values. Do not describe a No data panel as working.

### Step 24. Give your closing words

**DO** Keep the Grafana dashboard visible, or return to the live application.

**SAY**

> This completes the HireFlow walkthrough: application code, a new pipeline run, automated tests, security scans, image publication, Terraform infrastructure, GitOps deployment, Kubernetes and live monitoring. The project demonstrates delivery of a working recruitment application. Future production improvements include authentication, TLS, backups and stronger database availability. I will preserve the stack for evaluation and perform a verified teardown afterward. Thank you.

Pause briefly, then stop the recorder.

---

## AFTER RECORDING

### Step 25. Save and review the actual video

**DO** In Snipping Tool's playback window, use Save / Save as and save the recording here:

```text
C:\Users\jagri\Documents\DevOps_Project\demo\HireFlow_Walkthrough.mp4
```

This is the normal project `demo` folder, not `.local-notes\demo`.

Play the saved file and check:

- Your introduction and slides are included.
- The new README edit, push and the same new run's successful completion are visible.
- Terraform, AWS, Argo CD, Kubernetes, candidate operations, Prometheus and Grafana are included in order.
- Your voice is audible and command text can be read.
- No passwords, tokens or credentials appear.
- The ending was saved, and the file plays correctly.

Do not assume that saving locally uploaded the recording. The MP4 must still be uploaded to the repository or a GitHub release asset and linked from the README/evidence index. Keep your original local recording until upload and playback have been verified. If you want help with that upload, provide the saved MP4 path after recording; this guide does not push a potentially large video automatically.

### Step 26. Stop monitoring access after you finish

**DO** Click inside the Prometheus PowerShell window and press **Ctrl+C**. Repeat in the Grafana PowerShell window. This stops only the local forwards; it does not destroy the cluster.

Keep the AWS infrastructure for evaluation as planned. Teardown is a separate post-evaluation step.

---

## Problem help — use only if something interrupts the flow

| Problem | What to do |
|---|---|
| AWS session expired | Before recording, log in with the profile needed: `aws login --profile hireflow-admin --region ap-south-1` for Kubernetes, or `aws login --profile hireflow-terraform --region ap-south-1` for Terraform. Repeat the failed command afterward. |
| localhost Prometheus/Grafana page does not open | Check its forwarding terminal is still running. If it exited, rerun its Step 4 or Step 5 command. If the message says the local port is already in use, check whether your earlier forward is still running rather than starting another one. |
| Grafana No data | Confirm Prometheus backend target is UP. Generate the health traffic, choose Last 15 minutes, wait 30–60 seconds and refresh. If still empty, fix monitoring before the final take. |
| `git commit` says nothing to commit | Save README in the editor, then check `git diff -- README.md`. If the sentence is already present from an earlier take, make another useful documentation change. |
| `git push` is rejected because the remote is newer | Do not force push. If your working tree is clean after your commit, run `git pull --rebase origin main`, resolve any conflict if you understand it, then set `$demoSha = git rev-parse HEAD` again and retry the push. If unsure, stop the take and get the conflict resolved first. |
| Actions run is not visible | Refresh Actions and confirm your push succeeded. Match the commit message and source SHA; do not use the `[skip ci]` cleanup commit. |
| Pipeline fails or promotion is skipped | Open the failing/skipped job and determine the cause. Do not claim it passed. Resolve it, then record a successful new trigger and complete run for the final walkthrough. |
| Argo is temporarily OutOfSync/Progressing | Wait briefly and rerun the status commands. Continue only after the expected version is Synced/Healthy. Persistent failure needs investigation. |
| `$lb`, `$demoSha` or `$helmPath` is empty | You may have changed terminals. Return to Main. Recreate `$lb` using the hostname command and `$helmPath` using Step 8. Recover `$demoSha` from the successful Actions run's source commit; do not use the bot promotion SHA. |
| Candidate is not visible | Clear search/status/priority filters. Check whether the form showed a validation or duplicate-email error. |
| You make a small speaking mistake | Correct yourself and continue. If a real check fails or credentials appear, fix the issue and make a clean final take. |

**Complete screen order:** Slides > Codebase > README edit and push > New CI run > GHCR and promotion > Terraform > AWS Console > Argo CD > Kubernetes > Live application > Metrics and Prometheus > Grafana > Closing.
