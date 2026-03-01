A **production-grade URL shortener** built with **Node.js + Express + PostgreSQL**, scalably deployed on **AWS** using **EC2, RDS, ALB, and Nginx**—automated end-to-end with a single bash script.

---

##  What's in This Repo

### Application — URL Shortener API

A lightweight RESTful API that shortens URLs and serves redirects.

| File | Description |
|---|---|
| `src/index.js` | Express app entry point, starts server on `PORT` (default `3000`) |
| `src/routes.js` | API routes: `POST /shorten`, `GET /go/:path`, `GET /` (ALB health check) |
| `src/db.js` | PostgreSQL connection pool via `pg`, reads credentials from `.env` |
| `src/utils.js` | Generates random three-word short paths (e.g. `apple.vulture.monkey`) |
| `tests/urlShortener.test.js` | Jest + Supertest integration tests |

**Tech stack:** Node.js · Express · PostgreSQL (`pg`) · `dotenv` · `cors` · Jest · Supertest

### API Endpoints

```
POST /shorten          — Accepts { "originalUrl": "..." }, returns { "shortUrl": "/go/..." }
GET  /go/:path         — Redirects to the original URL
GET  /                 — Health check (returns 200 OK for ALB)
```

### AWS Infrastructure

The app is **scalably deployed on AWS** using the following services:

| Service | Role |
|---|---|
| **EC2** (Amazon Linux 2023, `t2.micro`) | Runs the Node.js application |
| **RDS** (PostgreSQL, `db.t4g.micro`) | Managed PostgreSQL database, privately networked |
| **ALB** (Application Load Balancer) | Distributes traffic across EC2 instances, exposes a public DNS |
| **Nginx** | Reverse proxy on EC2, forwards port 80 → Node.js on port 3000 |
| **PM2** | Process manager — keeps the app alive and auto-restarts on reboot |
| **VPC / Security Groups** | Private networking between EC2 and RDS; ALB-only inbound traffic to EC2 |

### Architecture

```
Internet
   │
   ▼
AWS ALB  (public DNS, port 80/443)
   │
   ▼
EC2 Instance  (Amazon Linux 2023)
   ├── Nginx  (reverse proxy, port 80 → 3000)
   └── Node.js + PM2  (port 3000)
         │
         ▼
   RDS PostgreSQL  (private subnet, port 5432)
```

---

##  Automated Deployment — `start.sh`

The entire EC2 setup is automated by a single **bash script** (`start.sh`). After SSH-ing into a fresh EC2 instance, you upload and run it once — it handles everything:

1. **System update** — `dnf update` to patch the OS.
2. **Install dependencies** — installs `git`, `curl`, and `nginx` via `dnf`.
3. **Install Node.js 18** — pulls from the NodeSource RPM repository.
4. **Clone or update the repo** — clones into `/var/www/myapp`; if the directory already exists, it resets and pulls the latest `main` branch instead (zero-downtime redeploys).
5. **Install npm packages** — runs `npm install` inside the app directory.
6. **Create `.env`** — writes database credentials (host, user, password, port) to `/var/www/myapp/.env` if one doesn't already exist.
7. **Configure Nginx** — writes a reverse-proxy config to `/etc/nginx/conf.d/myapp.conf` that forwards all HTTP traffic on port 80 to `localhost:3000`, passing the correct `Host`, `X-Real-IP`, and `X-Forwarded-For` headers.
8. **Restart & enable Nginx** — applies the new config and enables Nginx to start on boot via `systemctl`.
9. **Start app with PM2** — installs PM2 globally, starts `src/index.js` as `myapp`, saves the process list, and registers a `systemd` startup hook so the app survives reboots.

**To deploy, transfer the script and run it:**

```sh
# Secure the key and copy the script to EC2
chmod 400 /path/to/your-key.pem
scp -i /path/to/your-key.pem start.sh ec2-user@<EC2-PUBLIC-IP>:/home/ec2-user/

# SSH in and execute
ssh -i /path/to/your-key.pem ec2-user@<EC2-PUBLIC-IP>
chmod +x start.sh && ./start.sh
```

> **Note:** `start.sh` contains sensitive credentials and is excluded from version control via `.gitignore`.

**Useful commands once deployed:**

```sh
pm2 list                          # Check running processes
pm2 logs --lines 100              # Tail app logs
cat ~/.pm2/logs/myapp-error.log   # View error logs
sudo nginx -t                     # Validate Nginx config
sudo systemctl restart nginx      # Restart Nginx
```

---

## Database Setup (RDS PostgreSQL)

Once connected to your RDS instance via EC2, create the database and table:

```sql
CREATE DATABASE url_shortener;
\c url_shortener;

CREATE TABLE urls (
    id           SERIAL PRIMARY KEY,
    short_path   TEXT UNIQUE NOT NULL,
    original_url TEXT NOT NULL
);
```

Connect to RDS from inside your EC2 instance:

```sh
psql -h <your-rds-endpoint> -U url_shortener -d url_shortener
```

---

##  Running Locally

### Prerequisites
- Node.js
- PostgreSQL (running locally)

### Steps

```sh
git clone https://github.com/jamezmca/aws-full-course.git
cd aws-full-course
npm install
```

Create a `.env` file:

```env
DB_USER=postgres
DB_HOST=localhost
DB_NAME=url_shortener
DB_PASSWORD=your_password
DB_PORT=5432
PORT=3000
```

Start the app:

```sh
npm run dev
```

### Test the API

```sh
# Shorten a URL
curl -X POST http://localhost:3000/shorten \
     -H "Content-Type: application/json" \
     -d '{"originalUrl": "https://example.com"}'

# Follow a short URL
curl -i http://localhost:3000/go/apple.vulture.monkey
```

### Run Tests

```sh
npm test
```

