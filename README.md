# MEAN Stack CRUD App – Docker + Jenkins + AWS Deployment

This is a simple Tutorial Management app built with the MEAN stack (MongoDB, Express, Angular, Node.js).  
I have containerized it using Docker and deployed it on an AWS EC2 instance with a Jenkins CI/CD pipeline and Nginx as a reverse proxy.

---

## How It Works

```
Browser → Port 80 → Nginx
                      ├── /api/*  →  Backend (Node.js on port 8080)
                      └── /*      →  Frontend (Angular)
                                        
Backend → MongoDB (inside Docker network)
```

Everything runs as Docker containers on one EC2 Ubuntu server. Only port 80 is open to the outside.

---

## Project Structure

```
.
├── backend/
│   ├── app/
│   │   ├── config/db.config.js     # reads MONGODB_URL from environment
│   │   ├── controllers/
│   │   ├── models/
│   │   └── routes/turorial.routes.js
│   ├── server.js
│   ├── package.json
│   └── Dockerfile
│
├── frontend/
│   ├── src/
│   │   ├── app/
│   │   │   ├── components/
│   │   │   ├── models/
│   │   │   └── services/tutorial.service.ts
│   │   └── index.html
│   ├── angular.json
│   ├── package.json
│   ├── nginx.conf
│   └── Dockerfile
│
├── nginx/
│   └── nginx.conf       # reverse proxy config
│
├── docker-compose.yml
├── Jenkinsfile
└── README.md
```

---

## Running Locally

Make sure you have **Docker** and **Git** installed.

```bash
git clone https://github.com/<your-username>/<repo-name>.git
cd <repo-name>

export DOCKER_USERNAME=your_dockerhub_username

# Build and start all containers
docker compose build
DOCKER_USERNAME=$DOCKER_USERNAME IMAGE_TAG=latest docker compose up -d
```

Open **http://localhost** in your browser.

To stop:
```bash
docker compose down
```

---

## Docker Setup

### Backend (`backend/Dockerfile`)

- Built on **Node 18 Alpine** (lightweight)
- Uses a two-stage build to keep the image small
- Listens on port **8080**
- Connects to MongoDB using the `MONGODB_URL` environment variable

### Frontend (`frontend/Dockerfile`)

- **Stage 1:** Node 18 builds the Angular app (`npm run build`)
- **Stage 2:** The compiled files are served using **nginx:alpine**

### Building and Pushing Images Manually

```bash
export DOCKER_USERNAME=your_dockerhub_username

docker build -t $DOCKER_USERNAME/mean-backend:latest ./backend
docker build -t $DOCKER_USERNAME/mean-frontend:latest ./frontend

docker login
docker push $DOCKER_USERNAME/mean-backend:latest
docker push $DOCKER_USERNAME/mean-frontend:latest
```

---

## AWS EC2 Setup

### Step 1 – Launch an EC2 Instance

- **OS:** Ubuntu Server 22.04 LTS
- **Type:** t3.small (minimum)
- **Security Group – open these ports:**

| Port | Who can access | Why |
|---|---|---|
| 22 | Your IP only | SSH into the server |
| 80 | Everyone | Access the app |

### Step 2 – Install Docker on the Server

SSH into your EC2 instance and run:

```bash
sudo apt-get update -y
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker ubuntu
newgrp docker

# Verify
docker --version
docker compose version
```

### Step 3 – Deploy the App

```bash
git clone https://github.com/<your-username>/<repo-name>.git
cd <repo-name>

export DOCKER_USERNAME=your_dockerhub_username
DOCKER_USERNAME=$DOCKER_USERNAME IMAGE_TAG=latest docker compose pull
DOCKER_USERNAME=$DOCKER_USERNAME IMAGE_TAG=latest docker compose up -d
```

### Step 4 – Check Everything is Running

```bash
docker compose ps
```

All 4 containers (mongodb, backend, frontend, nginx) should show **Up**.

Visit **http://\<your-ec2-public-ip\>** in your browser.

---

## Nginx Reverse Proxy

The `nginx/nginx.conf` file handles all incoming traffic on port 80:

- Requests starting with `/api/` go to the **backend** container
- Everything else goes to the **frontend** container

The frontend also has its own `nginx.conf` that makes sure Angular page routing works correctly (`try_files $uri $uri/ /index.html`).

---

## Jenkins CI/CD Pipeline

The `Jenkinsfile` sets up an automated pipeline that runs every time you push code to GitHub.

### What the pipeline does, step by step:

1. **Checkout** – pulls the latest code from GitHub
2. **Docker Login** – logs into Docker Hub
3. **Build Backend Image** – builds and tags the backend Docker image
4. **Build Frontend Image** – builds and tags the frontend Docker image
5. **Push to Docker Hub** – pushes both images
6. **Deploy to EC2** – SSHs into the server, pulls the new images, and restarts the containers

### Credentials to add in Jenkins

Go to **Manage Jenkins → Credentials** and add these:

| Credential ID | Type | What to put |
|---|---|---|
| `docker-hub-username` | Secret text | Your Docker Hub username |
| `docker-hub-password` | Secret text | Your Docker Hub password or token |
| `aws-vm-ssh-key` | SSH private key | Your EC2 `.pem` key |
| `vm-host-ip` | Secret text | Your EC2 public IP address |

### Creating the Pipeline Job

1. Click **New Item** → give it a name → choose **Pipeline**
2. Under Pipeline, set Definition to **Pipeline script from SCM**
3. SCM: **Git** → paste your GitHub repo URL
4. Script Path: `Jenkinsfile`
5. Click **Save**, then **Build Now**

### Auto-trigger on GitHub Push

1. In your GitHub repo: **Settings → Webhooks → Add webhook**
2. Payload URL: `http://<jenkins-server-ip>:8080/github-webhook/`
3. Content type: `application/json`
4. Events: select **Just the push event**
5. In Jenkins job settings: enable **GitHub hook trigger for GITScm polling**

Now every `git push` will automatically trigger a new build and deployment.

---

## API Endpoints

All endpoints are under `/api/tutorials`

| Method | URL | What it does |
|---|---|---|
| GET | `/api/tutorials` | Get all tutorials |
| GET | `/api/tutorials/:id` | Get one tutorial by ID |
| POST | `/api/tutorials` | Add a new tutorial |
| PUT | `/api/tutorials/:id` | Update a tutorial |
| DELETE | `/api/tutorials/:id` | Delete one tutorial |
| DELETE | `/api/tutorials` | Delete all tutorials |
| GET | `/api/tutorials?title=xyz` | Search by title |

---

## Troubleshooting

```bash
# See which containers are running
docker compose ps

# Check logs for any errors
docker compose logs -f backend
docker compose logs -f nginx
docker compose logs -f mongodb

# Test if the API is working
curl http://localhost/api/tutorials

# Restart a container
docker compose restart backend
```

**MongoDB connection error?**  
Make sure `MONGODB_URL` is set to `mongodb://mongodb:27017/dd_db` — the hostname `mongodb` must exactly match the service name in `docker-compose.yml`.

---

## Screenshots

> Add your screenshots here after you deploy. Here's what to capture:

1. GitHub repo showing all the files
2. Docker Hub with the pushed images
3. Jenkins – pipeline stages view (all green)
4. EC2 – Security Group settings
5. Terminal – `docker compose ps` showing all containers running
6. App – Tutorials list page
7. App – Add tutorial page
8. App – Edit/details page
