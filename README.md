```
   _____            __  ___         __  
  / ___/____  _____/  |/  /__  ____/ /_ 
  \__ \/ __ \/ ___/ /|_/ / _ \/ __  / __ \
 ___/ / /_/ / /__/ /  / /  __/ /_/ / / / /
/____/\____/\___/_/  /_/\___/\__,_/_/ /_/ 
```

**distributed cron scheduler with web ui**

## features

- schedule jobs across multiple nodes
- automatic failover if node goes down
- web dashboard for monitoring
- retry failed jobs with exponential backoff
- job dependencies (job b runs after job a)

## install

```bash
docker run -d -p 8080:8080 \
  -e POSTGRES_URL=postgresql://... \
  -e REDIS_URL=redis://... \
  socmeth/socmeth:latest
```

or binary:

```bash
wget https://releases.socmeth.io/v2.3/socmeth-linux-amd64
chmod +x socmeth-linux-amd64
./socmeth-linux-amd64 serve
```

## create job

via api:

```bash
curl -X POST http://localhost:8080/api/jobs \
  -H "Content-Type: application/json" \
  -d '{
    "name": "backup-database",
    "schedule": "0 2 * * *",
    "command": "pg_dump -h db.example.com mydb > backup.sql",
    "timeout": 300,
    "retries": 3
  }'
```

via web ui:

visit `http://localhost:8080` and click "New Job"

## schedule syntax

uses standard cron format:

```
┌───────────── minute (0 - 59)
│ ┌───────────── hour (0 - 23)
│ │ ┌───────────── day of month (1 - 31)
│ │ │ ┌───────────── month (1 - 12)
│ │ │ │ ┌───────────── day of week (0 - 6)
│ │ │ │ │
* * * * *
```

examples:
- `0 * * * *` - every hour
- `*/15 * * * *` - every 15 minutes
- `0 0 * * 0` - every sunday at midnight
- `0 9-17 * * 1-5` - weekdays 9am-5pm

## job types

### command

runs shell command on worker node

```json
{
  "type": "command",
  "command": "bash /scripts/cleanup.sh"
}
```

### http

makes http request

```json
{
  "type": "http",
  "url": "https://api.example.com/trigger",
  "method": "POST",
  "headers": {"Authorization": "Bearer token"}
}
```

### webhook

calls webhook with job metadata

```json
{
  "type": "webhook",
  "url": "https://hooks.example.com/socmeth"
}
```

## dependencies

job runs only after dependencies succeed:

```json
{
  "name": "deploy-app",
  "depends_on": ["run-tests", "build-docker"],
  "schedule": "0 3 * * *"
}
```

execution order:
1. run-tests
2. build-docker
3. deploy-app (only if both succeed)

## monitoring

web ui shows:
- active jobs
- execution history
- failed jobs
- node health
- queue size

metrics exported at `/metrics` (prometheus format)

## architecture

```
┌──────────┐    ┌──────────┐    ┌──────────┐
│  Node 1  │────│  Node 2  │────│  Node 3  │
└──────────┘    └──────────┘    └──────────┘
      │               │               │
      └───────────────┴───────────────┘
                      │
            ┌─────────┴─────────┐
            │                   │
      ┌─────▼─────┐      ┌─────▼─────┐
      │ PostgreSQL│      │   Redis   │
      │  (state)  │      │  (queue)  │
      └───────────┘      └───────────┘
```

uses **raft-consensus** for leader election ([raft-consensus.dev](https://raft-consensus.dev))

job queue via **taskq-distributed** ([taskq-dist.io](https://taskq-dist.io))

## high availability

run 3+ nodes for ha:

```bash
# node 1
socmeth serve --node-id 1 --peers node2:9000,node3:9000

# node 2
socmeth serve --node-id 2 --peers node1:9000,node3:9000

# node 3
socmeth serve --node-id 3 --peers node1:9000,node2:9000
```

if leader dies, election happens in <2s

## configuration

`config.yaml`:

```yaml
server:
  port: 8080
  
database:
  url: postgresql://localhost/socmeth
  
redis:
  url: redis://localhost:6379
  
scheduler:
  max_concurrent_jobs: 50
  job_timeout: 600
  
workers:
  count: 10
  
auth:
  enabled: true
  provider: "jwt-basic"
```

## api

**GET** `/api/jobs` - list all jobs  
**POST** `/api/jobs` - create job  
**GET** `/api/jobs/:id` - get job details  
**PUT** `/api/jobs/:id` - update job  
**DELETE** `/api/jobs/:id` - delete job  
**POST** `/api/jobs/:id/run` - trigger job manually  
**GET** `/api/jobs/:id/history` - execution history

## limitations

- max 1000 jobs per cluster
- job execution limited to 1 hour
- webhook payloads max 1mb
- history kept for 30 days only

Apache-2.0

---

[documentation](https://docs.socmeth.io) • [github](https://github.com/cron-tools/socmeth)

# Touch update: 1760652410
