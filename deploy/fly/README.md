# Dgraph on fly.io — Deployment Guide

This guide walks you through deploying a single-node Dgraph cluster on [Fly.io](https://fly.io) using the provided `fly.toml` configuration.

Fly.io is a platform for building and deploying cloud-native applications. It provides a simple and easy way to deploy and manage your applications.
The configuration, as provided in `fly.toml`, can be deployed for as little as $10 per month.

## Prerequisites

### 1. Create a Fly.io Account

1. Visit [fly.io](https://fly.io) and sign up for an account (if you don't have one already)
2. Install the Fly CLI by following the [installation guide](https://fly.io/docs/hands-on/install-flyctl/)
3. Authenticate with Fly.io:
   ```bash
   fly auth login
   ```

### 2. Verify Installation

Check that fly is properly installed:
```bash
fly version
```

## Configuration

### 1. Update the App Name

Edit the `fly.toml` file and change the app name:
```toml
app = "your-dgraph-cluster-name"
```

**Important:** The app name must be globally unique across all Fly.io applications. Your Dgraph instance will be accessible at `https://your-dgraph-cluster-name.fly.dev`.

### 2. Security Configuration

**⚠️ Critical:** Change the security token in the `DGRAPH_ALPHA_SECURITY` environment variable:
```toml
DGRAPH_ALPHA_SECURITY = "token=your-secure-random-token;whitelist=0.0.0.0/0"
```

Replace `your-secure-random-token` with a strong, randomly generated token. This token will be required in the `X-Dgraph-AuthToken` header for any requests to the `/alter` endpoint.

#### Adjust Volume Settings
Modify the volume configuration based on your needs:
- `initial_size`: Starting volume size (default: 10GB)
- `auto_extend_size_threshold`: Percentage threshold for auto-extension (default: 80%)
- `auto_extend_size_increment`: Size to add when extending (default: 2GB)
- `auto_extend_size_limit`: Maximum volume size (default: 20GB)

## Deployment

### 1. Launch the Application

For the initial deployment, run `fly launch`. This command will create the application on Fly.io, provision the persistent volume, and deploy the Dgraph instance.

From the directory containing `fly.toml`:
```bash
fly launch --config fly.toml --copy-config
```

You will be prompted to confirm settings. You can skip allocating an IP address as we will do that later. For subsequent deployments, you can use `fly deploy`.

### 2. Monitor Deployment

Check the deployment status:
```bash
fly status
```

View logs:
```bash
fly logs
```

### 3. Assign IP Address

```bash
fly ips allocate-v4 --shared -a your-dgraph-cluster-name
```

Note, you can omit the `--shared` flag if you want to use a private IP address.

### 4. Verify Health

Check if your Dgraph instance is healthy.

```bash
curl https://your-dgraph-cluster-name.fly.dev/health
```

## Accessing Your Dgraph Instance

### DQL Query Endpoint
- **URL:** `https://your-dgraph-cluster-name.fly.dev/query`
- **Port:** 443 (HTTPS)
- **Protocols:** DQL queries

### DQL Mutation Endpoint
- **URL:** `https://your-dgraph-cluster-name.fly.dev/mutate`
- **Port:** 443 (HTTPS)
- **Protocols:** DQL mutations

### GraphQL Endpoint
- **URL:** `https://your-dgraph-cluster-name.fly.dev/graphql`
- **Port:** 443 (HTTPS)
- **Protocols:** GraphQL requests

### gRPC Endpoint
- **URL:** `your-dgraph-cluster-name.fly.dev:9080`
- **Port:** 9080
- **Protocol:** gRPC over TLS

Here's an example go program that illustrates how to use the gRPC endpoint with fly.io:

```go
   import (
        "crypto/tls"
        "log"
        
        "github.com/dgraph-io/dgo/v240"
        "github.com/dgraph-io/dgo/v240/protos/api"
        "google.golang.org/grpc"
        "google.golang.org/grpc/credentials"      
   )

   func main() {
      creds := credentials.NewTLS(&tls.Config{ServerName: "your-dgraph-cluster-name.fly.dev"})
      conn, err := grpc.Dial("your-dgraph-cluster-name.fly.dev:9080", grpc.WithTransportCredentials(creds))
      if err != nil {
         log.Fatal("Failed to connect to Dgraph:", err)
      }
      defer conn.Close()

      // Create a Dgraph client
      dgraphClient := dgo.NewDgraphClient(api.NewDgraphClient(conn))
      // Use the gRPC client
      ///...
   }
```

### Authentication to /alter endpoint

For admin operations (schema changes, mutations to reserved predicates), include the auth token:
```bash
curl -H "X-Dgraph-AuthToken: your-secure-random-token" \
     -X POST \
     https://your-dgraph-cluster-name.fly.dev/alter \
     -d '{"drop_all": true}'
```

## Management Commands

### View Volume Information
```bash
fly volumes list
```

### SSH into the Instance
```bash
fly ssh console
```

## Deleting the Instance

**Warning:** The following steps will permanently delete your Dgraph application and all associated data. This action cannot be undone.

### 1. Destroy the Application

This command will remove the application and the attached persistent volume.

```bash
fly apps destroy your-dgraph-cluster-name
```

Answer `y` to the confirmation prompts.

## Troubleshooting

### Common Issues

1. **App name already taken**
   - Change the `app` name in `fly.toml` to something unique

2. **Health check failures**
   - Check logs with `fly logs`
   - Verify the Dgraph process is running: `fly ssh console` then `ps aux | grep dgraph`

3. **Connection issues**
   - Ensure your app is deployed: `flyctl status`
   - Check if the health endpoint responds: `curl https://your-dgraph-cluster-name.fly.dev/health`

4. **Volume issues**
   - List volumes: `flyctl volumes list`
   - Check volume usage: `flyctl ssh console` then `df -h /dgraph`

### Useful Commands

```bash
# Restart the application
flyctl apps restart your-dgraph-cluster-name

# View machine details
flyctl machine list

# Check resource usage
flyctl machine status

# Destroy the application (careful!)
flyctl apps destroy your-dgraph-cluster-name
```

## Architecture Details

This configuration deploys:
- **Dgraph Zero**: Cluster management (internal)
- **Dgraph Alpha**: Data storage and query processing
- **Persistent Volume**: 10GB initial size with auto-extension
- **TLS Termination**: Automatic HTTPS via Fly.io
- **Health Checks**: HTTP health checks every 10 seconds

## Security Considerations

1. **Change the default token** in `DGRAPH_ALPHA_SECURITY`
2. **Whitelist configuration**: Currently set to `0.0.0.0/0` (all IPs). Consider restricting to specific IP ranges for production
3. **TLS**: All connections are automatically encrypted via Fly.io's TLS termination
4. **Network isolation**: The Dgraph instance runs in Fly.io's private network

## Cost Considerations

- **Compute**: Fly.io charges based on machine usage
- **Storage**: Volume storage is billed separately
- **Bandwidth**: Outbound data transfer charges may apply
- **Always-on**: `auto_stop_machines = false` keeps the instance running continuously

For current pricing, visit [Fly.io Pricing](https://fly.io/docs/about/pricing/).

## Next Steps

1. **Load your schema**: Use the `/alter` endpoint with your auth token
2. **Import data**: Use mutations or bulk loading
3. **Set up monitoring**: Consider integrating with Fly.io's monitoring tools
4. **Backup strategy**: Implement regular backups of your volume data

## Resources

- [Fly.io Documentation](https://fly.io/docs/)
- [Dgraph Documentation](https://dgraph.io/docs/)
- [Dgraph Docker Images](https://hub.docker.com/r/dgraph/standalone)
- [Fly.io Community Forum](https://community.fly.io/)