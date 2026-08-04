# Submission Analytics Pipeline
## Automated Dual Source Ingestion and BI Reporting for OpenTicket and OpenInvoice Data

### Tech Stack & Tools
* **Infrastructure as Code & DevOps:** AWS SAM (YAML), Docker (`sam build --use-container`), SAM CLI, AWS VPC
* **Core AWS Services:** AWS Lambda (Python 3.13), Amazon S3, Amazon SQS, AWS EventBridge, Amazon CloudWatch, Amazon SNS, AWS SSM Parameter Store
* **Database & Analytics:** Amazon RDS (MySQL), AWS Glue, Amazon Athena, Amazon QuickSight
* **Languages & Data Libraries:** Python 3.13, Pandas, SQLAlchemy, PyMySQL, AWSWrangler, PyArrow, Requests
* **Upstream Integrations:** OpenTicket API (mTLS), OpenInvoice (Power Automate to S3 XLSX exports)

---

## The Business Story: Unlocking Hidden Growth Opportunities

At EMI (EnterMyInvoice), we automate field service ticket and invoice submissions for oil and gas suppliers delivering to corporate buyers. Submissions route through two main platforms owned by Enverus: OpenTicket and OpenInvoice. 

In standard field operations, suppliers submit service tickets into OpenTicket. Once a ticket is approved, it can be flipped into an official invoice inside OpenInvoice. Suppliers have two ways to do this. They can log into the web portals and manually type in entries and flip tickets by hand, or they can use EMI's automated B2B service to process tickets and automatically flip approved tickets into OpenInvoice invoices.

Historically, our internal system only allowed my manager to see metrics for transactions processed through EMI and stored in our own database. He could analyze our own volume and success rates, but he had zero visibility into broader supplier activity happening directly on the manual web portals. 

He knew many suppliers still submitted or flipped tickets manually due to old habits, buyer preferences, or doubts about automated accuracy. However, because tickets often get disputed or canceled during manual entry, suppliers were wasting hours of payroll and delaying their own payments.

My manager realized we were missing major sales opportunities. He needed a way to track total manual portal volume across all suppliers, identify which specific supplier employees were manually typing in entries, and prove that EMI's automated service cuts approval lag times and dispute rates. He asked me to build a cross platform analytics system to answer these questions so our sales team could pitch B2B automation to existing clients.

---

## Business Solution: Executive QuickSight Dashboard

To solve this, I built a daily automated reporting system connected directly to an interactive 5 page Amazon QuickSight dashboard. The dashboard gives executive leadership and sales teams complete visibility into manual versus automated submission habits.

[![Watch Demo](https://img.shields.io/badge/▶️_Watch_the_B2B_QuickSight_Dashboard_Video_Demo-B1F6FC?style=for-the-badge)](https://youtu.be/5PW44m2eUOg)

<p align="center">
  <a href="https://youtu.be/5PW44m2eUOg" target="_blank">
    <img src="https://img.youtube.com/vi/5PW44m2eUOg/maxresdefault.jpg" alt="B2B QuickSight Dashboard Demo" width="100%" />
  </a>
</p>

### The 5 Dashboard Pages
* **Page 1: Ticket Analytics:** Compares ticket volumes, total dollar values, approval durations, and status breakdowns (approved, disputed, canceled) between EMI automated submissions and manual portal entries.

* **Page 2: Invoice Analytics:** Tracks financial totals, invoice counts, payment statuses, and macro turnaround lag times.

* **Page 3: Submission Source Comparison:** Side by side bar charts comparing EMI automated volume against client manual web portal entries across supplier and buyer accounts.

* **Page 4: Time & User Behavior Analysis:** Features monthly and quarterly distribution charts breaking down manual ticket creation by individual user names and company emails. This pinpoints exact supplier employees who rely on manual data entry.

* **Page 5: Joined View Table:** A unified spreadsheet view combining ticket and invoice line items via a `FULL OUTER JOIN` for client auditing and Excel exports.

### Business Results
* **Direct Sales Enablement:** Every morning, the business owner and sales team use QuickSight refreshes to review manual entry habits. They use this data to contact suppliers, show them how much payroll is wasted on manual entry, and pitch them on switching to EMI's automated service.

* **Quantified Cash Flow Acceleration:** The analytics proved that EMI's automation significantly cuts approval lag times and lowers dispute rates, showing suppliers that automation helps them get paid faster.

* **User Level Habit Tracking:** Pulling creator names and emails from API payloads allows sales to isolate manual entry habits down to individual supplier employees, providing concrete figures during client calls.

---

## Technical Architecture & Data Flow

Joining data across OpenTicket and OpenInvoice presented a unique technical challenge because access methods differ significantly between the two systems. OpenTicket provides an automated mTLS API returning continuous JSON history objects. OpenInvoice offers no direct API, providing snapshot multi-row Excel reports downloaded via Power Automate into an S3 bucket.

Originally, I built the storage layer using Parquet files on S3 and Athena views. However, as business requirements evolved and my manager requested new tracking fields, managing mismatched schemas in Athena became slow and difficult. I migrated the entire data layer to Amazon RDS running MySQL, allowing us to run fast SQL join views and easily modify table schemas.

<img width="2736" height="3280" alt="Enverus_submission_analytics_pipeline" src="https://github.com/user-attachments/assets/28432562-dc66-4918-84f0-23372b816980" />

### Data Processing Steps

1. **OpenTicket API Pipeline:** An EventBridge rule triggers Lambda 1 (Fanout) every 7 days. It iterates through 25+ suppliers and pushes individual SQS messages into a worker queue. Lambda 2 (Worker) scales up to 5 concurrent instances to fetch tickets using mutual TLS credentials stored in SSM Parameter Store. It standardizes fields using Pandas, extracts user metadata, and upserts records into the MySQL `opentickets` table.

2. **OpenInvoice Excel Pipeline:** Power Automate drops multi-row OpenInvoice Excel reports (`.xlsx`) into an S3 landing bucket (`emi-v3`). S3 upload events trigger SQS messages, invoking the OpenInvoice Worker Lambda. The function converts multi-row event logs into clean single-row records per invoice before upserting into the MySQL `openinvoices` table.

3. **Batch Backfill Pipeline:** Lambda 3 processes S3 JSON files and publishes batch updates, while Lambda 4 reads these payloads to update `invoice_number` values on existing ticket rows using temporary database staging tables.

4. **Downstream BI Reporting:** QuickSight connects directly to MySQL RDS SQL views and refreshes daily to power the 5 page executive dashboard.

---

## Engineering Challenges & Solutions

### 1. Scaling API calls with SQS Fanout
* **The Challenge:** Fetching API data for each supplier takes between 1 to 5 minutes. A single Lambda looping through 25+ suppliers sequentially would hit the 15 minute AWS execution timeout and fail the entire batch.

* **The Solution:** Implemented an SQS Fanout pattern. The controller Lambda finishes in seconds by sending one message per supplier to an SQS queue. SQS triggers separate worker Lambdas in parallel, giving each supplier isolated execution time and independent failure handling.

### 2. Moving from S3 Parquet files to MySQL RDS
* **The Challenge:** Joining mismatched OpenTicket and OpenInvoice structures in Athena was slow. As business requirements changed, adding new fields required backfilling hundreds of Parquet files, updating Glue tables, and rewriting complex queries.

* **The Solution:** Migrated the storage layer to Amazon RDS running MySQL. Adding new fields now only requires a simple `ALTER TABLE ADD COLUMN` command, while joining ticket and invoice datasets is faster and cleaner using standard MySQL views.

### 3. Preventing out of order updates with timestamps
* **The Challenge:** The OpenTicket API returns the full history of a receipt, which gets re-fetched across multiple pipeline runs as statuses update. Standard `INSERT` queries fail on duplicate keys, while blind updates risk overwriting fresh data if payloads arrive out of order.

* **The Solution:** Built an `ON DUPLICATE KEY UPDATE` query that checks the incoming `last_action_timestamp`. Incoming fields are only overwritten if the incoming record's timestamp is newer than what is currently stored in the database.

### 4. Avoiding deadlocks with staging tables
* **The Challenge:** A secondary Lambda processes S3 JSON files to update `invoice_number` values on existing ticket rows. Because multiple workers run in parallel, updating the main database table simultaneously risked deadlocks or race conditions.

* **The Solution:** Implemented a temporary staging table pattern. Each worker Lambda generates a uniquely named temporary staging table (`staging_{request_id}`), writes its local batch updates there, runs an update join against the main table, and drops the staging table inside a `finally` block to guarantee cleanup even if an error occurs.

### 5. Handling mTLS certificates in Lambda
* **The Challenge:** The OpenTicket API requires client certificate authentication. The certificate and private key are stored securely in SSM Parameter Store, but the Python `requests` library requires actual file paths for its SSL configuration, not raw text strings.

* **The Solution:** Programmed the Lambda to fetch certificate strings from SSM on startup and write them as temporary files to the `/tmp` directory. Because files persist across warm Lambda container invocations, the SSM fetch only happens during cold starts to keep execution fast.

### 6. Static IP whitelisting with AWS VPC
* **The Challenge:** The OpenInvoice platform requires all incoming API and file traffic to originate from authorized static IP addresses, which standard public Lambda execution cannot provide.

* **The Solution:** Configured the OpenInvoice worker Lambda inside a custom AWS VPC with dedicated security groups and static outbound Elastic IPs attached to a NAT Gateway, ensuring all traffic matches authorized whitelisted IPs.

### 7. Pipeline alerts with DLQs and CloudWatch
* **The Challenge:** Needed fast, reliable visibility into pipeline failures without swallowing errors inside custom `try/except` blocks.

* **The Solution:** Allowed Lambda exceptions to propagate naturally. SQS automatically retries failed messages up to 3 times before routing them to a Dead-Letter Queue (DLQ). A CloudWatch alarm monitors the DLQ and sends an SNS email alert within 2 minutes of any failure.

### 8. Single stack Infrastructure as Code with SAM
* **The Challenge:** Managing Lambdas, Step Functions, SQS queues, and alarms across separate files or manual setup makes it easy for ARNs to get out of sync and hard to trace in AWS.

* **The Solution:** Defined the entire system, including all Lambdas, queues, and alarms inside one unified SAM `template.yaml` template built with Docker. Because everything belongs to a single CloudFormation stack, deployments are fully automated and easy to reproduce without manual console tweaks.
