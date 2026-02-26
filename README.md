# aws-ses

Below is the **same setup you pasted**, but **sequenced + cleaned**, with **copy-paste code** and **practical notes** (idempotency, extra SES event types, DB indexes, supervisor).

---

## 0) Architecture (what happens)

1. Laravel sends email via **SES SMTP**
2. You add headers:

   * **Configuration Set** → turns on event publishing
   * **Message Tags** → lets you link events to campaign/tenant
3. SES publishes events → **SNS** → **SQS**
4. Laravel polls SQS, unwraps SNS JSON, stores SES event → **PostgreSQL**

---

## A) Laravel 11 — Add SES headers in every Mailable (SMTP)

### ✅ CampaignMail.php (recommended: keep `build()` but ensure headers apply)

```php
<?php

namespace App\Mail;

use Illuminate\Bus\Queueable;
use Illuminate\Mail\Mailable;
use Illuminate\Queue\SerializesModels;

class CampaignMail extends Mailable
{
    use Queueable, SerializesModels;

    public function __construct(
        public string $campaignId,
        public string $tenantId,
    ) {}

    public function build()
    {
        return $this
            ->subject('Campaign Subject')
            ->view('emails.campaign')
            ->withSymfonyMessage(function ($message) {
                // 1) Enable SES event publishing (SMTP) via Configuration Set
                $message->getHeaders()->addTextHeader(
                    'X-SES-CONFIGURATION-SET',
                    'campaign-events'
                );

                // 2) Tags (become mail.tags in event payload)
                $message->getHeaders()->addTextHeader(
                    'X-SES-MESSAGE-TAGS',
                    'campaign_id='.$this->campaignId.',tenant_id='.$this->tenantId
                );
            });
    }
}
```

Send:

```php
Mail::to($user->email)->send(new \App\Mail\CampaignMail($campaignId, $tenantId));
```

**Notes**

* SES will return tags in events under: `mail.tags.campaign_id[0]`, etc.
* Keep tag keys simple (`campaign_id`, `tenant_id`). Avoid spaces.

---

## B) AWS — Confirm SQS messages are arriving

After sending a test email:

* SQS queue: `ses-events-queue`
* “Messages available” should increase
* Body will be SNS wrapper JSON, like:

```json
{
  "Type": "Notification",
  "Message": "{ ... SES EVENT JSON ... }"
}
```

You’ll parse `Body.Message` (string) into SES JSON.

---

## C) Laravel — SQS consumer via Artisan command (poll + store)

### 1) Install AWS SDK

```bash
composer require aws/aws-sdk-php
```

### 2) .env

```env
AWS_ACCESS_KEY_ID=xxxx
AWS_SECRET_ACCESS_KEY=xxxx
AWS_DEFAULT_REGION=us-east-1
SES_EVENTS_SQS_URL=https://sqs.us-east-1.amazonaws.com/682033496260/ses-events-queue
```

### 3) Command: `app/Console/Commands/SesConsumeEvents.php`

This is your code, but with **important hardening**:

* handles **more event types**
* **idempotent insert** via unique key (prevents duplicates on retries)
* uses **tenant_id in upsert key** (multi-tenant safe)
* stores `event_id` (SQS/SNS doesn’t provide one consistently, so we create one)

```php
<?php

namespace App\Console\Commands;

use Aws\Sqs\SqsClient;
use Illuminate\Console\Command;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Str;

class SesConsumeEvents extends Command
{
    protected $signature = 'ses:consume-events {--once : Process one batch and exit}';
    protected $description = 'Consume SES event messages from SQS (SNS->SQS) and store into PostgreSQL';

    public function handle(): int
    {
        $sqs = new SqsClient([
            'version' => 'latest',
            'region'  => env('AWS_DEFAULT_REGION', 'us-east-1'),
            'credentials' => [
                'key'    => env('AWS_ACCESS_KEY_ID'),
                'secret' => env('AWS_SECRET_ACCESS_KEY'),
            ],
        ]);

        $queueUrl = env('SES_EVENTS_SQS_URL');
        if (!$queueUrl) {
            $this->error('Missing SES_EVENTS_SQS_URL in .env');
            return 1;
        }

        do {
            $resp = $sqs->receiveMessage([
                'QueueUrl'            => $queueUrl,
                'MaxNumberOfMessages' => 10,
                'WaitTimeSeconds'     => 20,
                'VisibilityTimeout'   => 60,
            ]);

            $messages = $resp->get('Messages') ?? [];
            if (count($messages) === 0) {
                if ($this->option('once')) break;
                continue;
            }

            foreach ($messages as $m) {
                $receiptHandle = $m['ReceiptHandle'];
                $body = json_decode($m['Body'] ?? '', true);

                try {
                    if (!is_array($body) || !isset($body['Message'])) {
                        throw new \RuntimeException('Not an SNS notification');
                    }

                    $ses = json_decode($body['Message'], true);
                    if (!is_array($ses)) {
                        throw new \RuntimeException('Invalid SES JSON inside SNS Message');
                    }

                    $eventType = $ses['eventType'] ?? null; // Send/Delivery/Bounce/Complaint/Open/Click/Reject...
                    $messageId = $ses['mail']['messageId'] ?? null;
                    $recipients = $ses['mail']['destination'] ?? [];

                    $tags = $ses['mail']['tags'] ?? [];
                    $campaignId = $tags['campaign_id'][0] ?? null;
                    $tenantId   = $tags['tenant_id'][0] ?? null;

                    // best-effort timestamp
                    $eventTime =
                        $ses['delivery']['timestamp'] ??
                        $ses['bounce']['timestamp'] ??
                        $ses['complaint']['timestamp'] ??
                        $ses['open']['timestamp'] ??
                        $ses['click']['timestamp'] ??
                        $ses['reject']['timestamp'] ??
                        $ses['mail']['timestamp'] ??
                        now()->toIso8601String();

                    if (!$messageId || !$eventType || empty($recipients)) {
                        throw new \RuntimeException('Missing messageId/eventType/destination');
                    }

                    DB::transaction(function () use ($ses, $eventType, $messageId, $recipients, $eventTime, $campaignId, $tenantId) {
                        foreach ($recipients as $to) {

                            // Deterministic-ish event_id to dedupe retries:
                            // messageId + recipient + eventType + eventTime
                            $eventId = hash('sha256', $messageId.'|'.$to.'|'.$eventType.'|'.$eventTime);

                            // 1) Audit log (idempotent)
                            DB::table('email_events')->updateOrInsert(
                                ['event_id' => $eventId],
                                [
                                    'message_id'      => $messageId,
                                    'event_type'      => $eventType,
                                    'recipient_email' => $to,
                                    'event_time'      => $eventTime,
                                    'campaign_id'     => $campaignId,
                                    'tenant_id'       => $tenantId,
                                    'payload'         => json_encode($ses),
                                    'created_at'      => now(),
                                ]
                            );

                            // 2) Recipient current status (idempotent)
                            $update = [
                                'tenant_id'   => $tenantId,
                                'message_id'  => $messageId,
                                'campaign_id' => $campaignId,
                                'status'      => strtolower((string) $eventType),
                                'updated_at'  => now(),
                            ];

                            if ($eventType === 'Send')     $update['sent_at']      = $eventTime;
                            if ($eventType === 'Delivery') $update['delivered_at'] = $eventTime;

                            if ($eventType === 'Bounce') {
                                $update['bounced_at'] = $eventTime;
                                $update['bounce_type'] = $ses['bounce']['bounceType'] ?? null;
                                $update['bounce_subtype'] = $ses['bounce']['bounceSubType'] ?? null;
                                $update['bounce_diagnostic'] =
                                    $ses['bounce']['bouncedRecipients'][0]['diagnosticCode'] ?? null;
                            }

                            if ($eventType === 'Complaint') {
                                $update['complaint_at'] = $eventTime;
                            }

                            // Multi-tenant safe upsert key
                            DB::table('email_recipients')->updateOrInsert(
                                ['tenant_id' => $tenantId, 'campaign_id' => $campaignId, 'recipient_email' => $to],
                                $update
                            );
                        }
                    });

                    // delete only after commit
                    $sqs->deleteMessage([
                        'QueueUrl'      => $queueUrl,
                        'ReceiptHandle' => $receiptHandle,
                    ]);
                } catch (\Throwable $e) {
                    $this->error('Failed: '.$e->getMessage());
                    // Do not delete => retry / DLQ
                }
            }

            if ($this->option('once')) break;

        } while (true);

        return 0;
    }
}
```

Run:

```bash
php artisan ses:consume-events
```

---

## D) PostgreSQL — Exact Laravel migrations (copy-paste)

### 1) `email_events` (append-only audit log)

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void
    {
        Schema::create('email_events', function (Blueprint $table) {
            $table->bigIncrements('id');

            // idempotency key (dedupe retries)
            $table->string('event_id', 64)->unique();

            $table->string('message_id')->index();
            $table->string('event_type')->index();

            $table->string('recipient_email')->index();

            $table->timestampTz('event_time')->index();

            $table->string('campaign_id')->nullable()->index();
            $table->string('tenant_id')->nullable()->index();

            $table->jsonb('payload');

            $table->timestampTz('created_at')->useCurrent();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('email_events');
    }
};
```

### 2) `email_recipients` (current status per recipient)

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void
    {
        Schema::create('email_recipients', function (Blueprint $table) {
            $table->bigIncrements('id');

            $table->string('tenant_id')->nullable();
            $table->string('campaign_id')->nullable();

            $table->string('recipient_email');

            $table->string('message_id')->nullable()->index();
            $table->string('status')->nullable()->index(); // send/delivery/bounce/complaint/...

            $table->timestampTz('sent_at')->nullable();
            $table->timestampTz('delivered_at')->nullable();
            $table->timestampTz('bounced_at')->nullable();
            $table->timestampTz('complaint_at')->nullable();

            $table->string('bounce_type')->nullable();
            $table->string('bounce_subtype')->nullable();
            $table->text('bounce_diagnostic')->nullable();

            $table->timestampTz('created_at')->useCurrent();
            $table->timestampTz('updated_at')->nullable();

            // one row per (tenant,campaign,recipient)
            $table->unique(['tenant_id', 'campaign_id', 'recipient_email']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('email_recipients');
    }
};
```

---

## E) Operational notes (stuff that saves you pain)

### 1) SES event types you might see (beyond the basics)

Even if you only “care” about Delivery/Bounce/Complaint, SES can emit:

* `Send`, `Delivery`, `Bounce`, `Complaint`
* `Reject`, `RenderingFailure`
* `Open`, `Click` (if you enable engagement tracking where applicable)

Your code now won’t break if these appear.

### 2) Why idempotency matters

SQS can redeliver messages (visibility timeout, consumer crash, retries).
That’s why `email_events.event_id` is **unique**, and we `updateOrInsert`.

### 3) Recommended SQS config

* Enable **DLQ** with `maxReceiveCount` (e.g., 5–10)
* Visibility timeout should exceed worst-case DB processing time

### 4) Supervisor example (simple)

Run command forever:

```ini
[program:ses_consumer]
command=php /var/www/html/artisan ses:consume-events
autostart=true
autorestart=true
user=www-data
redirect_stderr=true
stdout_logfile=/var/log/ses_consumer.log
```

---

If you paste **one actual inner SES event JSON** you see inside `Body.Message`, I can:

* confirm exactly where your tags are appearing,
* recommend the best **unique key**,
* and map additional fields (SMTP response, bounce recipient details, remote MTA IP, etc.) into columns if needed.
