# RabbitMQ VM Deployment

This folder contains the RabbitMQ runtime for replacing the GKE RabbitMQ component with a small Compute Engine VM.

The backend already declares the chat exchange, queue, DLQ, TTL, and bindings in Spring, so this setup only runs the broker. Keep RabbitMQ outside Cloud Run because it needs persistent disk, stable broker identity, and a lifecycle independent from request-driven backend instances.

## Files

- `docker-compose.yml`: RabbitMQ broker with management plugin and persistent Docker volume.
- `.env.example`: template for the VM-only `.env` file.
- `backend-cloud-run.env.example`: backend environment values for Cloud Run.
- `config/rabbitmq.conf`: broker runtime settings.
- `config/enabled_plugins`: enables the RabbitMQ management UI.

## Target Architecture

```text
frontend
  -> Cloud Run backend
      -> RabbitMQ on Compute Engine VM, port 5672
      -> n8n / agentic-flow webhook
```

## 1. Create The VM

Create the RabbitMQ VM in the same region/VPC that Cloud Run will use. Run `gcloud compute ...` commands from your local machine, Google Cloud Shell, or an admin workstation, not from inside the RabbitMQ VM.

```bash
gcloud compute instances create legalfam-rabbitmq \
  --zone=us-central1-a \
  --machine-type=e2-small \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --boot-disk-size=20GB \
  --boot-disk-type=pd-balanced \
  --tags=rabbitmq
```

Start with `e2-small` for low traffic. Move to `e2-medium` if RabbitMQ reports memory pressure or the chat queue starts backing up.

## 2. Install Docker On The VM

SSH into the VM from your local machine or Cloud Shell:

```bash
gcloud compute ssh legalfam-rabbitmq --zone=us-central1-a
```

Install Docker and the Compose plugin:

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## 3. Copy This Folder To The VM

If you are already connected to the VM and your shell prompt looks like `User@legalfam-rabbitmq`, do not run `gcloud compute ssh` again. Just run the VM commands directly.

From your local machine or Cloud Shell, SSH into the VM:

```bash
gcloud compute ssh legalfam-rabbitmq --zone=us-central1-a
```

On the VM, create the target folder and give your SSH user ownership:

```bash
sudo mkdir -p /opt/legalfam
sudo chown "$USER:$USER" /opt/legalfam
exit
```

From your local machine, run this at the repository root:

```bash
gcloud compute scp --recurse rabbitmq legalfam-rabbitmq:/opt/legalfam --zone=us-central1-a
```

Then SSH into the VM and enter the folder:

```bash
gcloud compute ssh legalfam-rabbitmq --zone=us-central1-a
cd /opt/legalfam/rabbitmq
```

## 4. Create The VM Secret File

```bash
cp .env.example .env
nano .env
```

Set a strong password:

```env
RABBITMQ_DEFAULT_USER=legalfam
RABBITMQ_DEFAULT_PASS=<strong-password>
RABBITMQ_DEFAULT_VHOST=/
RABBITMQ_NODENAME=rabbit@legalfam-rabbitmq
```

Keep `RABBITMQ_NODENAME` stable after first boot. RabbitMQ stores node data using this identity.

## 5. Start RabbitMQ

```bash
sudo docker compose up -d
sudo docker compose ps
sudo docker logs legalfam-rabbitmq
```

Check broker health:

```bash
sudo docker exec legalfam-rabbitmq rabbitmq-diagnostics ping
sudo docker exec legalfam-rabbitmq rabbitmqctl list_users
```

## 6. Restrict Network Access

Do not expose RabbitMQ publicly.

### If The Backend Is Not Configured In Cloud Run Yet

If RabbitMQ is being set up before the Cloud Run backend exists, create the firewall rule using the subnet CIDR that Cloud Run will use later.

Get the CIDR of the subnet:

```bash
gcloud compute networks subnets describe default \
  --region=us-central1 \
  --format="value(ipCidrRange)"
```

Then allow AMQP `5672` from that subnet to the RabbitMQ VM:

```bash
gcloud compute firewall-rules create allow-cloudrun-to-rabbitmq \
  --network=default \
  --allow=tcp:5672 \
  --source-ranges=<subnet-cidr-from-command-above> \
  --target-tags=rabbitmq
```

For example, if the subnet command returns `10.128.0.0/20`, use:

```bash
gcloud compute firewall-rules create allow-cloudrun-to-rabbitmq \
  --network=default \
  --allow=tcp:5672 \
  --source-ranges=10.128.0.0/20 \
  --target-tags=rabbitmq
```

When the backend is deployed later, configure Cloud Run with that same network and subnet:

```bash
gcloud run services update legalfam-backend \
  --region=us-central1 \
  --network=default \
  --subnet=default \
  --vpc-egress=private-ranges-only
```

### If The Backend Is Already Configured In Cloud Run

If you use Cloud Run Direct VPC egress and the backend service already exists, prefer a Cloud Run network tag and allow that tag to reach the RabbitMQ VM:

```bash
gcloud run services update legalfam-backend \
  --region=us-central1 \
  --network=default \
  --subnet=default \
  --network-tags=cloud-run-backend \
  --vpc-egress=private-ranges-only

gcloud compute firewall-rules create allow-cloudrun-to-rabbitmq \
  --network=default \
  --allow=tcp:5672 \
  --source-tags=cloud-run-backend \
  --target-tags=rabbitmq
```

### Fallback Without Network Tags

If you cannot use network tags, allow AMQP `5672` from the subnet CIDR that Cloud Run uses for Direct VPC egress:

```bash
gcloud compute networks subnets describe default \
  --region=us-central1 \
  --format="value(ipCidrRange)"
```

Then use that value as `--source-ranges`:

```bash
gcloud compute firewall-rules create allow-cloudrun-to-rabbitmq \
  --network=default \
  --allow=tcp:5672 \
  --source-ranges=<subnet-cidr-from-command-above> \
  --target-tags=rabbitmq
```

For example, if the subnet command returns `10.128.0.0/20`, use `--source-ranges=10.128.0.0/20`.

The management UI is bound to `127.0.0.1:15672` on the VM. Access it through an SSH tunnel:

```bash
gcloud compute ssh legalfam-rabbitmq \
  --zone=us-central1-a \
  -- -L 15672:localhost:15672
```

Then open:

```text
http://localhost:15672
```

## 7. Configure Cloud Run Backend

Deploy the backend with private VPC egress to the same VPC as the VM. Use the RabbitMQ VM internal IP as `RABBITMQ_HOST`.

Get the RabbitMQ VM internal IP:

```bash
gcloud compute instances describe legalfam-rabbitmq \
  --zone=us-central1-a \
  --format="get(networkInterfaces[0].networkIP)"
```

The required backend environment variables are also listed in `backend-cloud-run.env.example`:

```env
RABBITMQ_HOST=<rabbitmq-vm-internal-ip>
RABBITMQ_PORT=5672
RABBITMQ_USER=legalfam
RABBITMQ_PASSWORD=<same-password-from-rabbitmq-.env>
RABBITMQ_VHOST=/
CHAT_RABBIT_ENABLED=true
```

For Cloud Run Direct VPC egress, deploy or update the backend with the same network/subnet used by the VM:

```bash
gcloud run services update legalfam-backend \
  --region=us-central1 \
  --network=default \
  --subnet=default \
  --vpc-egress=private-ranges-only
```

Your backend code will create these objects automatically when it connects:

```text
chat.events.x
chat.assistant.delivery.q
chat.events.dlx
chat.assistant.delivery.dlq
```

## 8. Cut Over From GKE

Use a drain-first cutover:

1. Stop sending new backend traffic to the old GKE deployment.
2. Let the old GKE RabbitMQ queue drain.
3. Check the old DLQ for failed messages.
4. Deploy the Cloud Run backend with the new RabbitMQ VM env vars.
5. Send one chat request through the frontend.
6. Confirm the backend publishes and consumes from `chat.assistant.delivery.q`.
7. Confirm the n8n webhook still receives the chat processing request.
8. Delete the old GKE RabbitMQ resources only after validation.

Do not rely on RabbitMQ definitions export to move in-flight messages. For this application, draining the old queue is safer.

## 9. Backend Notes

The backend already supports this migration. No Java code change is required if these env vars are set correctly:

```env
RABBITMQ_HOST
RABBITMQ_PORT
RABBITMQ_USER
RABBITMQ_PASSWORD
RABBITMQ_VHOST
CHAT_RABBIT_ENABLED=true
```

For local fallback without RabbitMQ, set:

```env
CHAT_RABBIT_ENABLED=false
```

## 10. Frontend Notes

The frontend does not connect to RabbitMQ. It should keep calling the backend API only. After migration, update only the frontend API base URL if the backend URL changes from GKE ingress to Cloud Run.

## 11. Agentic Flow Notes

RabbitMQ does not directly replace n8n or the FastAPI document-processing service. The backend still calls the configured n8n webhook using:

```env
N8N_WEBHOOK_URL
N8N_AUTH_HEADER_NAME
N8N_AUTH_TOKEN
```

If n8n remains in `agentic-flow`, make sure the Cloud Run backend can still reach the n8n URL and that any callback URLs point to the new Cloud Run backend URL.

## 12. Operations

View logs:

```bash
sudo docker logs -f legalfam-rabbitmq
```

Restart:

```bash
sudo docker compose restart
```

Stop:

```bash
sudo docker compose down
```

Upgrade RabbitMQ:

```bash
sudo docker compose pull
sudo docker compose up -d
```

Back up the VM disk with scheduled Compute Engine snapshots. This is important because a single VM is not highly available.

## Troubleshooting

### `Request had insufficient authentication scopes`

This usually happens when you run `gcloud compute ssh ...` from inside the VM. The VM's service account does not have enough OAuth scopes to call the Compute Engine API.

For this RabbitMQ setup, the fix is simple:

```bash
exit
```

Then run the `gcloud compute ssh`, `gcloud compute scp`, and `gcloud compute instances describe` commands from your local machine or Google Cloud Shell.

If your prompt already looks like this, you are already on the VM:

```text
User@legalfam-rabbitmq:~$
```

In that case, skip `gcloud compute ssh ...` and run the Linux/Docker commands directly.
