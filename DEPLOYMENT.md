# Hyperliquid Signal System - Cloudflare Deployment Guide

This guide provides comprehensive instructions on how to deploy the Hyperliquid Signal System to Cloudflare using GitHub linking for continuous integration and deployment (CI/CD).

The project consists of two main parts:
1.  **Frontend (Admin Panel):** A React application designed to be deployed on Cloudflare Pages.
2.  **Backend (Workers):** Multiple Cloudflare Workers handling various functionalities, each deployed as a separate Worker.

## Prerequisites

Before you begin, ensure you have the following:

*   A Cloudflare account.
*   A GitHub account.
*   `wrangler` CLI installed locally (optional, but useful for local development and testing).

## Deployment Steps

### 1. Fork the Repository

First, fork the Hyperliquid Signal System repository to your GitHub account. This will allow you to link your repository to Cloudflare for automatic deployments.

### 2. Cloudflare D1 Database Setup

The backend Workers utilize Cloudflare D1 for database persistence. You will need to create a D1 database in your Cloudflare account.

1.  Log in to your Cloudflare dashboard.
2.  Navigate to the Workers & Pages section.
3.  Go to the 'D1' tab and click 'Create database'.
4.  Give your database a meaningful name (e.g., `hyperliquid_signals`).
5.  Note down the `Database ID` for this database. You will need it for configuring your Workers.

### 3. Cloudflare Queues Setup

Some backend Workers use the Cloudflare Queues for inter-service communication. You will need to create a queue in your Cloudflare account.

1.  Log in to your Cloudflare dashboard.
2.  Navigate to the Workers & Pages section.
3.  Go to the 'Queues' tab and click 'Create queue'.
4.  Give your queue a meaningful name (e.g., `signal-events`).

### 4. Deploying the Frontend (Cloudflare Pages)

The frontend is a React application that can be easily deployed using Cloudflare Pages.

1.  Log in to your Cloudflare dashboard.
2.  Navigate to the Workers & Pages section.
3.  Click 'Create application' and then 'Connect to Git'.
4.  Select your forked Hyperliquid Signal System repository.
5.  For the 'Build and deployment configuration':
    *   **Framework preset:** `Vite` (Cloudflare should auto-detect this).
    *   **Build command:** `npm run build` or `pnpm run build` (depending on your package manager).
    *   **Build directory:** `frontend/admin-panel/dist`.
6.  Click 'Deploy site'.

Cloudflare Pages will automatically build and deploy your frontend whenever you push changes to your GitHub repository.

### 5. Deploying the Backend (Cloudflare Workers)

Each backend Worker needs to be deployed individually. The project contains several Workers (e.g., `ingestion`, `notifier`, `performance-tracker`, `price-monitor`, `signal-processor`). Each Worker has its own `wrangler.toml` file.

For each Worker (e.g., `ingestion`):

1.  **Update `wrangler.toml`:**
    Navigate to the Worker's directory (e.g., `backend/workers/ingestion/`). Open the `wrangler.toml` file.
    You will find placeholders like `database_id = "<YOUR_D1_DATABASE_ID>"` and `queue = "<YOUR_QUEUE_NAME>"`.
    Replace `<YOUR_D1_DATABASE_ID>` with the actual Database ID you noted from Step 2.
    Replace `<YOUR_QUEUE_NAME>` with the actual Queue name you noted from Step 3.

    Example `wrangler.toml` (after updates):
    ```toml
    name = "ingestion-worker"
    main = "src/index.ts"
    compatibility_date = "2024-01-01"
    [vars]
    ENVIRONMENT = "production"
    [[d1_databases]]
    binding = "DB"
    database_name = "hyperliquid_signals"
    database_id = "YOUR_ACTUAL_D1_DATABASE_ID_HERE"
    [[queues.producers]]
    binding = "SIGNAL_QUEUE"
    queue = "signal-events"
    ```

2.  **Deploy the Worker:**
    You can deploy each Worker either via the Cloudflare dashboard or using `wrangler` CLI.

    *   **Option A: Cloudflare Dashboard (Recommended for initial setup)**
        1.  Log in to your Cloudflare dashboard.
        2.  Navigate to the Workers & Pages section.
        3.  Click 'Create application' and then 'Create Worker'.
        4.  Give your Worker a name (e.g., `ingestion-worker`).
        5.  Select 'Deploy from Git' and connect to your forked repository.
        6.  Specify the root directory for this Worker (e.g., `backend/workers/ingestion`).
        7.  Cloudflare will detect the `wrangler.toml` file and use its configuration.
        8.  Ensure that the D1 database and Queue bindings are correctly configured under the 'Settings' tab of your Worker in the Cloudflare dashboard. You might need to manually add them if they are not automatically picked up.

    *   **Option B: Using `wrangler` CLI (for advanced users/CI/CD pipelines)**
        1.  Ensure you have `wrangler` CLI installed and authenticated (`wrangler login`).
        2.  Navigate to the Worker's directory in your local clone of the repository (e.g., `cd backend/workers/ingestion`).
        3.  Run `wrangler deploy`.

Repeat Step 5 for each backend Worker in the `backend/workers/` directory.

## Post-Deployment

After deploying both the frontend and all backend Workers, your Hyperliquid Signal System should be fully operational on Cloudflare. You can access your admin panel via the Cloudflare Pages URL and monitor your Workers' logs in the Cloudflare dashboard.

Remember to set the `ENVIRONMENT` variable in each `wrangler.toml` to `"production"` for your deployed Workers.

## Troubleshooting

If you encounter any issues during deployment, refer to the Cloudflare documentation for Workers and Pages, or check the project's `README.md` for additional guidance.




## Additional Configuration and Setup

### Environment Variables

For each worker, you might need to set additional environment variables beyond what's in `wrangler.toml`. These can be configured in the Cloudflare dashboard under the Worker's settings, or via the `wrangler.toml` file itself under the `[vars]` section.

Common environment variables you might need to configure:

*   `TELEGRAM_BOT_TOKEN`: Your Telegram Bot API token for sending notifications.
*   `TELEGRAM_CHAT_ID`: The chat ID where Telegram notifications will be sent.
*   `HYPERLIQUID_INFO_API`: The API endpoint for Hyperliquid information (e.g., `https://api.hyperliquid.xyz/info`).
*   `KUCOIN_API`: The API endpoint for KuCoin (e.g., `https://api.kucoin.com`).

### Database Migrations

After creating your D1 database, you will need to apply the necessary schema migrations. The project includes SQL migration files in `backend/db/migrations/`.

You can apply these migrations using the `wrangler d1 execute` command. For example:

```bash
wrangler d1 execute <YOUR_D1_DATABASE_NAME> --file=backend/db/migrations/0001_initial_schema.sql
wrangler d1 execute <YOUR_D1_DATABASE_NAME> --file=backend/db/migrations/0002_performance_tables.sql
wrangler d1 execute <YOUR_D1_DATABASE_NAME> --file=backend/db/migrations/0003_notification_log.sql
wrangler d1 execute <YOUR_D1_DATABASE_NAME> --file=backend/db/migrations/0004_schema_fixes.sql
```

Replace `<YOUR_D1_DATABASE_NAME>` with the name of your D1 database (e.g., `hyperliquid_signals`).

### Cron Triggers

Some Workers might utilize Cron Triggers for scheduled execution (e.g., price monitoring, performance tracking). These are typically defined within the `wrangler.toml` file for the respective Worker. Cloudflare will automatically set up these triggers upon deployment.

You can verify the cron triggers in your Cloudflare dashboard under the Worker's settings or by using the `wrangler cron list` command.

## Continuous Integration and Deployment (CI/CD)

By linking your GitHub repository to Cloudflare Pages and Workers, you enable CI/CD. Any push to your main branch (or a configured branch) will automatically trigger a new build and deployment.

### GitHub Actions (Optional)

For more complex CI/CD workflows, you might consider setting up GitHub Actions to automate tasks like:

*   Running tests before deployment.
*   Linting code.
*   Building specific assets.
*   Deploying Workers via `wrangler deploy` in a CI environment.

## Troubleshooting

If you encounter issues during deployment or operation, consider the following:

*   **Check Cloudflare Logs:** The Cloudflare dashboard provides detailed logs for both Pages and Workers, which can help diagnose issues.
*   **`wrangler` CLI:** Use `wrangler tail <WORKER_NAME>` to stream logs from a Worker in real-time.
*   **Environment Variables:** Double-check that all required environment variables are correctly set for each Worker.
*   **Database Bindings:** Ensure your D1 database and Queue bindings are correctly configured in the `wrangler.toml` files and in the Cloudflare dashboard.
*   **Build Errors:** If your Pages deployment fails, review the build logs in the Cloudflare dashboard for errors during the build process.

## Next Steps After Deployment

Once your Hyperliquid Signal System is deployed:

1.  **Access the Admin Panel:** Navigate to the URL provided by Cloudflare Pages for your frontend.
2.  **Configure Wallets:** Use the admin panel to add and manage wallet addresses you wish to monitor.
3.  **Monitor Performance:** Observe the system's performance and signal generation through the admin panel and Cloudflare logs.
4.  **Adjust Settings:** Modify any configurable parameters (e.g., signal thresholds, notification preferences) as needed.

This comprehensive guide should help you successfully deploy and manage your Hyperliquid Signal System on Cloudflare. If you have further questions, refer to the official Cloudflare documentation or the project's `README.md` file.




## Free Tier Considerations

This project can be deployed on Cloudflare's free tier, but it's important to be aware of the limitations to avoid unexpected charges or service disruptions.

### Cloudflare D1 (Database)

Cloudflare D1 has a free tier that includes:

*   **10 databases**
*   **500 MB of storage per database**
*   **5 million reads per month**
*   **100,000 writes per month**

For most users, the free tier should be sufficient for initial use. However, if you anticipate a large volume of data or high traffic, you may need to upgrade to a paid plan. You can monitor your D1 usage in the Cloudflare dashboard.

### Cloudflare Queues

Cloudflare Queues also has a free tier, but the limits are less generous than D1. The free tier includes:

*   **10 queues**
*   **1 million operations per month** (includes both reads and writes)

This project uses queues for inter-worker communication. While the free tier might be sufficient for low-traffic scenarios, high-frequency signaling could exceed the free tier limits. It's crucial to monitor your Queue usage in the Cloudflare dashboard.

### Workers

Cloudflare Workers have a generous free tier, including:

*   **100,000 requests per day**
*   **10ms CPU time per request**

These limits are generally sufficient for this project's backend workers.

### Recommendations for Free Tier Users

*   **Monitor Usage:** Regularly check your D1 and Queues usage in the Cloudflare dashboard to stay within the free tier limits.
*   **Optimize Data:** Be mindful of the data you store in D1 to stay within the 500MB limit. You might need to periodically prune old data.
*   **Consider Alternatives:** If you consistently exceed the free tier limits, you can either upgrade to a paid Cloudflare plan or consider alternative free database/queue providers that can be integrated with Cloudflare Workers.


