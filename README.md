
# 🚀 Full-Stack App Deployment on AWS EC2 with GitHub Actions

This guide walks you through deploying a full-stack (MERN-style) application to an AWS EC2 instance using a GitHub self-hosted runner.

---

## 🛠️ 1. Setup the Full Stack App

Make sure your project has the following structure:

```

MernProjectDeploy/
├── client/        # React frontend (Vite or CRA)
├── server/        # Node.js backend (Express)
├── .github/
│   └── workflows/
│       └── main.yml
└── README.md

````

### Backend (Basic API Example)

Create a basic Express API in `server/index.js`:

```js
const express = require('express');
const app = express();
const cors = require('cors');
require('dotenv').config();
const PORT = process.env.PORT || 6000;

app.use(cors({
    origin: '*',
    methods: ['GET', 'POST', 'PUT', 'DELETE'],
}));

app.use(express.json());
app.use(express.urlencoded({ extended: true }));
app.use(express.static('public'));


app.get('/api/get', (req, res) => {
    res.status(200).json({ message: 'GET request successful' });
});
app.get('/api/health', (req, res) => {
    res.status(200).json({ status: 'OK', Message: 'Server is running' });
});


app.listen(PORT, () => {
    console.log(`Server is running on port ${PORT}`);
});
````

---

## ☁️ 2. Create EC2 Instance

* Go to [AWS EC2 Console](https://console.aws.amazon.com/ec2/)
* Launch an **Ubuntu 22.04** instance.
* Allow inbound rules for:

  * HTTP (80)
  * HTTPS (443)
  * Custom TCP (port 8000 or your backend port)
  * SSH (22) for access

---

## 📂 3. Push Code to GitHub

* Initialize a git repo (if not already):

  ```bash
  git init
  git remote add origin https://github.com/your-username/MernProjectDeploy.git
  git push -u origin main
  ```

---

## 🤖 4. Setup Self-Hosted Runner on EC2

SSH into your EC2 instance and set up the GitHub runner:

```bash
# From your EC2 terminal:
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-2.315.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.315.0/actions-runner-linux-x64-2.315.0.tar.gz
tar xzf actions-runner-linux-x64-2.315.0.tar.gz

# Connect your runner to GitHub (get token from repo -> Settings -> Actions -> Runners)
./config.sh --url https://github.com/your-username/MernProjectDeploy --token <TOKEN>

# Start the runner
sudo ./svc.sh install / start
```

---

## ⚙️ 5. GitHub Actions Workflow (`.github/workflows/deploy.yml`)

```yaml
name: Deploy Fullstack App on EC2

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: self-hosted

    strategy:
      matrix:
        node-version: [22.x]

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Use Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
          cache-dependency-path: |
            server/package-lock.json
            client/package-lock.json

      # ------------------------------
      # BACKEND SETUP
      # ------------------------------
      - name: Install backend dependencies
        working-directory: ./server
        run: npm install

      - name: Create backend .env file
        working-directory: ./server
        run: |
          touch .env
          echo "${{ secrets.PROD_BACKEND_ENV }}" > .env

      - name: Start backend with PM2
        working-directory: ./server
        run: |
          /home/ubuntu/.nvm/versions/node/v22.16.0/bin/pm2 restart backend || /home/ubuntu/.nvm/versions/node/v22.16.0/bin/pm2 start index.js --name backend

      # ------------------------------
      # FRONTEND SETUP
      # ------------------------------
      - name: Clean npm cache & remove existing dependencies
        working-directory: ./client
        run: |
          rm -rf node_modules package-lock.json
          npm cache clean --force

      - name: Install frontend dependencies
        working-directory: ./client
        run: npm install

      - name: Create frontend .env file
        working-directory: ./client
        run: |
          touch .env
          echo "${{ secrets.PROD_FRONTEND_ENV }}" > .env

      - name: Build frontend
        working-directory: ./client
        run: npm run build

      - name: Deploy frontend to Nginx
        run: |
          sudo rm -rf /var/www/html/*
          sudo cp -r ./client/dist/* /var/www/html/

```

---

## 📦 6. Install Dependencies on EC2

SSH into your EC2 instance and run:

```bash
curl -sL https://raw.githubusercontent.com/nvm-sh/nvm/v0.35.0/install.sh -o install_nvm.sh

bash install_nvm.sh

export NVM_DIR="$HOME/.nvm"
  [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
  [ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # This loads nvm bash_completion

#for latest version of node use 
nvm install --lts

# for specifi verison
nvm install 22 #mention version name

sudo apt-get update
sudo apt-get install -y nginx
sudo npm i -g pm2

#If sudo does't work install directly
```

---

## 🌐 7. Configure Nginx

Edit the Nginx config:

```bash
sudo nano /etc/nginx/sites-available/default
```

Add inside the `server {}` block:

```nginx
location /api {
    rewrite ^/api/(.*)$ /api/$1 break;
    proxy_pass http://localhost:8000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
```

Restart Nginx:

```bash
sudo systemctl restart nginx
```

---

## 🚀 8. Access Your App

* Frontend: `http://<your-ec2-ip>/`
* API Endpoint: `http://<your-ec2-ip>/api/health`

---

## ✅ Optional Enhancements

* Add HTTPS using Let's Encrypt with Certbot
* Use environment variables securely via GitHub Secrets
* Add CI checks (e.g., tests, lint) before deployment

---

## 📌 Notes

* Make sure your backend listens on `0.0.0.0` not `localhost`
* Frontend should use `/api` as the base for API calls
* Use `.env` with `VITE_API_URL=/api` to simplify backend URL handling

---

## 🧑‍💻 Author
Satyam Singh
Full Stack Developer
[LinkedIn](https://www.linkedin.com/in/satyam-singh-dev)
[YouTube](https://www.linkedin.com/in/satyam-singh-dev)
