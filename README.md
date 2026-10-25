# Invoice Payment Reminder

A daily n8n workflow for a small business that sends payment reminders. Each morning it checks the invoices table for unpaid invoices. An invoice due in the next three days gets an "upcoming" reminder, and an invoice past its due date gets an "overdue" reminder. Each reminder is sent once per stage, so the same customer is not emailed every day.

## Architecture

```
Schedule Trigger (daily, 09:00)
   → Get Unpaid Invoices (Supabase: invoices, status = unpaid)
   → Decide Reminder Stage (days left, stage: upcoming or overdue)
        └── only invoices with a new stage continue
   → Send Reminder (Gmail)
        ├── success → Mark Reminder Sent (Supabase: last_reminder_stage, last_reminder_at)
        └── error   → left unmarked, retried the next day
```

## Features

* Runs every day at 09:00 (Karachi time)
* Reads all unpaid invoices, not just the first page
* Sends an "upcoming" reminder when an invoice is due in three days or less, including the day it is due
* Sends an "overdue" reminder when the due date has passed
* Each stage is sent once. The stored `last_reminder_stage` stops repeat emails
* One failed email does not stop the others. The failed invoice stays unmarked and is tried again the next day
* Records when each reminder was sent

## Tech Stack

* n8n 2.x (workflow automation, scheduling)
* Supabase (invoices table)
* Gmail API (reminder emails)

## Setup

1. Create a Supabase table named `invoices` with these columns:

```
id | customer_name | customer_email | amount | due_date | status | last_reminder_stage | last_reminder_at
```

   Use `due_date` in `YYYY-MM-DD` format. Set `status` to `unpaid` for open invoices. Leave `last_reminder_stage` empty for new invoices.

2. Import `invoice_payment_reminder.json` into n8n.
3. Add credentials: Supabase and Gmail OAuth2. Assign them to the matching nodes.
4. Activate the workflow.
5. Test with an invoice due in two days, then check the email and the `last_reminder_stage` value (it should be `upcoming`).

## Reminder Stages

| Days until due | Stage | Email |
|---|---|---|
| 3 or fewer, not yet due | upcoming | "Upcoming payment reminder" |
| Due date passed | overdue | "Payment overdue: Invoice reminder" |

An invoice moves from `upcoming` to `overdue` automatically when its due date passes. Each stage is sent once.

## Sample Row

```
customer_name:  Ali Raza
customer_email: ali.test@example.com
amount:         25000
due_date:       2026-11-07
status:         unpaid
```

If today is 5 November 2026, this invoice is two days from due and gets the upcoming reminder.

## Known Limits

* This workflow does not change an invoice's status when it is paid. Set `status` to `paid` in your invoices table when payment is received. Otherwise reminders will continue.
* The amount is sent as a plain number with no currency symbol. Add the currency to the email text if you need it.
* Overdue invoices get one overdue reminder only. If you want repeat reminders, add a schedule (for example every seven days) in the Decide Reminder Stage node.

## Project Notes

* Built as part of my AI Automation portfolio.
* No AI model is used. The stage is decided in code from the due date, so each reminder is predictable.
