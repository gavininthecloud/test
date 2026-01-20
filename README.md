

# **README.md \- GitHub Actions CI/CD Migration Guide**

## **📋 Table of Contents**

* Overview  
* Architecture  
* Getting Started  
* Configuration Guide  
* How to Use  
* Troubleshooting  
* FAQs

---

## **🎯 Overview**

This document explains our new GitHub Actions-based CI/CD pipeline that replaces Jenkins. We've migrated our 11-level pipeline (Level 0 through Level 10\) to GitHub Actions with improved monitoring, automated error detection, and email notifications.

### **What's New?**

* ✅ **No more Jenkins maintenance** \- Everything runs on GitHub Actions  
* ✅ **Automated log scanning** \- Automatically detects errors in Cloud Run logs  
* ✅ **Email notifications** \- Get notified immediately when jobs succeed or fail  
* ✅ **Better visibility** \- See execution history right in GitHub  
* ✅ **Easy to modify** \- Simple YAML files for each level

### **Key Benefits**

| Feature | Old (Jenkins) | New (GitHub Actions) |
| ----- | ----- | ----- |
| Infrastructure | Self-hosted Jenkins | GitHub-managed |
| Error Detection | Manual log checking | Automated scanning |
| Notifications | None | Email to team |
| Visibility | Jenkins UI only | GitHub UI \+ Artifacts |
| Maintenance | Weekly updates needed | Zero maintenance |

---

## **🏗️ Architecture**

### **High-Level Flow Diagram**

┌─────────────────────────────────────────────────────────────────┐  
│                    GITHUB ACTIONS PIPELINE                      │  
└─────────────────────────────────────────────────────────────────┘  
                                │  
                                ▼  
        ┌───────────────────────────────────────────┐  
        │   SCHEDULED TRIGGER (Cron Schedule)       │  
        │   • Level 0: 00:00 (Midnight)            │  
        │   • Level 1: 02:00 (2 AM)                │  
        │   • Level 2: 04:00 (4 AM)                │  
        │   • ... (up to Level 10\)                 │  
        └───────────────────────────────────────────┘  
                                │  
                                ▼  
        ┌───────────────────────────────────────────┐  
        │   WORKFLOW FILE (level-XX-XXam.yml)       │  
        │   • Triggers at scheduled time            │  
        │   • Calls reusable template               │  
        └───────────────────────────────────────────┘  
                                │  
                                ▼  
        ┌───────────────────────────────────────────┐  
        │   REUSABLE TEMPLATE (\_level-executor.yml) │  
        │   Step 1: Load Configuration              │  
        └───────────────────────────────────────────┘  
                                │  
                                ▼  
        ┌───────────────────────────────────────────┐  
        │   CONFIGURATION FILE (level-N.yml)        │  
        │   • Branch name                           │  
        │   • Cloud Run job name                    │  
        │   • Infrastructure setup                  │  
        │   • Timeout, region, etc.                 │  
        └───────────────────────────────────────────┘  
                                │  
                                ▼  
        ┌───────────────────────────────────────────┐  
        │   GCP AUTHENTICATION                      │  
        │   • Uses service account key              │  
        │   • Sets up gcloud CLI                    │  
        └───────────────────────────────────────────┘  
                                │  
                                ▼  
        ┌───────────────────────────────────────────┐  
        │   EXECUTE cloud\_runexecute.sh             │  
        │   (Your existing shell script)            │  
        │   • Clones cse-detective-control-repo     │  
        │   • Executes Cloud Run job                │  
        │   • Returns execution ID                  │  
        └───────────────────────────────────────────┘  
                                │  
                                ▼  
        ┌───────────────────────────────────────────┐  
        │   COLLECT LOGS FROM GCP                   │  
        │   • Fetch Cloud Run execution logs        │  
        │   • Query Cloud Logging API               │  
        └───────────────────────────────────────────┘  
                                │  
                                ▼  
        ┌───────────────────────────────────────────┐  
        │   ANALYZE LOGS FOR ERRORS                 │  
        │   • Scan for: ERROR, EXCEPTION, FATAL     │  
        │   • Count occurrences                     │  
        │   • Extract top 50 errors                 │  
        └───────────────────────────────────────────┘  
                                │  
                                ▼  
        ┌───────────────────────────────────────────┐  
        │   UPLOAD ARTIFACTS                        │  
        │   • Raw logs (JSON)                       │  
        │   • Processed logs (Text)                 │  
        │   • Error summary                         │  
        │   • Retained for 30 days                  │  
        └───────────────────────────────────────────┘  
                                │  
                                ▼  
        ┌───────────────────────────────────────────┐  
        │   SEND EMAIL NOTIFICATION                 │  
        │   To: cse@cvshealth.com                   │  
        │   • ✅ SUCCESS                            │  
        │   • ⚠️ SUCCESS WITH ERRORS                │  
        │   • ❌ FAILURE                            │

        └───────────────────────────────────────────┘

---

### **Directory Structure**

your-repository/  
├── .github/  
│   ├── workflows/  
│   │   ├── \_level-executor.yml          \# ⚙️ Reusable template (DON'T MODIFY)  
│   │   ├── level-00-midnight.yml        \# 📅 Level 0 scheduler  
│   │   ├── level-01-2am.yml             \# 📅 Level 1 scheduler  
│   │   ├── level-02-4am.yml             \# 📅 Level 2 scheduler  
│   │   ├── level-03-6am.yml             \# 📅 Level 3 scheduler  
│   │   ├── level-04-8am.yml             \# 📅 Level 4 scheduler  
│   │   ├── level-05-10am.yml            \# 📅 Level 5 scheduler  
│   │   ├── level-06-12pm.yml            \# 📅 Level 6 scheduler  
│   │   ├── level-07-2pm.yml             \# 📅 Level 7 scheduler  
│   │   ├── level-08-4pm.yml             \# 📅 Level 8 scheduler  
│   │   ├── level-09-6pm.yml             \# 📅 Level 9 scheduler  
│   │   ├── level-10-8pm.yml             \# 📅 Level 10 scheduler  
│   │   └── manual-any-level.yml         \# 🔧 Manual trigger  
│   │  
│   └── config/  
│       └── levels/  
│           ├── level-0.yml              \# ⚙️ Level 0 config (MODIFY THIS)  
│           ├── level-1.yml              \# ⚙️ Level 1 config  
│           ├── level-2.yml              \# ⚙️ Level 2 config  
│           ├── level-3.yml              \# ⚙️ Level 3 config  
│           ├── level-4.yml              \# ⚙️ Level 4 config  
│           ├── level-5.yml              \# ⚙️ Level 5 config  
│           ├── level-6.yml              \# ⚙️ Level 6 config  
│           ├── level-7.yml              \# ⚙️ Level 7 config  
│           ├── level-8.yml              \# ⚙️ Level 8 config  
│           ├── level-9.yml              \# ⚙️ Level 9 config  
│           └── level-10.yml             \# ⚙️ Level 10 config  
│  
├── cse-detective-control/  
│   └── src/  
│       └── utils/  
│           └── cloud\_runexecute.sh      \# 🔨 Your existing script  
│

└── README.md                             \# 📖 This file

---

## **🚀 Getting Started**

### **Prerequisites**

Before you start, make sure you have:

1. ✅ **GitHub repository access** with write permissions  
2. ✅ **GCP service account key** with Cloud Run execution permissions  
3. ✅ **SMTP credentials** for sending emails  
4. ✅ **Basic knowledge** of YAML files

### **Initial Setup (One-Time)**

#### **Step 1: Add GitHub Secrets**

Navigate to your repository → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

Add these secrets:

| Secret Name | Description | Example Value |
| ----- | ----- | ----- |
| `GCP_SA_KEY` | GCP Service Account JSON key | `{"type": "service_account", ...}` |
| `SMTP_SERVER` | SMTP server address | `smtp.office365.com` |
| `SMTP_PORT` | SMTP port number | `587` |
| `SMTP_USERNAME` | Email username | `cicd-notifications@cvshealth.com` |
| `SMTP_PASSWORD` | Email password/app password | `your-password-here` |

**🔒 Important:** Never commit secrets directly in code\!

#### **Step 2: Verify File Structure**

Make sure all workflow files and config files are in the correct directories as shown in the Directory Structure section above.

#### **Step 3: Test with One Level**

Before rolling out to all levels, test with Level 0:

1. Go to **Actions** tab in GitHub  
2. Click on **"Manual Level Execution"**  
3. Click **"Run workflow"**  
4. Select **Level: 0**  
5. Click **"Run workflow"** button  
6. Watch the execution and verify:  
   * ✅ Workflow runs successfully  
   * ✅ Cloud Run job executes  
   * ✅ Email is received  
   * ✅ Logs are uploaded as artifacts

---

## **⚙️ Configuration Guide**

### **Understanding Configuration Files**

Each level has its own configuration file (e.g., `level-0.yml`, `level-1.yml`). These files contain all the parameters needed to run that specific level.

### **Configuration File Structure**

yaml  
*\# .github/config/levels/level-0.yml*

*\# Basic Information*  
level: 0  
description: "Level 0 \- Initial data ingestion and validation"

*\# Git Repository Settings*  
branch\_name: main                          *\# Branch to use*  
repository: cse-detective-control          *\# Repository name*

*\# Cloud Run Settings*  
cloud\_run\_job\_name: cse-detective-level-0-prod     *\# GCP Cloud Run job name*  
cloud\_run\_infra\_setup: level0-prod-infra            *\# Infrastructure setup identifier*  
region: us-central1                                  *\# GCP region*

*\# Execution Settings*  
timeout: 600                                *\# Timeout in seconds (10 minutes)*  
parallel\_jobs: 5                           *\# Max parallel executions*

*\# Notification Settings*  
notify\_on\_success: true                    *\# Send email on success*  
notify\_on\_failure: true                    *\# Send email on failure*

*\# Metadata*  
owner: data-engineering-team               *\# Team responsible*

sla\_minutes: 30                           *\# Expected completion time*

### **How to Modify a Configuration**

**Example: Change Cloud Run job region for Level 3**

1. Open the file:

bash

  vim .github/config/levels/level-3.yml

2. Find the region line:

yaml

  region: us-central1

3. Change to new region:

yaml

  region: us-west1

4. Save the file  
5. Commit and push:

bash  
  git add .github/config/levels/level-3.yml  
   git commit \-m "Change Level 3 region to us-west1"

   git push

6. **That's it\!** Next scheduled run will use the new region.

### **Common Configuration Changes**

#### **Change Branch Name**

yaml  
*\# From:*  
branch\_name: main

*\# To:*

branch\_name: develop

#### **Change Timeout**

yaml  
*\# From:*  
timeout: 600    *\# 10 minutes*

*\# To:*

timeout: 1800   *\# 30 minutes*

#### **Change Cloud Run Job Name**

yaml  
*\# From:*  
cloud\_run\_job\_name: cse-detective-level-5-prod

*\# To:*  
cloud\_run\_job\_name: cse-detective-level-5-staging  
\`\`\`

\---

*\#\# 📖 How to Use*

*\#\#\# Scheduled Execution (Automatic)*

Each level runs automatically on its schedule. You don't need to do anything\!

| Level | Schedule | Time (UTC) | Time (IST) |  
|-------|----------|------------|------------|  
| 0 | Daily midnight | 00:00 | 05:30 AM |  
| 1 | Daily | 02:00 | 07:30 AM |  
| 2 | Daily | 04:00 | 09:30 AM |  
| 3 | Daily | 06:00 | 11:30 AM |  
| 4 | Daily | 08:00 | 01:30 PM |  
| 5 | Daily | 10:00 | 03:30 PM |  
| 6 | Daily | 12:00 | 05:30 PM |  
| 7 | Daily | 14:00 | 07:30 PM |  
| 8 | Daily | 16:00 | 09:30 PM |  
| 9 | Daily | 18:00 | 11:30 PM |  
| 10 | Daily | 20:00 | 01:30 AM (next day) |

*\#\#\# Manual Execution*

\*\*When to use:\*\* Testing, re-running failed jobs, or executing outside schedule.

\*\*Steps:\*\*

1\. Go to your repository on GitHub  
2\. Click on \*\*"Actions"\*\* tab  
3\. Click on \*\*"Manual Level Execution"\*\* workflow  
4\. Click \*\*"Run workflow"\*\* button (top right)  
5\. Select the level you want to run (0-10)  
6\. (Optional) Override branch name if needed  
7\. Click \*\*"Run workflow"\*\*

\*\*Screenshot Guide:\*\*  
\`\`\`  
GitHub Repository  
    ↓  
Actions Tab  
    ↓  
Manual Level Execution  
    ↓  
Run workflow button  
    ↓  
Select Level: \[dropdown: 0, 1, 2, ..., 10\]  
    ↓  
Click "Run workflow"  
\`\`\`

*\#\#\# Viewing Execution Results*

*\#\#\#\# Option 1: GitHub Actions UI*

1\. Go to \*\*Actions\*\* tab  
2\. Click on the workflow run you want to see  
3\. Click on the job to see detailed logs  
4\. Download artifacts if needed

*\#\#\#\# Option 2: Email Notification*

You'll receive an email at \*\*cse@cvshealth.com\*\* with:  
\- Execution status (SUCCESS / WARNING / FAILURE)  
\- Error count (if any)  
\- Direct links to logs and artifacts  
\- Quick action buttons

*\#\#\#\# Option 3: Download Artifacts*

1\. Go to workflow run page  
2\. Scroll to bottom  
3\. Click on \*\*"Artifacts"\*\* section  
4\. Download logs zip file  
5\. Unzip and view logs

\---

*\#\# 🔍 Understanding Email Notifications*

*\#\#\# Email Status Types*

*\#\#\#\# ✅ SUCCESS*  
\`\`\`  
Subject: ✅ Level 0 \- SUCCESS \- cse-detective-level-0-prod \[42\]

\- All jobs completed successfully  
\- No errors found in logs  
\- No action needed  
\`\`\`

*\#\#\#\# ⚠️ SUCCESS WITH ERRORS*  
\`\`\`  
Subject: ⚠️ Level 3 \- SUCCESS WITH ERRORS \- cse-detective-level-3-prod \[128\]

\- Jobs completed but errors were found in logs  
\- Check error details in email  
\- Review if errors are critical  
\- Consider re-running if needed  
\`\`\`

*\#\#\#\# ❌ FAILURE*  
\`\`\`  
Subject: ❌ Level 7 \- FAILURE \- cse-detective-level-7-prod \[205\]

\- Job execution failed  
\- Check error logs immediately  
\- Fix the issue

\- Re-run manually

### **Email Content**

Every email contains:

1. **Execution Summary**  
   * Level number  
   * Status  
   * Cloud Run job name  
   * Branch used  
   * Region  
   * Error count  
2. **Error Analysis** (if errors found)  
   * Total error count  
   * Top 50 error messages  
   * Error types (ERROR, EXCEPTION, FATAL, etc.)  
3. **Quick Actions**  
   * View Full Logs button  
   * Download Artifacts button  
   * GCP Console link

---

## **🔧 Troubleshooting**

### **Common Issues and Solutions**

#### **Issue 1: Workflow Not Running on Schedule**

**Symptoms:**

* Workflow should run at scheduled time but doesn't

**Possible Causes & Solutions:**

1. **GitHub Actions disabled**  
   * Go to **Settings** → **Actions** → **General**  
   * Make sure "Allow all actions" is selected  
   * Enable workflows if disabled  
2. **Schedule syntax error**  
   * Check cron syntax in workflow file  
   * Use [crontab.guru](https://crontab.guru/) to validate  
3. **Repository is private and inactive**  
   * GitHub disables scheduled workflows after 60 days of no activity  
   * Make a commit or run workflow manually to re-enable

**Fix:**

bash  
*\# Make a dummy commit to re-activate*  
git commit \--allow-empty \-m "Re-activate scheduled workflows"  
git push  
\`\`\`

\---

*\#\#\#\# Issue 2: GCP Authentication Failed*

\*\*Symptoms:\*\*  
\`\`\`

Error: google-github-actions/auth failed with: failed to generate Google Cloud credentials

**Solution:**

1. Verify `GCP_SA_KEY` secret exists:  
   * Settings → Secrets → Actions  
   * Check if `GCP_SA_KEY` is listed  
2. Verify service account has correct permissions:

bash  
  *\# Required roles:*  
   \- Cloud Run Admin  
   \- Logs Viewer

   \- Service Account User

3. Verify JSON key is valid:  
   * Should be valid JSON format  
   * Should not have extra quotes or formatting  
4. Re-generate and update secret if needed

---

#### **Issue 3: Email Not Received**

**Symptoms:**

* Workflow runs successfully but no email received

**Possible Causes & Solutions:**

1. **Check spam folder**  
   * Emails might be filtered as spam  
   * Add sender to safe list  
2. **SMTP credentials incorrect**  
   * Verify secrets: `SMTP_SERVER`, `SMTP_PORT`, `SMTP_USERNAME`, `SMTP_PASSWORD`  
   * Test credentials using mail client  
3. **Firewall blocking SMTP**  
   * GitHub Actions needs outbound access to SMTP server  
   * Contact IT to whitelist SMTP server  
4. **Check workflow logs**  
   * Look for email sending step  
   * Check for error messages

**Debug:**

yaml  
*\# Add this to notification step to see actual error*  
\- name: Send email  
  continue-on-error: true  
  *\# ... rest of step*

*\# Then check logs for detailed error*  
\`\`\`

\---

*\#\#\#\# Issue 4: Cloud Run Job Not Executing*

\*\*Symptoms:\*\*  
\`\`\`

Error: gcloud run jobs execute failed

**Solution:**

1. **Verify Cloud Run job exists:**

bash

  gcloud run jobs list \--region\=us-central1

2. **Check job name in config:**  
   * Open `.github/config/levels/level-N.yml`  
   * Verify `cloud_run_job_name` matches actual job name in GCP  
3. **Check region:**  
   * Verify `region` in config matches where job is deployed  
4. **Check permissions:**  
   * Service account needs `roles/run.admin`

---

#### **Issue 5: Logs Not Collected**

**Symptoms:**

* Workflow runs but no logs in artifacts  
* Error count shows 0 but you know there were errors

**Solution:**

1. **Check Cloud Logging API is enabled:**

bash

  gcloud services list \--enabled | grep logging

2. **Verify log retention settings:**  
   * Logs must be available when collection runs  
   * Default retention is 30 days  
3. **Check time filters:**  
   * Log collection uses `--freshness=30m`  
   * If job runs longer, increase this value  
4. **Verify service account permissions:**  
   * Needs `roles/logging.viewer`

---

#### **Issue 6: Configuration File Changes Not Taking Effect**

**Symptoms:**

* Modified configuration but workflow still uses old values

**Solution:**

1. **Verify file is committed and pushed:**

bash  
  git status

   git log \--oneline \-5

2. **Check file path is correct:**  
   * Must be exactly: `.github/config/levels/level-N.yml`  
   * Case sensitive\!  
3. **Verify workflow references correct config:**

yaml  
  *\# In level-N-schedule.yml*  
   with:

     config\_file: '.github/config/levels/level-N.yml'  *\# Check this path*

4. **Clear GitHub Actions cache (if needed):**  
   * Re-run workflow manually  
   * Sometimes cache needs to refresh

---

### **Getting Help**

If you're still stuck:

1. **Check workflow logs:**  
   * Actions → Select run → Click job → Read detailed logs  
2. **Check artifacts:**  
   * Download log artifacts  
   * Look for error patterns  
3. **Ask the team:**  
   * DevOps team Slack channel: `#devops-support`  
   * Tag: `@devops-oncall`  
4. **Create an issue:**  
   * Use GitHub Issues in repository  
   * Include:  
     * Level number  
     * Workflow run URL  
     * Error message  
     * What you tried

---

## **❓ FAQs**

### **General Questions**

**Q: What happens to Jenkins? Can I still use it?**

A: Jenkins will be decommissioned after successful migration. During transition period (30 days), both systems will run in parallel for validation.

**Q: Will this affect my existing Cloud Run jobs?**

A: No\! The same `cloud_runexecute.sh` script is used. Only the trigger mechanism (Jenkins → GitHub Actions) has changed.

**Q: Can I run levels out of order?**

A: Yes\! Each level is independent. Use the manual trigger to run any level any time.

**Q: What if I need to run a level on a different branch?**

A: Use the manual trigger workflow and provide branch override parameter.

---

### **Configuration Questions**

**Q: Can I change the schedule for a level?**

A: Yes\! Edit the cron expression in the workflow file:

yaml  
*\# .github/workflows/level-03-6am.yml*  
on:  
  schedule:

    \- cron: '0 6 \* \* \*'  *\# Change this line*

**Q: Can I add a new Level 11?**

A: Yes\! Follow these steps:

1. Copy `.github/config/levels/level-10.yml` to `level-11.yml`  
2. Modify parameters in `level-11.yml`  
3. Copy `.github/workflows/level-10-8pm.yml` to `level-11-10pm.yml`  
4. Update schedule and level number  
5. Commit and push

**Q: Can different levels use different branches?**

A: Yes\! Each level config has its own `branch_name` parameter. Change it independently.

**Q: Can I disable email notifications for a specific level?**

A: Yes\! In the level config:

yaml  
notify\_on\_success: false  *\# No email on success*  
notify\_on\_failure: true   *\# Still email on failure*  
\`\`\`

\---

*\#\#\# Execution Questions*

\*\*Q: How do I know if my level ran successfully?\*\*

A: Three ways:  
1\. Check email notification  
2\. Check GitHub Actions UI  
3\. Check GCP Cloud Run console

\*\*Q: Can I see historical execution logs?\*\*

A: Yes\!  
\- GitHub Actions: Last 90 days (configurable)  
\- Artifacts: 30 days (configurable)  
\- GCP Cloud Logging: 30 days (default)

\*\*Q: What happens if a level fails?\*\*

A:   
1\. Email sent immediately to team  
2\. Other levels continue running (no impact)  
3\. You can re-run manually  
4\. Logs are preserved in artifacts

\*\*Q: Can I cancel a running job?\*\*

A: Yes\!  
1\. Go to Actions tab  
2\. Click on running workflow  
3\. Click "Cancel workflow" button (top right)

\---

*\#\#\# Troubleshooting Questions*

\*\*Q: How do I debug a failed level?\*\*

A:  
1\. Check email for error summary  
2\. Go to Actions tab → Failed run → View logs  
3\. Download artifacts for detailed logs  
4\. Check GCP Cloud Run console for job status

\*\*Q: Why am I not getting email notifications?\*\*

A: Check:  
1\. Spam folder  
2\. SMTP secrets are correct  
3\. Workflow logs for email sending step errors  
4\. Firewall not blocking SMTP

\*\*Q: How do I test changes without affecting production?\*\*

A:  
1\. Create a feature branch  
2\. Modify config files  
3\. Use manual trigger with branch override  
4\. Test thoroughly  
5\. Merge to main when confident

\---

*\#\# 📚 Additional Resources*

*\#\#\# Documentation*

\- \[GitHub Actions Documentation\](https://docs.github.com/en/actions)  
\- \[GCP Cloud Run Documentation\](https://cloud.google.com/run/docs)  
\- \[Cron Schedule Syntax\](https://crontab.guru/)

*\#\#\# Internal Resources*

\- DevOps Wiki: \`https://wiki.cvshealth.com/devops/github-actions\`  
\- Team Slack: \`*\#devops-support\`*  
\- On-call rotation: \`https://pagerduty.cvshealth.com/devops\`

*\#\#\# Contact*

\- \*\*DevOps Team Lead:\*\* \[Name\] \- \[email@cvshealth.com\]  
\- \*\*Platform Engineering:\*\* \[Name\] \- \[email@cvshealth.com\]  
\- \*\*Support Channel:\*\* \`*\#devops-support\` on Slack*

\---

*\#\# 🔄 Version History*

| Version | Date | Changes | Author |  
|---------|------|---------|--------|  
| 1.0 | 2026-01-20 | Initial migration from Jenkins | DevOps Team |

\---

*\#\# 📝 License*

Internal CVS Health Documentation \- Confidential

\---

*\# 🎯 Quick Start Checklist*

Use this checklist for your first time setup:

\- \[ \] Add all GitHub Secrets (GCP\_SA\_KEY, SMTP credentials)  
\- \[ \] Verify all workflow files are in \`.github/workflows/\`  
\- \[ \] Verify all config files are in \`.github/config/levels/\`  
\- \[ \] Test Level 0 manually  
\- \[ \] Verify email notification received  
\- \[ \] Verify logs uploaded as artifacts  
\- \[ \] Check GCP Cloud Run job executed  
\- \[ \] Review email content  
\- \[ \] Share documentation with team  
\- \[ \] Set up monitoring alerts  
\- \[ \] Schedule team training session

\---

\*\*Need Help?\*\* Contact DevOps team on Slack: \`*\#devops-support\`*

\---

Now I'll create the Google Docs version with an Indian English writing style (which tends to be more formal, detailed, and explanatory):

\---

*\# 📄 Google Docs Version (Indian English Style)*

I'll create this as formatted text that you can copy into Google Docs:

\---

\*\*SUBJECT: GitHub Actions CI/CD Pipeline \- Implementation Guide\*\*

\*\*TO: Engineering Team\*\*  
\*\*FROM: DevOps Team\*\*  
\*\*DATE: 20th January, 2026\*\*

\---

*\#\# TABLE OF CONTENTS*

1\. Introduction and Background  
2\. System Architecture Overview  
3\. Prerequisites and Initial Setup  
4\. Configuration Management  
5\. Operational Procedures  
6\. Monitoring and Notifications  
7\. Troubleshooting Guidelines  
8\. Frequently Asked Questions  
9\. Annexures

\---

*\#\# 1\. INTRODUCTION AND BACKGROUND*

*\#\#\# 1.1 Purpose of This Document*

This document has been prepared to provide comprehensive guidance regarding the migration of our existing Jenkins-based CI/CD pipeline to GitHub Actions. The same has been done keeping in mind the ease of understanding for all team members, irrespective of their prior experience with GitHub Actions.

*\#\#\# 1.2 Background*

As you all are aware, we have been using Jenkins for orchestrating our multi-level data processing pipeline consisting of 11 independent levels (Level 0 through Level 10). Each level executes specific Cloud Run jobs at scheduled intervals with proper time gaps between consecutive levels.

However, due to increasing maintenance overhead and limitations in visibility, it was decided to migrate to GitHub Actions, which will provide us with:

\- Reduced infrastructure management burden  
\- Native integration with our source code repository  
\- Automated error detection and analysis  
\- Proactive email notifications to the team  
\- Better audit trail and execution history

*\#\#\# 1.3 Scope of Migration*

The migration covers all 11 pipeline levels while retaining our existing \`cloud\_runexecute.sh\` script. We are not making any changes to the actual job execution logic, only to the triggering and monitoring mechanism.

\---

*\#\# 2\. SYSTEM ARCHITECTURE OVERVIEW*

*\#\#\# 2.1 High-Level Architecture Diagram*

Kindly refer to the diagram below which illustrates the complete flow of execution:  
\`\`\`  
\[GitHub Schedule Trigger\]  
        ↓  
\[Scheduled Workflow File\]  
        ↓  
\[Reusable Workflow Template\]  
        ↓  
\[Load Configuration from YAML\]  
        ↓  
\[GCP Authentication\]  
        ↓  
\[Execute cloud\_runexecute.sh\]  
        ↓  
\[Collect Cloud Run Logs\]  
        ↓  
\[Analyze for Errors\]  
        ↓  
\[Upload Artifacts\]  
        ↓  
\[Send Email Notification\]  
\`\`\`

*\#\#\# 2.2 Component Description*

*\#\#\#\# 2.2.1 Scheduled Workflow Files*  
These are individual workflow files (one per level) which contain the schedule (cron expression) and trigger the actual execution. For example, \`level-00-midnight.yml\` runs at midnight every day.

*\#\#\#\# 2.2.2 Reusable Workflow Template*  
This is a single template file (\`\_level-executor.yml\`) which contains the actual execution logic. All 11 levels use this same template, thereby avoiding code duplication and ensuring consistency.

*\#\#\#\# 2.2.3 Configuration Files*  
Each level has its own configuration file (\`.github/config/levels/level-N.yml\`) which contains all the parameters specific to that level such as branch name, Cloud Run job name, infrastructure setup, region, timeout, etc.

*\#\#\#\# 2.2.4 Existing Shell Script*  
Our existing \`cloud\_runexecute.sh\` script remains unchanged and continues to perform the same function of cloning the repository and executing Cloud Run jobs.

*\#\#\# 2.3 Key Design Decisions*

*\#\#\#\# 2.3.1 Why Static Configuration Files?*

After careful evaluation of multiple approaches (static config files, dynamic config, hardcoded parameters), we have opted for static configuration files for the following reasons:

\*\*Advantages:\*\*  
\- Clear separation of concerns \- each level has its own config file  
\- Easy to modify one level without affecting others  
\- Better version control \- changes to one level don't create noise in git history  
\- Simple to understand and maintain  
\- No complex logic required  
\- Easy code reviews

\*\*Comparison with Other Approaches:\*\*

| Aspect | Static Config (Our Choice) | Dynamic Config | Hardcoded |  
|--------|---------------------------|----------------|-----------|  
| Ease of Understanding | Very Easy | Moderate | Easy |  
| Ease of Modification | Very Easy | Moderate | Difficult |  
| Maintainability | Excellent | Moderate | Poor |  
| Risk of Breaking Other Levels | Very Low | Medium | Low |  
| Code Duplication | Minimal | None | High |

\---

*\#\# 3\. PREREQUISITES AND INITIAL SETUP*

*\#\#\# 3.1 Prerequisites*

Before proceeding with the setup, please ensure you have the following:

1\. \*\*GitHub Repository Access\*\*  
   \- You must have write access to the repository  
   \- Administrator access required for creating secrets

2\. \*\*GCP Service Account\*\*  
   \- Service account JSON key file  
   \- The account should have following roles:  
     \- Cloud Run Admin  
     \- Logs Viewer  
     \- Service Account User

3\. \*\*SMTP Credentials\*\*  
   \- SMTP server address  
   \- Port number  
   \- Username and password for authentication

4\. \*\*Basic Knowledge\*\*  
   \- Familiarity with YAML syntax  
   \- Understanding of our existing Jenkins pipeline  
   \- Basic Git operations (commit, push, pull)

*\#\#\# 3.2 Initial Setup Procedure*

The setup needs to be done only once. Please follow the below steps carefully:

*\#\#\#\# Step 3.2.1: Configure GitHub Secrets*

Secrets are encrypted environment variables that are used by GitHub Actions. We need to configure the following secrets:

\*\*Procedure:\*\*

1\. Navigate to your repository on GitHub  
2\. Click on "Settings" tab  
3\. In the left sidebar, click on "Secrets and variables"  
4\. Click on "Actions"  
5\. Click on "New repository secret" button

\*\*Secrets to Add:\*\*

| Secret Name | Description | Where to Get It |  
|-------------|-------------|-----------------|  
| GCP\_SA\_KEY | Complete JSON content of GCP service account key | From GCP Console → IAM & Admin → Service Accounts |  
| SMTP\_SERVER | SMTP server hostname | From IT team / Email admin |  
| SMTP\_PORT | SMTP port number (usually 587 or 465\) | From IT team / Email admin |  
| SMTP\_USERNAME | Email account username | Your service account email |  
| SMTP\_PASSWORD | Email account password | Your service account password |

\*\*Important Security Notes:\*\*  
\- Never commit these secrets in code  
\- Do not share secrets via email or Slack  
\- Rotate credentials periodically (every 90 days recommended)

*\#\#\#\# Step 3.2.2: Verify Directory Structure*

Please ensure all files are placed in correct directories as per the following structure:  
\`\`\`  
Repository Root/  
├── .github/  
│   ├── workflows/  
│   │   ├── \_level-executor.yml  
│   │   ├── level-00-midnight.yml  
│   │   ├── level-01-2am.yml  
│   │   └── ... (remaining workflow files)  
│   └── config/  
│       └── levels/  
│           ├── level-0.yml  
│           ├── level-1.yml

│           └── ... (remaining config files)

#### **Step 3.2.3: Initial Testing**

Before going live, we should test with one level (Level 0\) to ensure everything is working correctly.

**Testing Procedure:**

1. Go to GitHub repository  
2. Click on "Actions" tab  
3. Click on "Manual Level Execution" workflow  
4. Click "Run workflow" button  
5. Select Level: 0  
6. Click "Run workflow"

**Verification Checklist:**

* Workflow starts executing  
* GCP authentication succeeds  
* Configuration loads correctly  
* cloud\_runexecute.sh executes successfully  
* Cloud Run job completes  
* Logs are collected  
* Email notification is received at cse@cvshealth.com  
* Artifacts are uploaded

If all items are checked, setup is successful\!

---

## **4\. CONFIGURATION MANAGEMENT**

### **4.1 Understanding Configuration Files**

Configuration files are YAML format files that contain all the parameters required for executing a particular level. Each level has its own separate configuration file.

### **4.2 Configuration File Structure**

Below is the detailed explanation of each field in the configuration file:

yaml  
*\# Level identifier*  
level: 0

*\# Human-readable description*  
description: "Level 0 \- Initial data ingestion and validation"

*\# Git repository settings*  
branch\_name: main                    *\# Which branch to checkout*  
repository: cse-detective-control    *\# Repository name*

*\# Cloud Run job settings*  
cloud\_run\_job\_name: cse-detective-level-0-prod     *\# Exact job name in GCP*  
cloud\_run\_infra\_setup: level0-prod-infra            *\# Infrastructure identifier*  
region: us-central1                                  *\# GCP region*

*\# Execution parameters*  
timeout: 600                         *\# Maximum execution time (seconds)*  
parallel\_jobs: 5                     *\# How many jobs can run in parallel*

*\# Notification preferences*  
notify\_on\_success: true              *\# Send email when succeeds*  
notify\_on\_failure: true              *\# Send email when fails*

*\# Metadata for tracking*  
owner: data-engineering-team         *\# Responsible team*

sla\_minutes: 30                     *\# Expected completion time*

### **4.3 How to Modify Configuration**

Modifying configuration is straightforward. However, please follow the proper procedure to avoid mistakes.

**Example Scenario:** You need to change the region for Level 3 from `us-central1` to `us-west1`

**Step-by-Step Procedure:**

1. **Clone the repository (if not already done)**

bash  
  git clone \<repository-url\>

   cd \<repository-name\>

2. **Create a new branch**

bash

  git checkout \-b update-level-3-region

3. **Edit the configuration file**

bash

  vim .github/config/levels/level-3.yml

Find the line:

yaml

  region: us-central1

Change to:

yaml

  region: us-west1

4. **Verify your changes**

bash

  git diff .github/config/levels/level-3.yml

The output should show:

diff  
 \- region: us-central1

  \+ region: us-west1

5. **Commit your changes**

bash  
  git add .github/config/levels/level-3.yml

   git commit \-m "Update Level 3 region to us-west1"

6. **Push your changes**

bash

  git push origin update-level-3-region

7. **Create Pull Request**  
   * Go to GitHub repository  
   * You will see a notification to create Pull Request  
   * Click on it and create PR  
   * Add proper description explaining why you made this change  
   * Request review from team members  
8. **After PR Approval**  
   * Merge the PR  
   * Delete the feature branch  
9. **Verification**  
   * Either wait for scheduled run, or  
   * Trigger manual execution to verify the change

**Important Notes:**

* Always create a new branch for changes  
* Never push directly to main branch  
* Always get peer review before merging  
* Test your changes using manual trigger before scheduled run

### **4.4 Common Configuration Modifications**

Below are some common scenarios you might encounter:

#### **4.4.1 Changing Branch Name**

**Scenario:** You want to test with a feature branch before deploying to production.

yaml  
*\# From:*  
branch\_name: main

*\# To:*

branch\_name: feature/new-logic-implementation

After changing, test using manual trigger to ensure it works correctly.

#### **4.4.2 Modifying Timeout**

**Scenario:** Your job started taking longer to complete, and it's timing out.

yaml  
*\# From:*  
timeout: 600      *\# 10 minutes*

*\# To:*

timeout: 1800     *\# 30 minutes*

Note: Timeout is in seconds. Convert minutes to seconds (minutes × 60).

#### **4.4.3 Changing Cloud Run Job Name**

**Scenario:** You deployed a new version of the job with a different name.

yaml  
*\# From:*  
cloud\_run\_job\_name: cse-detective-level-5-prod

*\# To:*

cloud\_run\_job\_name: cse-detective-level-5-prod-v2

Ensure the new job name actually exists in GCP before making this change\!

#### **4.4.4 Disabling Notifications**

**Scenario:** During maintenance window, you don't want to receive notifications.

yaml  
*\# From:*  
notify\_on\_success: true  
notify\_on\_failure: true

*\# To:*  
notify\_on\_success: false  
notify\_on\_failure: false  
\`\`\`

Remember to re-enable after maintenance\!

\---

*\#\# 5\. OPERATIONAL PROCEDURES*

*\#\#\# 5.1 Scheduled Execution*

The pipeline runs automatically on schedule. You don't need to do anything for routine executions.

\*\*Schedule Details:\*\*

| Level | Time (UTC) | Time (IST) | Frequency |  
|-------|-----------|------------|-----------|  
| 0 | 00:00 | 05:30 AM | Daily |  
| 1 | 02:00 | 07:30 AM | Daily |  
| 2 | 04:00 | 09:30 AM | Daily |  
| 3 | 06:00 | 11:30 AM | Daily |  
| 4 | 08:00 | 01:30 PM | Daily |  
| 5 | 10:00 | 03:30 PM | Daily |  
| 6 | 12:00 | 05:30 PM | Daily |  
| 7 | 14:00 | 07:30 PM | Daily |  
| 8 | 16:00 | 09:30 PM | Daily |  
| 9 | 18:00 | 11:30 PM | Daily |  
| 10 | 20:00 | 01:30 AM | Daily |

\*\*Note:\*\* All times are UTC. Please convert to your local timezone if needed.

*\#\#\# 5.2 Manual Execution*

There might be situations where you need to run a level manually:  
\- Testing after configuration changes  
\- Re-running a failed level  
\- Running outside schedule for urgent data processing

\*\*Procedure for Manual Execution:\*\*

1\. Open your browser and navigate to GitHub repository

2\. Click on "Actions" tab in the top menu

3\. In the left sidebar, click on "Manual Level Execution"

4\. On the right side, click "Run workflow" dropdown button

5\. You will see a form with following fields:  
   \- \*\*Branch:\*\* Keep it as "main" (unless you specifically want to test a feature branch)  
   \- \*\*Level:\*\* Select the level number you want to execute (dropdown will show 0 through 10\)  
   \- \*\*Branch Override:\*\* (Optional) If you want to use a different branch than what's in config, enter it here

6\. Click the "Run workflow" green button

7\. The workflow will start executing. Refresh the page to see the status.

8\. Click on the running workflow to see detailed logs

\*\*When to Use Manual Execution:\*\*

\- \*\*After Config Changes:\*\* Always test manually before waiting for scheduled run  
\- \*\*Failed Execution:\*\* Re-run immediately rather than waiting for next schedule  
\- \*\*Data Re-processing:\*\* If you need to process data again  
\- \*\*Testing:\*\* When testing new features or configurations

*\#\#\# 5.3 Monitoring Execution Status*

You can monitor execution status in multiple ways:

*\#\#\#\# 5.3.1 GitHub Actions UI*

\*\*Advantages:\*\*  
\- Real-time status updates  
\- Detailed logs  
\- Artifact downloads  
\- Can see past runs

\*\*How to Access:\*\*  
1\. Go to repository → Actions tab  
2\. You'll see list of all workflow runs  
3\. Color coding:  
   \- 🟢 Green checkmark \= Success  
   \- 🔴 Red X \= Failed  
   \- 🟡 Yellow circle \= Running  
4\. Click on any run to see details

*\#\#\#\# 5.3.2 Email Notifications*

\*\*Advantages:\*\*  
\- Immediate notification in your inbox  
\- Don't need to check GitHub manually  
\- Includes error summary and direct links

\*\*What You'll Receive:\*\*  
\- Email sent to: cse@cvshealth.com  
\- Subject line includes: Status emoji, Level number, Status text, Run number  
\- Email body includes: Execution details, Error count, Quick action links

*\#\#\#\# 5.3.3 GCP Cloud Run Console*

\*\*Advantages:\*\*  
\- See actual Cloud Run job status  
\- View GCP-level logs  
\- Check resource utilization

\*\*How to Access:\*\*  
1\. Go to GCP Console  
2\. Navigate to Cloud Run → Jobs  
3\. Find the specific job (e.g., cse-detective-level-0-prod)  
4\. View execution history

*\#\#\# 5.4 Downloading and Analyzing Logs*

Logs are very important for troubleshooting. Here's how to access them:

*\#\#\#\# 5.4.1 From GitHub Actions*

\*\*Steps:\*\*  
1\. Go to Actions tab  
2\. Click on the workflow run  
3\. Scroll to bottom of page  
4\. Under "Artifacts" section, you'll see zip files  
5\. Click to download  
6\. Unzip the file  
7\. You'll find:  
   \- \`raw-logs.json\` \- Complete raw logs  
   \- \`execution.log\` \- Processed text logs  
   \- \`errors.txt\` \- Extracted errors (if any)

\*\*Retention:\*\* Artifacts are kept for 30 days by default.

*\#\#\#\# 5.4.2 From GCP Cloud Logging*

\*\*Steps:\*\*  
1\. Go to GCP Console  
2\. Navigate to Logging → Logs Explorer  
3\. Use filter:  
\`\`\`  
   resource.type="cloud\_run\_job"  
   resource.labels.job\_name="YOUR\_JOB\_NAME"  
\`\`\`  
4\. Select time range  
5\. View or export logs

\---

*\#\# 6\. MONITORING AND NOTIFICATIONS*

*\#\#\# 6.1 Email Notification System*

Our new pipeline includes automated email notifications for every execution. This is a major improvement over Jenkins where we had to manually check logs.

*\#\#\# 6.2 Email Status Types*

*\#\#\#\# 6.2.1 Success (✅)*

\*\*When You Get This:\*\*  
\- All jobs completed successfully  
\- No errors found in logs  
\- Everything is working perfectly

\*\*Email Subject Example:\*\*  
\`\`\`  
✅ Level 0 \- SUCCESS \- cse-detective-level-0-prod \[42\]  
\`\`\`

\*\*What to Do:\*\*  
\- No action required  
\- This is just for your information  
\- You can review the execution summary if needed

*\#\#\#\# 6.2.2 Success with Errors (⚠️)*

\*\*When You Get This:\*\*  
\- Jobs completed and returned success status  
\- BUT errors/warnings were found in Cloud Run logs  
\- Might need investigation

\*\*Email Subject Example:\*\*  
\`\`\`  
⚠️ Level 3 \- SUCCESS WITH ERRORS \- cse-detective-level-3-prod \[128\]  
\`\`\`

\*\*What to Do:\*\*  
1\. Open the email and review error summary  
2\. Check if errors are critical or just warnings  
3\. Download full logs if needed  
4\. Determine if re-run is required  
5\. If errors are acceptable, no action needed  
6\. If errors are critical, investigate and fix

*\#\#\#\# 6.2.3 Failure (❌)*

\*\*When You Get This:\*\*  
\- Job execution failed  
\- Script returned non-zero exit code  
\- Cloud Run job didn't complete successfully

\*\*Email Subject Example:\*\*  
\`\`\`

❌ Level 7 \- FAILURE \- cse-detective-level-7-prod \[205\]

**What to Do (URGENT):**

1. Open email immediately and check error details  
2. Go to GitHub Actions and check detailed logs  
3. Go to GCP Console and check Cloud Run job status  
4. Identify root cause:  
   * Infrastructure issue?  
   * Code bug?  
   * Configuration problem?  
   * Timeout?  
5. Fix the issue  
6. Re-run manually to verify fix  
7. If issue persists, escalate to team lead

### **6.3 Email Content Details**

Every email contains the following sections:

#### **6.3.1 Header Section**

* Status emoji and text (✅/⚠️/❌)  
* Level number  
* Cloud Run job name

#### **6.3.2 Execution Summary Table**

| Field | Description |
| ----- | ----- |
| Level | Which level was executed |
| Status | SUCCESS / SUCCESS WITH ERRORS / FAILURE |
| Cloud Run Job | Name of the GCP Cloud Run job |
| Branch | Which branch was used |
| Region | GCP region where job ran |
| Errors Found | Number of errors detected |
| Run Number | Sequential run number for tracking |
| Triggered By | Who/what triggered this execution |

#### **6.3.3 Error Analysis Section**

(Shown only if errors were detected)

* Total error count  
* Error breakdown by type (ERROR, EXCEPTION, FATAL)  
* Top 50 error messages  
* Timestamps of when errors occurred

#### **6.3.4 Quick Action Buttons**

* **View Full Logs:** Takes you directly to GitHub Actions run  
* **Download Artifacts:** Download log files  
* **GCP Console:** Opens relevant Cloud Run job in GCP

### **6.4 Acting on Notifications**

**Standard Operating Procedure:**

1. **For SUCCESS (✅):**  
   * File the email for record  
   * No action needed  
2. **For SUCCESS WITH ERRORS (⚠️):**  
   * Review error details within 1 hour  
   * Classify as critical or non-critical  
   * If non-critical: Document and monitor  
   * If critical: Follow failure procedure  
3. **For FAILURE (❌):**  
   * Acknowledge within 15 minutes  
   * Start investigation immediately  
   * Fix within SLA window (check config for SLA)  
   * Inform team lead if cannot resolve in time  
   * Document root cause and resolution

---

## **7\. TROUBLESHOOTING GUIDELINES**

This section covers common problems you might encounter and their solutions.

### **7.1 Scheduled Workflow Not Running**

**Problem Description:** You notice that a level which should have run at its scheduled time did not run.

**Diagnostic Steps:**

1. **Check if GitHub Actions is enabled:**  
   * Go to Settings → Actions → General  
   * Verify "Allow all actions and reusable workflows" is selected  
2. **Check repository activity:**  
   * GitHub disables scheduled workflows after 60 days of repository inactivity  
   * Make a commit to re-activate  
3. **Check workflow file syntax:**  
   * Ensure cron syntax is correct  
   * Use [https://crontab.guru](https://crontab.guru) to validate  
4. **Check GitHub Status:**  
   * Visit [https://www.githubstatus.com](https://www.githubstatus.com)  
   * See if there are any ongoing incidents

**Solution Steps:**

If repository is inactive:

bash  
*\# Make an empty commit to re-activate*  
git commit \--allow-empty \-m "Re-activate scheduled workflows"

git push

If syntax error:

yaml  
*\# Correct format:*  
on:  
  schedule:  
    \- cron: '0 0 \* \* \*'    *\# ✓ Correct*  
    *\# Not: cron: 0 0 \* \* \*  \# ✗ Wrong (missing quotes)*  
\`\`\`

*\#\#\# 7.2 GCP Authentication Failure*

\*\*Problem Description:\*\*  
Workflow fails at the GCP authentication step with error message like:  
\`\`\`

Error: google-github-actions/auth failed with: failed to generate Google Cloud credentials

**Diagnostic Steps:**

1. **Verify secret exists:**  
   * Go to Settings → Secrets → Actions  
   * Check if `GCP_SA_KEY` is listed  
2. **Verify secret content:**  
   * Secret should be complete JSON content  
   * Should start with `{"type": "service_account"`  
   * Should not have extra quotes or formatting  
3. **Verify service account permissions:**  
   * Go to GCP Console → IAM & Admin → Service Accounts  
   * Find your service account  
   * Check if it has required roles

**Solution Steps:**

To fix secret:

1. Go to GCP Console  
2. Navigate to IAM & Admin → Service Accounts  
3. Find the service account  
4. Click on three dots → Manage Keys  
5. Create new JSON key  
6. Download the JSON file  
7. Go to GitHub → Settings → Secrets  
8. Edit `GCP_SA_KEY` secret  
9. Copy entire content of JSON file  
10. Paste and save

To fix permissions:

1. Go to GCP Console → IAM & Admin → IAM  
2. Find your service account  
3. Click Edit  
4. Add these roles if missing:  
   * Cloud Run Admin  
   * Logs Viewer  
   * Service Account User  
5. Save

### **7.3 Email Not Being Received**

**Problem Description:** Workflow completes but you don't receive email notification.

**Diagnostic Steps:**

1. **Check spam folder**  
   * Search for sender email  
   * If found in spam, mark as "Not Spam"  
2. **Check GitHub workflow logs**  
   * Go to the workflow run  
   * Click on "Send Email Notification" step  
   * Look for error messages  
3. **Verify SMTP secrets**  
   * Settings → Secrets → Actions  
   * Ensure all SMTP secrets exist:  
     * SMTP\_SERVER  
     * SMTP\_PORT  
     * SMTP\_USERNAME  
     * SMTP\_PASSWORD  
4. **Test SMTP credentials**  
   * Try sending email manually using same credentials  
   * Use a mail client or command line tool

**Solution Steps:**

If credentials are wrong:

1. Obtain correct SMTP details from IT team  
2. Update all SMTP secrets in GitHub  
3. Re-run workflow to test

If firewall is blocking:

1. Contact IT team  
2. Request to whitelist:  
   * Source: GitHub Actions IP ranges  
   * Destination: Your SMTP server  
   * Port: 587 or 465

If email is being filtered:

1. Add sender to safe senders list  
2. Create email filter rule to mark as important  
3. Contact email admin to whitelist

### **7.4 Logs Not Being Collected**

**Problem Description:** Workflow runs successfully but no logs appear in artifacts, or error count shows 0 when you know there were errors.

**Diagnostic Steps:**

1. **Check if Cloud Logging API is enabled:**

bash

  gcloud services list \--enabled | grep logging

2. **Check service account permissions:**  
   * Verify service account has `roles/logging.viewer`  
3. **Check log freshness parameter:**  
   * Current setting: `--freshness=30m`  
   * If job runs longer, this might need adjustment  
4. **Check GCP log retention:**  
   * Default is 30 days  
   * Ensure logs haven't expired

**Solution Steps:**

If API not enabled:

bash

gcloud services enable logging.googleapis.com

If permissions missing:

1. Go to GCP Console → IAM & Admin → IAM  
2. Find service account  
3. Add role: "Logs Viewer"

If freshness too short:

1. Edit `_level-executor.yml`  
2. Find the log collection step  
3. Change:

yaml

  \--freshness=30m

To:

yaml

  \--freshness=2h

### **7.5 Configuration Changes Not Taking Effect**

**Problem Description:** You modified a configuration file but the workflow still seems to use old values.

**Diagnostic Steps:**

1. **Verify commit and push:**

bash  
  git status              *\# Should show "nothing to commit"*  
   git log \-1              *\# Should show your commit*

   git push \--dry-run      *\# Should show "Everything up-to-date"*

2. **Check file path:**  
   * File must be exactly: `.github/config/levels/level-N.yml`  
   * Check for typos, extra spaces, or wrong directory  
3. **Verify workflow references correct file:**  
   * Open workflow file (e.g., `level-00-midnight.yml`)  
   * Check `config_file` parameter  
   * Ensure it points to correct path  
4. **Check for YAML syntax errors:**  
   * Use online YAML validator  
   * Ensure proper indentation

**Solution Steps:**

If not committed:

bash  
git add .github/config/levels/level-N.yml  
git commit \-m "Update configuration for level N"

git push

If path is wrong:

1. Rename/move file to correct location  
2. Update workflow file to reference correct path  
3. Commit and push

If syntax error:

1. Use YAML validator to find error  
2. Fix the syntax  
3. Commit and push

---

## **8\. FREQUENTLY ASKED QUESTIONS**

### **8.1 General Questions**

**Q1: What happens to our existing Jenkins setup?**

**A:** Jenkins will be decommissioned after successful migration validation. During the transition period (approximately 30 days), both systems will run in parallel for comparison and verification purposes. Once we are confident that GitHub Actions is working correctly, Jenkins will be shut down permanently.

---

**Q2: Do we need to change our cloud\_runexecute.sh script?**

**A:** No, absolutely not. The script remains exactly as it is. We are only changing how and when it gets triggered. All your existing logic, error handling, and processing remains unchanged.

---

**Q3: Can we run levels in different order?**

**A:** Yes, each level is completely independent. You can use the manual trigger to run any level at any time. The scheduled runs follow the defined timing, but manual runs have no such restrictions.

---

**Q4: What if we need to temporarily disable a level?**

**A:** You have multiple options:

Option 1: Disable the workflow

1. Go to Actions → Select the level workflow  
2. Click on three dots (⋮) → Disable workflow

Option 2: Comment out the schedule

yaml  
*\# on:*  
*\#   schedule:*

*\#     \- cron: '0 0 \* \* \*'*

Option 3: Add a condition to skip

yaml  
jobs:  
  execute:

    if: false  *\# This will skip execution*

Remember to re-enable when maintenance is complete\!

---

**Q5: How can we see execution history?**

**A:** Multiple ways:

1. **GitHub Actions UI:**  
   * Go to Actions tab  
   * Shows last 90 days by default  
   * Can filter by status, level, date range  
2. **Artifacts:**  
   * Stored for 30 days  
   * Can be downloaded anytime within retention period  
3. **GCP Cloud Logging:**  
   * Default retention 30 days  
   * Can be extended if needed  
4. **Email Archive:**  
   * All notifications are sent to cse@cvshealth.com  
   * Can search mailbox for historical records

---

### **8.2 Configuration Questions**

**Q6: Can we use different branches for different levels?**

**A:** Yes, absolutely\! Each level configuration file has independent `branch_name` parameter. For example:

yaml  
*\# Level 0 can use main*  
branch\_name: main

*\# Level 3 can use develop*  
branch\_name: develop

*\# Level 7 can use feature branch*

branch\_name: feature/new-algorithm

This is useful for gradual rollouts and A/B testing.

---

**Q7: Can we change schedule timing for a level?**

**A:** Yes, but you need to modify the workflow file (not config file). For example:

yaml  
*\# In .github/workflows/level-03-6am.yml*  
on:  
  schedule:

    \- cron: '0 6 \* \* \*'  *\# Change this line*

Cron syntax guide:

* `0 6 * * *` \= 6:00 AM every day  
* `0 6 * * 1` \= 6:00 AM every Monday  
* `0 6 1 * *` \= 6:00 AM on 1st of every month  
* `0 */2 * * *` \= Every 2 hours

Use [https://crontab.guru](https://crontab.guru) to generate cron expressions.

---

**Q8: How do we add a new level (e.g., Level 11)?**

**A:** Follow these steps:

1. **Create configuration file:**

bash

  cp .github/config/levels/level-10.yml .github/config/levels/level-11.yml

2. **Edit the new config:**

yaml  
  level: 11  
   description: "Level 11 \- Description here"

   *\# ... update other parameters*

3. **Create workflow file:**

bash

  cp .github/workflows/level-10-8pm.yml .github/workflows/level-11-10pm.yml

4. **Edit the workflow:**

yaml  
  name: Level 11 \- Daily 10 PM  
     
   on:  
     schedule:  
       \- cron: '0 22 \* \* \*'  *\# 10 PM*  
     
   jobs:  
     execute-level-11:  
       uses: ./.github/workflows/\_level-executor.yml  
       with:  
         level: 11

         config\_file: '.github/config/levels/level-11.yml'

5. **Update manual trigger:** Add '11' to the options list in `manual-any-level.yml`  
6. **Commit and push**  
7. **Test with manual trigger first**

---

**Q9: Can we have different timeout values for different jobs within same level?**

**A:** The current design has a single timeout per level. If you need per-job timeouts, you would need to:

1. Handle it in your `cloud_runexecute.sh` script, or  
2. Modify the template to support job-level configuration (more complex)

For most cases, level-wide timeout should be sufficient.

---

### **8.3 Execution Questions**

**Q10: How do we re-run a failed level?**

**A:** Two methods:

**Method 1: Re-run from GitHub UI**

1. Go to Actions tab  
2. Click on the failed run  
3. Click "Re-run jobs" dropdown (top right)  
4. Select "Re-run failed jobs" or "Re-run all jobs"

**Method 2: Manual trigger**

1. Go to Actions → Manual Level Execution  
2. Select the level number  
3. Run workflow

Method 1 is preferred as it uses exact same parameters and code version.

---

**Q11: Can we run multiple levels simultaneously?**

**A:** Yes, all levels are independent. They can run at the same time without interfering with each other. However, be mindful of:

1. **GCP quotas:** Ensure you don't exceed Cloud Run concurrent execution limits  
2. **Resource contention:** If levels access same data, there might be race conditions  
3. **Email flood:** Running all 11 levels at once will send 11 emails

For testing purposes, stagger the executions by few minutes.

---

**Q12: What if a level execution is taking too long?**

**A:** You have several options:

**Option 1: Increase timeout**

yaml  
*\# In config file*

timeout: 3600  *\# Increase to 1 hour*

**Option 2: Cancel and investigate**

1. Go to Actions → Running workflow  
2. Click "Cancel workflow"  
3. Check logs to see where it's stuck  
4. Fix the issue  
5. Re-run

**Option 3: Monitor and wait**

* Some jobs legitimately take long time  
* If progress is being made, let it run  
* Set alerts if execution exceeds expected SLA

---

**Q13: How do we handle urgent/emergency runs?**

**A:** For emergency situations:

1. **Use manual trigger immediately** \- don't wait for schedule  
2. **Override branch if needed:**  
   * If fix is in a feature branch  
   * Use branch override parameter  
   * No need to merge to main first  
3. **Inform team:**  
   * Post in Slack channel  
   * Send email to stakeholders  
   * Document in incident report  
4. **Monitor closely:**  
   * Watch the execution in real-time  
   * Keep GCP console open  
   * Be ready to intervene if needed

---

### **8.4 Troubleshooting Questions**

**Q14: Error says "Cloud Run job not found" \- what to do?**

**A:** This means the job name in config doesn't match GCP.

**Diagnosis:**

bash  
*\# List all Cloud Run jobs*  
gcloud run jobs list \--region\=us-central1  
\`\`\`

\*\*Solution:\*\*  
1. Find the exact job name from above list  
2. Update config file with correct name  
3. Ensure spelling and case match exactly  
4. Commit and re-run

\*\*Prevention:\*\*  
\- Always verify job exists in GCP before creating config  
\- Use copy-paste instead of typing job names  
\- Maintain a central registry of all Cloud Run jobs

\---

\*\*Q15: Getting "permission denied" errors \- how to fix?\*\*

\*\*A:\*\* Permission errors usually mean service account lacks required role.

\*\*Common scenarios:\*\*

1. \*\*Cannot execute Cloud Run job:\*\*  
   \- Need: \`roles/run.admin\`

2. \*\*Cannot read logs:\*\*  
   \- Need: \`roles/logging.viewer\`

3. \*\*Cannot access Secret Manager:\*\*  
   \- Need: \`roles/secretmanager.secretAccessor\`

\*\*How to fix:\*\*  
1. Identify which resource is giving permission error  
2. Go to GCP Console → IAM & Admin → IAM  
3. Find your service account  
4. Click Edit  
5. Add the required role  
6. Save  
7. Wait 1\-2 minutes for propagation  
8. Re-run the workflow

\---

\*\*Q16: Email shows errors but I don't understand them \- what should I do?\*\*

\*\*A:\*\* Don't panic\! Follow this process:

1. \*\*Download full logs:\*\*  
   \- Go to workflow run  
   \- Download artifacts  
   \- Extract and open execution.log

2. \*\*Search for the first error:\*\*  
   \- Errors often cascade  
   \- First error is usually the root cause  
   \- Look for timestamp of first error

3. \*\*Understand the context:\*\*  
   \- Read 10\-20 lines before and after the error  
   \- This gives context about what was happening

4. \*\*Google the error message:\*\*  
   \- Copy exact error message  
   \- Search Google or Stack Overflow  
   \- Often you'll find similar issues and solutions

5\. \*\*Ask for help:\*\*  
   \- If still stuck, post in \#devops-support Slack channel  
   \- Include:  
     \- Level number  
     \- Error message  
     \- What you've tried  
     \- Link to workflow run

6. \*\*Escalate if urgent:\*\*  
   \- Tag @devops-oncall in Slack  
   \- Call on-call engineer if business-critical  
   \- Create P1 incident ticket

\---

*\#\# 9\. ANNEXURES*

*\#\#\# Annexure A: Cron Schedule Reference*

Quick reference for cron expressions:  
\`\`\`  
\* \* \* \* \*  
│ │ │ │ │  
│ │ │ │ └─── Day of week (0\-7, 0 and 7 are Sunday)  
│ │ │ └───── Month (1\-12)  
│ │ └─────── Day of month (1\-31)  
│ └───────── Hour (0\-23)

└─────────── Minute (0\-59)

**Common Examples:**

| Expression | Description |
| ----- | ----- |
| `0 0 * * *` | Every day at midnight |
| `0 6 * * *` | Every day at 6 AM |
| `0 */2 * * *` | Every 2 hours |
| `0 0 * * 0` | Every Sunday at midnight |
| `0 0 1 * *` | First day of every month |

### **Annexure B: Useful Commands**

**Git Commands:**

bash  
*\# Check status*  
git status

*\# View recent commits*  
git log \--oneline \-10

*\# See what changed*  
git diff

*\# Discard local changes*  
git checkout \-- filename

*\# Pull latest changes*  
git pull origin main

*\# Create new branch*  
git checkout \-b feature-name

*\# Switch branch*

git checkout branch-name

**GCloud Commands:**

bash  
*\# List Cloud Run jobs*  
gcloud run jobs list \--region\=us-central1

*\# Describe a job*  
gcloud run jobs describe JOB\_NAME \--region\=us-central1

*\# View recent executions*  
gcloud run jobs executions list \--job\=JOB\_NAME \--region\=us-central1

*\# View logs*  
gcloud logging read "resource.type=cloud\_run\_job" \--limit\=100

*\# Check service account*

gcloud iam service-accounts list

**YAML Validation:**

bash  
*\# Using Python*  
python \-c "import yaml; yaml.safe\_load(open('file.yml'))"

*\# Using yq*  
yq eval . file.yml

*\# Online validators*  
*\# https://www.yamllint.com/*

*\# https://codebeautify.org/yaml-validator*

### **Annexure C: Contact Information**

**Primary Contacts:**

| Role | Name | Email | Slack |
| ----- | ----- | ----- | ----- |
| DevOps Lead | \[Name\] | devops-lead@cvshealth.com | @devops-lead |
| Platform Engineer | \[Name\] | platform-eng@cvshealth.com | @platform-eng |
| Data Engineering Lead | \[Name\] | data-lead@cvshealth.com | @data-lead |

**Support Channels:**

* **Slack:** \#devops-support  
* **Email:** devops-team@cvshealth.com  
* **On-call:** PagerDuty rotation ([https://pagerduty.cvshealth.com](https://pagerduty.cvshealth.com))

**Escalation Matrix:**

| Severity | Response Time | Contact |
| ----- | ----- | ----- |
| P1 (Critical) | 15 minutes | On-call engineer \+ Manager |
| P2 (High) | 2 hours | DevOps team lead |
| P3 (Medium) | 1 day | Slack channel |
| P4 (Low) | 3 days | Email |

### **Annexure D: Glossary**

**Technical Terms:**

* **Artifact:** Files generated during workflow execution (logs, reports, etc.) that are stored for later download  
* **Cron:** A time-based job scheduler in Unix-like systems, we use cron syntax for scheduling  
* **GCP:** Google Cloud Platform  
* **IAM:** Identity and Access Management  
* **Pull Request (PR):** A request to merge code changes from one branch to another  
* **Service Account:** A special type of account used by applications to make API requests  
* **SMTP:** Simple Mail Transfer Protocol, used for sending emails  
* **UTC:** Coordinated Universal Time, standard time zone for scheduling  
* **YAML:** Yet Another Markup Language / YAML Ain't Markup Language, a human-readable data serialization format  
* **Workflow:** A configurable automated process made up of one or more jobs

**Project-Specific Terms:**

* **Level:** One of 11 independent pipeline stages (Level 0 through Level 10\)  
* **cloud\_runexecute.sh:** Our existing shell script that executes Cloud Run jobs  
* **Static Config:** Configuration approach where each level has its own separate file  
* **Reusable Workflow:** Template workflow that can be called by other workflows

---

## **10\. CONCLUSION**

This migration to GitHub Actions represents a significant improvement in our CI/CD infrastructure. We have:

✅ Eliminated Jenkins maintenance burden ✅ Implemented automated error detection ✅ Added proactive email notifications ✅ Improved visibility and audit trail ✅ Maintained all existing functionality

The static configuration approach we have adopted ensures that the system remains easy to understand and maintain for all team members.

**Next Steps for Team Members:**

1. Read this document thoroughly  
2. Attend the hands-on training session (will be scheduled)  
3. Test manual trigger for at least one level  
4. Bookmark this document for future reference  
5. Join \#devops-support Slack channel  
6. Provide feedback and suggestions

**Remember:**

* Don't hesitate to ask questions  
* Use manual trigger for testing before scheduled runs  
* Always create PR for configuration changes  
* Monitor email notifications regularly  
* Report any issues immediately

Thank you for your cooperation during this migration. Together, we will make this transition smooth and successful\!

---

**Document Version:** 1.0 **Last Updated:** 20th January, 2026 **Prepared By:** DevOps Team **Approved By:** \[Manager Name\]

---

**Feedback and Suggestions:**

If you have any suggestions to improve this document or the system, please send email to: devops-team@cvshealth.com or post in \#devops-support Slack channel.

Your feedback is valuable and will help us make continuous improvements.

---

This Google Docs version is written in Indian English style which typically features:

* More formal tone  
* Detailed explanations  
* Step-by-step procedures  
* Extensive use of tables  
* Clear section numbering  
* Polite and respectful language  
* Thorough coverage of edge cases  
* Emphasis on procedures and proper channels

You can copy this entire text into a Google Doc and it will maintain proper formatting with headings, tables, code blocks, etc.

