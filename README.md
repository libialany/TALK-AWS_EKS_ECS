# Friendly server

Welcome to the workshop! In this session, we will guide you through setting up a development environment using Docker Compose to manage `dnsmasq`, `nginx`, and your application.

## Prerequisites

Before you begin, ensure you have the following installed on your machine:

- [Docker](https://docs.docker.com/get-docker/)
- [Docker Compose](https://docs.docker.com/compose/install/)

## Overview

In this workshop, we will:

1. Set up `dnsmasq` for DNS management.
2. Configure `nginx` as a reverse proxy.
3. Deploy your application using Docker Compose.

## Getting Started

### Step 1: Clone the Repository

First, clone the repository containing the Docker Compose configuration:

`git clone <> && cd lab`

### Step 2: Review the Docker Compose File

Open the `docker-compose.yml` file to understand the services being set up.


### Step 3: Configure `nginx`

Edit the `nginx.conf` file to set up your reverse proxy settings. This file is mounted into the `nginx` container.

### Step 4: Build and Run the Containers

Use Docker Compose to build and start the services:

`docker-compose up --build`

### Step 5: Access Your Application

Once the services are running, you can access your application at `http://localhost:8080`.

## Troubleshooting

- Ensure Docker and Docker Compose are installed and running.
- Check the logs for each service using `docker-compose logs <service_name>`.

## Conclusion

By the end of this workshop, you should have a working setup of `dnsmasq`, `nginx`, and your application using Docker Compose. Feel free to experiment with the configurations to suit your needs.

## Resources

- [Docker Documentation](https://docs.docker.com/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)

Thank you for participating in the workshop!