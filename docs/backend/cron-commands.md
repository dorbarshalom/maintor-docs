# CLI Cron Commands

This guide documents the CLI commands available in `maintor-api` for executing and testing specific cron tasks.

## Tasks and CLI Commands

You can run individual tasks or all tasks by invoking the cron script with specific command-line arguments:

### 1. Generating Planned Tickets
Runs the scheduled maintenance trigger to process active templates and generate their future ticket occurrences.
```bash
node run-cron.js --tickets
```

### 2. Sending Weekly Report Emails
Generates the "Downtime over time" report chart using QuickChart and emails it to `dorbarshalom@gmail.com` via Resend.
```bash
node run-cron.js --reports
```

### 3. Running All Cron Tasks (Default)
If no arguments are provided, the script runs all standard cron operations (maintenance trigger and hourly updates).
```bash
node run-cron.js
```
