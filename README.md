<h1 align="center">nginx-proxy</h1>

<p align="center">
  <a aria-label="docker" href="https://www.docker.com/">
    <img src="https://img.shields.io/badge/-docker-2496ED.svg?logo=docker&style=for-the-badge&labelColor=000000" alt="docker">
  </a>
  <a aria-label="nginx" href="https://nginx.org/">
    <img src="https://img.shields.io/badge/-nginx-009639.svg?logo=nginx&style=for-the-badge&labelColor=000000" alt="nginx">
  </a>
  <a aria-label="Let's Encrypt" href="https://letsencrypt.org/">
    <img src="https://img.shields.io/badge/-letsencrypt-003A70.svg?logo=letsencrypt&style=for-the-badge&labelColor=000000" alt="Let's Encrypt">
  </a>
  <a aria-label="License" href="https://github.com/JamBalaya56562/nginx-proxy/blob/main/LICENSE">
    <img src="https://img.shields.io/github/license/JamBalaya56562/nginx-proxy?style=for-the-badge&labelColor=000000" alt="License">
  </a>
</p>

<p align="center">
  Set up a container running nginx and docker-gen with SSL certificates automatically:robot:
</p>

## 📄 Usage

### 1. Clone
To clone and run this application, you'll need [Git](https://git-scm.com) and [Docker Desktop](https://www.docker.com/products/docker-desktop/) installed on your computer.  
From your command line:

```bash
# Clone this repository
$ git clone https://github.com/JamBalaya56562/nginx-proxy
```

### 2. Set variables in the `.env` file
`DEFAULT_EMAIL` is used by [Let's Encrypt](https://letsencrypt.org/) in order to warn you about expiring certificates and allow you to recover your account.  
`NETWORK` is whatever you like.

```diff
# For example
- DEFAULT_EMAIL=
- NETWORK=
+ DEFAULT_EMAIL=example@gmail.com
+ NETWORK=nginx_web
```
### 3. Docker Compose
```bash
# Check if variables are appropriate
$ docker compose convert

# Run and up the containers using docker compose
$ docker compose up -d

# Make sure if there are errors in the containers
$ docker compose logs
```

## 📦 Credits

This software uses the following open source packages:

- [Docker](https://www.docker.com/)
- [Nginx](https://nginx.org/)
- [Let's Encrypt](https://letsencrypt.org/)
- [docker-gen](https://github.com/nginx-proxy/docker-gen)
- [nginx-proxy](https://github.com/nginx-proxy/nginx-proxy)
- [acme-companion](https://github.com/nginx-proxy/acme-companion)

## ⚖️ License

The MIT License Copyright (c) 2023 - [JamBalaya56562](https://github.com/JamBalaya56562).  
Please have a look at the [LICENSE](https://github.com/JamBalaya56562/nginx-proxy/blob/main/LICENSE) for more details.
