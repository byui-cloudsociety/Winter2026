# Week 08 — Chat Wall / Message Board

## Overview

In this project you will build a serverless message board backed entirely by AWS. Users will be able to post short messages and view all messages through a simple web frontend.

**Estimated time:** 60–120 minutes  
**Difficulty:** Beginner–Intermediate

### Architecture

```
Browser (S3 Static Site)
        |
        v
  API Gateway (REST API)
        |
   +---------+----------+
   |                    |
Lambda: postMessage   Lambda: getMessages
        |                    |
        +--------+-----------+
                 |
            DynamoDB Table (messages)
```

### Services Used

| Service            | Purpose                                           |
| ------------------ | ------------------------------------------------- |
| Amazon DynamoDB    | Store messages with timestamps                    |
| AWS Lambda         | Backend logic for posting and retrieving messages |
| Amazon API Gateway | Expose Lambda functions as HTTP endpoints         |
| Amazon S3          | Host the static HTML/JS frontend                  |

### IAM Note

All Lambda functions in this lab will use the **LabRole** that was pre-created in your AWS Academy Learner Lab. Do **not** create a new IAM role — select `LabRole` wherever a role is required.

---

## Part 1 — Create the DynamoDB Table

1. In the AWS Console, navigate to **DynamoDB** → **Tables** → **Create table**.
2. Set the following:
   - **Table name:** `messages`
   - **Partition key:** `messageId` (String)
3. Leave all other settings as default and click **Create table**.
4. Wait until the table status shows **Active** before continuing.

---

## Part 2 — Create the Lambda Functions

You will create two Lambda functions: one to post a message and one to retrieve all messages.

### 2a — postMessage Lambda

1. Navigate to **Lambda** → **Create function**.
2. Choose **Author from scratch** and set:
   - **Function name:** `postMessage`
   - **Runtime:** `Node.js 22.x`
   - **Permissions:** Expand _Change default execution role_ → select **Use an existing role** → choose **LabRole**
3. Click **Create function**.
4. In the **Code** tab, replace the default code with the following:

```javascript
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient, PutCommand } from "@aws-sdk/lib-dynamodb";
import { randomUUID } from "crypto";

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);

export const handler = async (event) => {
  const body = JSON.parse(event.body || "{}");
  const text = body.message?.trim();

  if (!text) {
    return {
      statusCode: 400,
      headers: { "Access-Control-Allow-Origin": "*" },
      body: JSON.stringify({ error: "message field is required" }),
    };
  }

  const item = {
    messageId: randomUUID(),
    message: text,
    timestamp: new Date().toISOString(),
  };

  await docClient.send(new PutCommand({ TableName: "messages", Item: item }));

  return {
    statusCode: 201,
    headers: { "Access-Control-Allow-Origin": "*" },
    body: JSON.stringify({ success: true, item }),
  };
};
```

5. Click **Deploy**.

### 2b — getMessages Lambda

1. Navigate to **Lambda** → **Create function**.
2. Choose **Author from scratch** and set:
   - **Function name:** `getMessages`
   - **Runtime:** `Node.js 22.x`
   - **Permissions:** Expand _Change default execution role_ → select **Use an existing role** → choose **LabRole**
3. Click **Create function**.
4. In the **Code** tab, replace the default code with the following:

```javascript
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient, ScanCommand } from "@aws-sdk/lib-dynamodb";

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);

export const handler = async () => {
  const result = await docClient.send(
    new ScanCommand({ TableName: "messages" }),
  );

  const sorted = (result.Items || []).sort(
    (a, b) => new Date(b.timestamp) - new Date(a.timestamp),
  );

  return {
    statusCode: 200,
    headers: { "Access-Control-Allow-Origin": "*" },
    body: JSON.stringify(sorted),
  };
};
```

5. Click **Deploy**.

---

## Part 3 — Create the API Gateway

1. Navigate to **API Gateway** → **Create API**.
2. Choose **REST API** (not private) → **Build**.
3. Set:
   - **API name:** `ChatWallAPI`
   - **Endpoint type:** Regional
4. Click **Create API**.

### 3a — /messages resource

1. In the left panel click **Resources**, then **Actions** → **Create Resource**.
2. Set **Resource Name** to `messages` (path will auto-fill to `/messages`).
3. Check **Enable API Gateway CORS** → **Create Resource**.

### 3b — POST method (postMessage)

1. With `/messages` selected, click **Actions** → **Create Method** → choose **POST** → click the checkmark.
2. Set:
   - **Integration type:** Lambda Function
   - **Lambda Proxy Integration:** Make sure this is checked
   - **Lambda Region:** your current region (e.g., `us-east-1`)
   - **Lambda Function:** `postMessage`
3. Click **Save** → **OK** when prompted to grant permission.

### 3c — GET method (getMessages)

1. With `/messages` selected, click **Actions** → **Create Method** → choose **GET** → click the checkmark.
2. Set:
   - **Integration type:** Lambda Function
   - **Lambda Proxy Integration:** Make sure this is checked
   - **Lambda Region:** your current region
   - **Lambda Function:** `getMessages`
3. Click **Save** → **OK**.

### 3d — Deploy the API

1. Click **Actions** → **Deploy API**.
2. Set:
   - **Deployment stage:** `[New Stage]`
   - **Stage name:** `prod`
3. Click **Deploy**.
4. Copy the **Invoke URL** shown at the top (e.g., `https://abc123.execute-api.us-east-1.amazonaws.com/prod`). You will need this for the frontend.

### 3e — Test the API

Use the API Gateway console to send a test POST request:

1. Click **Resources** → `POST` under `/messages` → **Test**.
2. In the **Request Body** field enter:
   ```json
   { "message": "Hello from the chat wall!" }
   ```
3. Click **Test** — you should see a `201` response.
4. Click **GET** under `/messages` → **Test** → click **Test** again — you should see your message in the response body.

---

## Part 4 — Build and Host the Frontend on S3

### 4a — Create an S3 bucket

1. Navigate to **S3** → **Create bucket**.
2. Set:
   - **Bucket name:** `chat-wall-frontend-<your-initials>` (must be globally unique)
   - **Region:** same region as your Lambda functions
3. Under **Block Public Access settings**, uncheck **Block all public access** and confirm the warning checkbox.
4. Click **Create bucket**.

### 4b — Enable static website hosting

1. Open your new bucket → **Properties** tab.
2. Scroll to **Static website hosting** → **Edit**.
3. Set:
   - **Static website hosting:** Enable
   - **Index document:** `index.html`
4. Click **Save changes**.

### 4c — Add a bucket policy

1. Go to the **Permissions** tab → **Bucket policy** → **Edit**.
2. Paste the following (replace `YOUR-BUCKET-NAME`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*"
    }
  ]
}
```

3. Click **Save changes**.

### 4d — Create the frontend file

Create a file named `index.html` on your local machine with the following content. Replace `YOUR_API_INVOKE_URL` with the URL you copied in Step 3d.

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Chat Wall</title>
    <style>
      body {
        font-family: sans-serif;
        max-width: 600px;
        margin: 40px auto;
        padding: 0 16px;
      }
      h1 {
        color: #232f3e;
      }
      textarea {
        width: 100%;
        height: 80px;
        padding: 8px;
        font-size: 1rem;
        box-sizing: border-box;
      }
      button {
        margin-top: 8px;
        padding: 10px 24px;
        background: #ff9900;
        border: none;
        color: #fff;
        font-size: 1rem;
        cursor: pointer;
        border-radius: 4px;
      }
      button:hover {
        background: #e68900;
      }
      #status {
        margin-top: 8px;
        font-size: 0.9rem;
        color: green;
      }
      #feed {
        margin-top: 24px;
      }
      .msg-card {
        background: #f4f4f4;
        border-left: 4px solid #ff9900;
        padding: 12px;
        margin-bottom: 12px;
        border-radius: 4px;
      }
      .msg-card .text {
        font-size: 1rem;
      }
      .msg-card .meta {
        font-size: 0.75rem;
        color: #888;
        margin-top: 4px;
      }
    </style>
  </head>
  <body>
    <h1>💬 Chat Wall</h1>

    <textarea id="msgInput" placeholder="Write a message..."></textarea>
    <br />
    <button onclick="postMessage()">Post Message</button>
    <div id="status"></div>

    <h2>Messages</h2>
    <div id="feed">Loading...</div>

    <script>
      const API = "YOUR_API_INVOKE_URL/messages";

      async function loadMessages() {
        document.getElementById("feed").innerHTML = "Loading...";
        try {
          const res = await fetch(API);
          const data = await res.json();
          if (!data.length) {
            document.getElementById("feed").innerHTML =
              "<p>No messages yet. Be the first!</p>";
            return;
          }
          document.getElementById("feed").innerHTML = data
            .map(
              (m) => `
          <div class="msg-card">
            <div class="text">${escapeHtml(m.message)}</div>
            <div class="meta">${new Date(m.timestamp).toLocaleString()}</div>
          </div>
        `,
            )
            .join("");
        } catch (e) {
          document.getElementById("feed").innerHTML =
            "<p style='color:red'>Failed to load messages.</p>";
        }
      }

      async function postMessage() {
        const text = document.getElementById("msgInput").value.trim();
        const status = document.getElementById("status");
        if (!text) {
          status.style.color = "red";
          status.textContent = "Please enter a message.";
          return;
        }
        status.style.color = "gray";
        status.textContent = "Posting...";
        try {
          const res = await fetch(API, {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({ message: text }),
          });
          if (res.ok) {
            document.getElementById("msgInput").value = "";
            status.style.color = "green";
            status.textContent = "Message posted!";
            loadMessages();
          } else {
            status.style.color = "red";
            status.textContent = "Failed to post.";
          }
        } catch (e) {
          status.style.color = "red";
          status.textContent = "Network error.";
        }
      }

      function escapeHtml(str) {
        return str
          .replace(/&/g, "&amp;")
          .replace(/</g, "&lt;")
          .replace(/>/g, "&gt;");
      }

      loadMessages();
    </script>
  </body>
</html>
```

### 4e — Upload the file

1. Open your S3 bucket → **Objects** tab → **Upload**.
2. Click **Add files**, select your `index.html`, and click **Upload**.

### 4f — Visit the website

1. Go to the **Properties** tab of your bucket.
2. Scroll to **Static website hosting** and click the **Bucket website endpoint** URL.
3. You should see the Chat Wall. Post a message and verify it appears in the feed!

---

## Verification Checklist

- [ ] DynamoDB `messages` table is Active
- [ ] `postMessage` Lambda returns `201` with a test POST
- [ ] `getMessages` Lambda returns posted messages
- [ ] API Gateway `POST /messages` and `GET /messages` are deployed to `prod` stage
- [ ] S3 bucket is publicly accessible with static hosting enabled
- [ ] Frontend loads, posts a message, and displays it without errors

---

## Stretch Challenges

### Add Usernames

- Update the POST body to accept a `username` field alongside `message`.
- Store `username` as an additional attribute in DynamoDB.
- Update `postMessage` Lambda to save and return `username`.
- Update the frontend to include a username input and display it on each message card.

### Add Message Deletion

- Create a new Lambda function `deleteMessage` that calls `DeleteCommand` with the `messageId`.
- Add a `DELETE /messages/{messageId}` resource and method in API Gateway.
- Add a delete button on each message card in the frontend that calls the new endpoint.

### Add a Message Limit

- In `getMessages`, use a `Limit` parameter on the `ScanCommand` (or sort and slice) to return only the 20 most recent messages.

---

## Cleanup (Important!)

To avoid charges, delete resources when you are done:

1. **S3** — Empty the bucket, then delete it.
2. **API Gateway** — Delete `ChatWallAPI`.
3. **Lambda** — Delete `postMessage` and `getMessages`.
4. **DynamoDB** — Delete the `messages` table.
