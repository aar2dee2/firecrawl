# Self-hosting Firecrawl

#### Contributor?

Welcome to [Firecrawl](https://firecrawl.dev) 🔥! Here are some instructions on how to get the project locally so you can run it on your own and contribute.

If you're contributing, note that the process is similar to other open-source repos, i.e., fork Firecrawl, make changes, run tests, PR.

If you have any questions or would like help getting on board, join our Discord community [here](https://discord.gg/gSmWdAkdwd) for more information or submit an issue on Github [here](https://github.com/mendableai/firecrawl/issues/new/choose)!

## Why?

Self-hosting Firecrawl is particularly beneficial for organizations with stringent security policies that require data to remain within controlled environments. Here are some key reasons to consider self-hosting:

- **Enhanced Security and Compliance:** By self-hosting, you ensure that all data handling and processing complies with internal and external regulations, keeping sensitive information within your secure infrastructure. Note that Firecrawl is a Mendable product and relies on SOC2 Type2 certification, which means that the platform adheres to high industry standards for managing data security.
- **Customizable Services:** Self-hosting allows you to tailor the services, such as the Playwright service, to meet specific needs or handle particular use cases that may not be supported by the standard cloud offering.
- **Learning and Community Contribution:** By setting up and maintaining your own instance, you gain a deeper understanding of how Firecrawl works, which can also lead to more meaningful contributions to the project.

### Considerations

However, there are some limitations and additional responsibilities to be aware of:

1. **Limited Access to Fire-engine:** Currently, self-hosted instances of Firecrawl do not have access to Fire-engine, which includes advanced features for handling IP blocks, robot detection mechanisms, and more. This means that while you can manage basic scraping tasks, more complex scenarios might require additional configuration or might not be supported.
2. **Manual Configuration Required:** If you need to use scraping methods beyond the basic fetch and Playwright options, you will need to manually configure these in the `.env` file. This requires a deeper understanding of the technologies and might involve more setup time.

Self-hosting Firecrawl is ideal for those who need full control over their scraping and data processing environments but comes with the trade-off of additional maintenance and configuration efforts.

## Steps

1. First, start by installing the dependencies

- Docker [instructions](https://docs.docker.com/get-docker/)

2. Set environment variables

Create an `.env` in the root directory you can copy over the template in `apps/api/.env.example`

To start, we won't set up authentication or any optional subservices (pdf parsing, JS blocking support, AI features)

`.env:`

```
# ===== Required ENVS ======
NUM_WORKERS_PER_QUEUE=8
PORT=3002
HOST=0.0.0.0
REDIS_URL=redis://redis:6379
REDIS_RATE_LIMIT_URL=redis://redis:6379

## To turn on DB authentication, you need to set up Supabase.
USE_DB_AUTHENTICATION=false

# ===== Optional ENVS ======

# Supabase Setup (used to support DB authentication, advanced logging, etc.)
SUPABASE_ANON_TOKEN=
SUPABASE_URL=
SUPABASE_SERVICE_TOKEN=

# Other Optionals
TEST_API_KEY= # use if you've set up authentication and want to test with a real API key
SCRAPING_BEE_API_KEY= # use if you'd like to use as a fallback scraper
OPENAI_API_KEY= # add for LLM-dependent features (e.g., image alt generation)
BULL_AUTH_KEY= @
LOGTAIL_KEY= # Use if you're configuring basic logging with logtail
PLAYWRIGHT_MICROSERVICE_URL=  # set if you'd like to run a playwright fallback
LLAMAPARSE_API_KEY= #Set if you have a llamaparse key you'd like to use to parse pdfs
SLACK_WEBHOOK_URL= # set if you'd like to send slack server health status messages
POSTHOG_API_KEY= # set if you'd like to send posthog events like job logs
POSTHOG_HOST= # set if you'd like to send posthog events like job logs
```

3.  _(Optional) Running with TypeScript Playwright Service_

    - Update the `docker-compose.yml` file to change the Playwright service:

      ```plaintext
          build: apps/playwright-service
      ```

      TO

      ```plaintext
          build: apps/playwright-service-ts
      ```

    - Set the `PLAYWRIGHT_MICROSERVICE_URL` in your `.env` file:

      ```plaintext
      PLAYWRIGHT_MICROSERVICE_URL=http://localhost:3000/scrape
      ```

    - Don't forget to set the proxy server in your `.env` file as needed.

4.  Build and run the Docker containers:

    ```bash
    docker compose build
    docker compose up
    ```

This will run a local instance of Firecrawl which can be accessed at `http://localhost:3002`.

You should be able to see the Bull Queue Manager UI on `http://localhost:3002/admin/@/queues`.

5.  _(Optional)_ Deploy to [Fly.io](https://fly.io/)

    - Install the [Fly CLI](https://fly.io/docs/reference/cli/)
    - You will need to deploy 2 services `/apps/api` and `/apps/playwright-service-ts` as separate apps.
    - Start by creating a Redis instance using [Fly](https://fly.io/docs/upstash/redis/). You will need to do this as an IPv6 address is needed to connect to Redis from Fly, but if you create the instance directly on Upstash, you will need the Pro plan to get a private addess.
    - Setup a Proxy Server. [froxy](https://froxy.com/).
    - Now deploy the `playwright-service-ts` app. Set `PORT = 3000` in the `fly.toml` file. Here's the fly config for reference (you must set `[[services.ports]]` or Fly will throw warnings like "is your app listening on port 3000?"):

      ```toml
        app = 'playwright-service'
        primary_region = 'sin'
        [build]
        [env]
        PORT = '3000'

        [http_service]
        internal_port = 3000
        force_https = true
        auto_stop_machines = 'stop'
        auto_start_machines = true
        min_machines_running = 0
        processes = ['app']

        [[services]]
        protocol = ''
        internal_port = 0

        [[services.ports]]
        port = 3000
        handlers = ['http']
        force_https = true

        [[vm]]
        memory = '1gb'
        cpu_kind = 'shared'
        cpus = 1
      ```

- Before deploying the `api` app, set the `PLAYWRIGHT_MICROSERVICE_URL` in the `.env` file to point to the `playwright-service-ts` app. Also, set the `REDIS_URL` and `REDIS_RATE_LIMIT_URL` in the `.env` file to point to the Redis instance you created earlier. You should use a [redis connection string starting with `rediss://`](https://github.com/redis/ioredis?tab=readme-ov-file#connect-to-redis) to connect to the TLS-enabled Redis instance.
- You may want to set up an [`http_check`](https://fly.io/docs/reference/configuration/#the-checks-section) on the `playwright-service-ts` app to ensure that only your Firecrawl instance makes requests to it.
- Deploy the `api` app. Here's the fly config for reference (you must set `[[services.ports]]` or Fly will throw warnings like "is your app listening on port 8080?"):

  ```toml
  primary_region = 'sin'
  kill_signal = 'SIGINT'
  kill_timeout = '30s'

  [build]

  [env]
  HOST = '0.0.0.0'
  PORT = '8080'

  [processes]
  app = 'node --max-old-space-size=8192 dist/src/index.js'
  worker = 'node --max-old-space-size=8192 dist/src/services/queue-worker.js'

  [http_service]
  internal_port = 8080
  force_https = true
  auto_stop_machines = 'off'
  auto_start_machines = true
  min_machines_running = 2
  processes = ['app']

  [http_service.concurrency]
  type = 'requests'
  soft_limit = 200

  [[http_service.checks]]
  interval = '30s'
  timeout = '15s'
  grace_period = '20s'
  method = 'GET'
  path = '/'

  [[services]]
  protocol = 'tcp'
  internal_port = 8080
  processes = ['app']

  [[services.ports]]
  port = 8080
  handlers = ['http']
  force_https = true

  [[services.ports]]
  port = 443
  handlers = ['tls', 'http']

  [services.concurrency]
  type = 'connections'
  soft_limit = 200

  [[vm]]
  size = 'performance-1x'
  processes = ['app']
  ```

  After deploying, set env variables using `fly secrets set`. The Redis connection string does not work with `ioredis`, for some reason. After many tries, I used the object format as describe in this Fly community thread.

  ```ts
  export const redis = new Redis({
    host: process.env.REDIS_DOMAIN,
    password: process.env.REDIS_PASSWORD,
    port: 6379,
    username: "default",
    family: 6,
    db: 0,
  });
  ```

  If your redis instance has TLS enabled, you will also need to set the `tls` options in the `Redis` object.
  For the REDIS_DOMAIN, you can get the ipv6 address of the redis instance by running `dig +short AAAA <redis instance host>`.
  (IPv6 addresses are available only for the Redis instances created through Fly. For the ones created through Upstash, you will need the Pro plan to get a private address.)

6. _(Optional)_ Test the API

If you’d like to test the crawl endpoint, you can run this:

```bash
curl -X POST http://localhost:3002/v1/crawl \
  -H 'Content-Type: application/json' \
  -d '{
    "url": "https://mendable.ai"
  }'
```

## Troubleshooting

This section provides solutions to common issues you might encounter while setting up or running your self-hosted instance of Firecrawl.

### Supabase client is not configured

**Symptom:**

```bash
[YYYY-MM-DDTHH:MM:SS.SSSz]ERROR - Attempted to access Supabase client when it's not configured.
[YYYY-MM-DDTHH:MM:SS.SSSz]ERROR - Error inserting scrape event: Error: Supabase client is not configured.
```

**Explanation:**
This error occurs because the Supabase client setup is not completed. You should be able to scrape and crawl with no problems. Right now it's not possible to configure Supabase in self-hosted instances.

### You're bypassing authentication

**Symptom:**

```bash
[YYYY-MM-DDTHH:MM:SS.SSSz]WARN - You're bypassing authentication
```

**Explanation:**
This error occurs because the Supabase client setup is not completed. You should be able to scrape and crawl with no problems. Right now it's not possible to configure Supabase in self-hosted instances.

### Docker containers fail to start

**Symptom:**
Docker containers exit unexpectedly or fail to start.

**Solution:**
Check the Docker logs for any error messages using the command:

```bash
docker logs [container_name]
```

- Ensure all required environment variables are set correctly in the .env file.
- Verify that all Docker services defined in docker-compose.yml are correctly configured and the necessary images are available.

### Connection issues with Redis

**Symptom:**
Errors related to connecting to Redis, such as timeouts or "Connection refused".

**Solution:**

- Ensure that the Redis service is up and running in your Docker environment.
- Verify that the REDIS_URL and REDIS_RATE_LIMIT_URL in your .env file point to the correct Redis instance, ensure that it points to the same URL in the `docker-compose.yaml` file (`redis://redis:6379`)
- Check network settings and firewall rules that may block the connection to the Redis port.

### API endpoint does not respond

**Symptom:**
API requests to the Firecrawl instance timeout or return no response.

**Solution:**

- Ensure that the Firecrawl service is running by checking the Docker container status.
- Verify that the PORT and HOST settings in your .env file are correct and that no other service is using the same port.
- Check the network configuration to ensure that the host is accessible from the client making the API request.

By addressing these common issues, you can ensure a smoother setup and operation of your self-hosted Firecrawl instance.

## Install Firecrawl on a Kubernetes Cluster (Simple Version)

Read the [examples/kubernetes/cluster-install/README.md](https://github.com/mendableai/firecrawl/blob/main/examples/kubernetes/cluster-install/README.md) for instructions on how to install Firecrawl on a Kubernetes Cluster.
