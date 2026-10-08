# ISWE406P – Agile Development Process & DevOps – Assessment VIII

**Student:** Aditya (24MIS0292) · Slot L49+50

Three static web apps, each containerised with Docker, pushed to Docker Hub, and deployed to
Docker Desktop Kubernetes through a Jenkins pipeline.

| Folder | Question | App | Replicas | NodePort | Jenkins Script Path |
|---|---|---|---|---|---|
| `q1-notice-board/` | Q1 | College Notice Board | 2 | 30081 | `q1-notice-board/Jenkinsfile` |
| `q2-student-feedback/` | Q2 | Student Feedback | 3 | 30082 | `q2-student-feedback/Jenkinsfile` |
| `q3-placement-portal/` | Q3 | College Placement Portal | 3 | 30083 | `q3-placement-portal/Jenkinsfile` |

Each folder contains `index.html`, `Dockerfile`, `deployment.yaml` (Deployment + NodePort Service)
and `Jenkinsfile`. `q2/v2/` and `q3/v2/` hold the modified pages used for the "update" steps.
`diagrams/` holds the CI/CD pipeline pictures.

**Pipeline stages (all questions):** Clone Code → Build Docker Image → Test Application → Push Image → Deploy to Kubernetes → Verify Deployment

---

## 0. One-time setup

### 0.1 Install / check tools
- **Docker Desktop** (running), **Git**, **Java 17 or 21** (for Jenkins).
- Check: `docker version`, `git --version`, `java -version`.

### 0.2 Enable Kubernetes in Docker Desktop  *(Q1 step 5)*
Docker Desktop → ⚙ Settings → **Kubernetes** → tick **Enable Kubernetes** → *Apply & Restart*.
Wait until the bottom-left shows **Kubernetes running** (green). Then:
```powershell
kubectl config use-context docker-desktop
kubectl get nodes          # STATUS should be Ready
```

### 0.3 Docker Hub access token
hub.docker.com → Account settings → **Personal access tokens** → *Generate new token*
(Read, Write, Delete) → copy it. Use it as the password everywhere below.

### 0.4 Put your Docker Hub username into the files
Every `Jenkinsfile` and `deployment.yaml` contains the placeholder `DOCKERHUB_USERNAME`.
Replace it (PowerShell, from the repo root):
```powershell
Get-ChildItem -Recurse -Include Jenkinsfile,deployment.yaml |
  ForEach-Object { (Get-Content $_) -replace 'DOCKERHUB_USERNAME','adityarajarora123' | Set-Content $_ }
```
(Linux/macOS: `sed -i 's/DOCKERHUB_USERNAME/adityarajarora123/g' */Jenkinsfile */deployment.yaml`)

### 0.5 Push the code to GitHub  *(Q1 step 1, Q2 step 1, "all code on GitHub")*
Create an empty **public** repo on GitHub named `devops-assessment-8`, then:
```powershell
cd devops-assessment-8
git init
git add .
git commit -m "Assessment VIII: notice board, feedback app, placement portal"
git branch -M main
git remote add origin https://github.com/aditya-raj-arora/devops-assessment-8.git
git push -u origin main
```

### 0.6 Install & start Jenkins
Easiest and most reliable on Windows: run Jenkins **as your own user** so it can reach Docker Desktop and kubectl.
```powershell
# download jenkins.war (LTS) from https://www.jenkins.io/download/ then:
java -jar jenkins.war --httpPort=8080
```
Open http://localhost:8080 → paste the initial admin password printed in the terminal →
**Install suggested plugins** → create admin user.
Suggested plugins already include Pipeline, Git, Credentials Binding and Timestamper — nothing else is needed.

> If you installed Jenkins as a Windows *service* instead, it runs as Local System and usually
> cannot talk to Docker Desktop. Either stop the service and use `java -jar jenkins.war`, or
> change the service's "Log On" account to your Windows user.
> Port 8080 clashes with nothing in this project (apps use 30081–30083, tests use 8091–8093).

### 0.7 Jenkins credentials  *(Q1 step 7, Q2 step 5, Q3 step 7)*
Manage Jenkins → **Credentials** → System → Global credentials → **Add Credentials**:

| Kind | ID (must match) | Value |
|---|---|---|
| Username with password | `dockerhub` | Docker Hub username + access token |
| Secret file | `kubeconfig` | upload `C:\Users\<you>\.kube\config` |

📸 Screenshot this credentials list.

---

## Question 1 – College Notice Board

### Manual steps (build → run → push → deploy)
```powershell
cd q1-notice-board

# Step 2: build image
docker build -t adityarajarora123/notice-board:latest .

# Step 3: run locally and verify
docker run -d --name notice-test -p 8081:80 adityarajarora123/notice-board:latest
#   open http://localhost:8081   📸 screenshot
docker rm -f notice-test

# Step 4: push to Docker Hub
docker login -u adityarajarora123
docker push adityarajarora123/notice-board:latest

# Step 6 + 9: Deployment (2 replicas) + NodePort Service
kubectl apply -f deployment.yaml
kubectl get deployment notice-board
kubectl get svc notice-board-service
```

### Jenkins pipeline  *(step 8)*
1. Jenkins → **New Item** → name `q1-notice-board` → **Pipeline** → OK.
2. Pipeline section → Definition: **Pipeline script from SCM** → SCM: **Git**
   → Repository URL `https://github.com/aditya-raj-arora/devops-assessment-8.git`
   → Branch `*/main` → **Script Path** `q1-notice-board/Jenkinsfile` → Save.
3. **Build Now**. When green: 📸 *Stage View*, 📸 *Console Output* (scroll to the end:
   `SUCCESS: ... deployed. Open http://localhost:30081` and `Finished: SUCCESS`).

### Verify  *(steps 10, 11)*
```powershell
kubectl get pods -l app=notice-board      # 2 pods, STATUS Running   📸
kubectl get svc notice-board-service      # 80:30081/TCP
```
Open **http://localhost:30081** 📸

---

## Question 2 – Student Feedback (deploy, then update)

### Initial deploy  *(steps 1–9)*
```powershell
cd q2-student-feedback
docker build -t adityarajarora123/student-feedback:v1 .
docker run -d --name fb-test -p 8082:80 adityarajarora123/student-feedback:v1   # open http://localhost:8082 📸
docker rm -f fb-test
docker tag adityarajarora123/student-feedback:v1 adityarajarora123/student-feedback:latest
docker push adityarajarora123/student-feedback:v1
docker push adityarajarora123/student-feedback:latest
```
Create Jenkins job `q2-student-feedback` exactly as in Q1 with Script Path
`q2-student-feedback/Jenkinsfile` → **Build Now** (this applies the 3-replica Deployment + NodePort Service).
```powershell
kubectl get deployment student-feedback        # READY 3/3   📸
kubectl get pods -l app=student-feedback       # 3 Running   📸
```
Open **http://localhost:30082** 📸 (shows "Version 1.0").

### Update the application  *(steps 10–14)*
`v2/index.html` adds a **Course Rating Summary** (average + star distribution bars) and a
**Thank-you message** after submit.
```powershell
copy v2\index.html index.html            # (Linux/mac: cp v2/index.html index.html)
git add index.html
git commit -m "Q2: add course rating summary and thank-you message"
git push
```
Jenkins → `q2-student-feedback` → **Build Now**. The pipeline builds a new image tagged with the
build number, pushes it, and runs `kubectl set image` + `kubectl rollout status` (rolling update).

Manual equivalent if your faculty wants to see the commands:
```powershell
docker build -t adityarajarora123/student-feedback:v2 .
docker push adityarajarora123/student-feedback:v2
kubectl set image deployment/student-feedback student-feedback=adityarajarora123/student-feedback:v2
kubectl rollout status deployment/student-feedback
```
Verify: `kubectl get pods -l app=student-feedback` (3 new Running pods) and
**http://localhost:30082** now shows "Version 2.0" and the rating summary 📸 (submit the form once to show the thank-you card).

---

## Question 3 – End-to-End CI/CD for the Placement Portal

1. Jenkins → New Item `q3-placement-portal` → Pipeline → Pipeline script from SCM → same repo →
   Script Path `q3-placement-portal/Jenkinsfile` → Save.
2. **Build Now** once. (The first run also registers the `pollSCM('H/2 * * * *')` trigger
   written in the Jenkinsfile — after that, Jenkins checks GitHub every 2 minutes.)
3. Verify run 1: `kubectl get pods -l app=placement-portal` → 3 Running; open **http://localhost:30083** (3 companies).
4. **Modify the page on GitHub** *(step 9)*: either edit `q3-placement-portal/index.html` directly on
   github.com (pencil icon → add a table row → Commit), or locally:
   ```powershell
   copy q3-placement-portal\v2\index.html q3-placement-portal\index.html
   git commit -am "Q3: add TCS Digital opportunity"
   git push
   ```
5. **Do not click Build Now.** Within ~2 minutes Jenkins starts build #2 by itself
   (build page says *"Started by an SCM change"*) 📸 — this proves the automatic trigger *(steps 10, 11)*.
   Its console shows: new commit cloned → image `:2` built → pushed → `kubectl set image` → rollout complete.
6. Verify *(steps 12, 13)*:
   ```powershell
   kubectl get pods -l app=placement-portal   # 3 Running (new pod names)   📸
   kubectl rollout history deployment/placement-portal
   ```
   Open **http://localhost:30083** → TCS Digital row marked **(NEW)** 📸
7. Docker Hub → your `placement-portal` repo → **Tags** shows `1`, `2` and `latest` 📸

> GitHub *webhooks* can't reach `localhost:8080`, which is why the pipeline uses SCM polling.
> (Webhook alternative: expose Jenkins with ngrok and tick "GitHub hook trigger for GITScm polling".)

---

## Screenshot checklist (per question)

| Required | What to capture |
|---|---|
| Code on GitHub | repo page showing the 3 folders + one folder's files |
| Jenkins pipeline | job page → **Stage View** with all 6 stages green |
| Successful execution | Console Output → bottom (`SUCCESS: ...` + `Finished: SUCCESS`) |
| Docker Desktop dashboard | **Images** tab (your 3 images) and **Containers** tab (k8s pods running) |
| CI/CD pipeline picture | `diagrams/q1-cicd-pipeline.png`, `q2-…`, `q3-…` |
| Deployment on localhost | browser at `localhost:30081 / 30082 / 30083`, plus `kubectl get pods` terminal |

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `docker: command not found` / pipe error in Jenkins | Jenkins isn't running as your user; use `java -jar jenkins.war` (see 0.6). |
| `kubectl` not recognised in Jenkins | Docker Desktop installs it at `C:\Program Files\Docker\Docker\resources\bin`; add that to the system PATH and restart Jenkins. |
| `ErrImagePull` / `ImagePullBackOff` | Placeholder `DOCKERHUB_USERNAME` not replaced, or repo is private — make the Docker Hub repo public. |
| `provided port is already allocated` (NodePort) | Another service uses 30081–30083: `kubectl get svc -A`, delete the old one. |
| Test stage: `port is already allocated` | A leftover test container: `docker rm -f notice-board-test` (or the matching name). |
| Old page still visible after update | Hard refresh (Ctrl+F5); check `kubectl rollout status deployment/<app>`. |
| `docker login` fails | Use the access token, not your account password. |

### Cleanup after evaluation
```powershell
kubectl delete -f q1-notice-board/deployment.yaml -f q2-student-feedback/deployment.yaml -f q3-placement-portal/deployment.yaml
```
