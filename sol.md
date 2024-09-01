Certainly! Let's implement the core system using Node.js. Below is a detailed plan with examples.

### 1. **Database Setup with Elasticsearch**

We will set up two indices in Elasticsearch:

1. **Emails Index**: For storing email data like subject, sender, recipient, etc.
2. **Mailboxes Index**: For storing mailbox details like folder names and email counts.

### 2. **Node.js API Development**

Let's start by setting up an Express.js server with the necessary routes and OAuth integration using Microsoft's `@azure/msal-node` package.

**1. Set Up the Project**

First, create a new Node.js project and install the necessary packages:

```bash
mkdir email-client-system
cd email-client-system
npm init -y
npm install express body-parser @elastic/elasticsearch @azure/msal-node dotenv
```

**2. Create the Basic Structure**

Create the following files:
- `index.js`: Entry point for the Express server.
- `auth.js`: Handles the OAuth process.
- `emailSync.js`: Manages the email synchronization.
- `.env`: Stores environment variables (e.g., client secrets).

**3. OAuth Integration with Outlook (using MSAL)**

In `auth.js`, set up the OAuth flow:

```javascript
const msal = require('@azure/msal-node');
require('dotenv').config();

const msalConfig = {
    auth: {
        clientId: process.env.CLIENT_ID,
        authority: `https://login.microsoftonline.com/${process.env.TENANT_ID}`,
        clientSecret: process.env.CLIENT_SECRET,
    }
};

const pca = new msal.ConfidentialClientApplication(msalConfig);

const authUrl = (req, res) => {
    const authCodeUrlParameters = {
        scopes: ["User.Read", "Mail.ReadWrite"],
        redirectUri: process.env.REDIRECT_URI,
    };

    pca.getAuthCodeUrl(authCodeUrlParameters).then(response => {
        res.redirect(response);
    }).catch(error => console.log(JSON.stringify(error)));
};

const handleAuthRedirect = (req, res) => {
    const tokenRequest = {
        code: req.query.code,
        scopes: ["User.Read", "Mail.ReadWrite"],
        redirectUri: process.env.REDIRECT_URI,
    };

    pca.acquireTokenByCode(tokenRequest).then(response => {
        // Store the access token securely
        const { accessToken, idTokenClaims } = response;
        const userId = idTokenClaims.sub; // Map or create a local ID

        // Store the user info in your database
        // Then initiate email synchronization
        res.redirect('/sync');
    }).catch(error => console.log(error));
};

module.exports = { authUrl, handleAuthRedirect };
```

**4. Email Synchronization**

In `emailSync.js`, implement the logic to fetch and store emails:

```javascript
const { Client } = require('@elastic/elasticsearch');
const axios = require('axios');

const elasticClient = new Client({ node: process.env.ELASTIC_NODE });

const syncEmails = async (accessToken, userId) => {
    try {
        const response = await axios.get('https://graph.microsoft.com/v1.0/me/mailFolders/inbox/messages', {
            headers: { Authorization: `Bearer ${accessToken}` }
        });

        const emails = response.data.value;

        for (const email of emails) {
            await elasticClient.index({
                index: 'emails',
                document: {
                    user_local_id: userId,
                    subject: email.subject,
                    sender: email.from.emailAddress.address,
                    recipient: email.toRecipients.map(recipient => recipient.emailAddress.address).join(', '),
                    body: email.body.content,
                    timestamp: email.receivedDateTime,
                    folder: 'inbox',
                    is_read: email.isRead,
                    is_flagged: email.flag.flagStatus === 'flagged'
                }
            });
        }
    } catch (error) {
        console.error(error);
    }
};

module.exports = { syncEmails };
```

**5. Create Express Routes**

In `index.js`, set up the routes to handle the OAuth flow and email synchronization:

```javascript
const express = require('express');
const bodyParser = require('body-parser');
require('dotenv').config();

const { authUrl, handleAuthRedirect } = require('./auth');
const { syncEmails } = require('./emailSync');

const app = express();
app.use(bodyParser.json());

app.get('/login', authUrl);
app.get('/callback', handleAuthRedirect);
app.get('/sync', async (req, res) => {
    const accessToken = ''; // Retrieve from your database
    const userId = ''; // Retrieve from your database
    await syncEmails(accessToken, userId);
    res.send('Email synchronization initiated');
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

**6. Environment Configuration**

Create a `.env` file to store sensitive information:

```
CLIENT_ID=your_client_id
CLIENT_SECRET=your_client_secret
TENANT_ID=your_tenant_id
REDIRECT_URI=http://localhost:3000/callback
ELASTIC_NODE=http://localhost:9200
PORT=3000
```

### 3. **Real-Time Updates**

To handle real-time updates, you can either poll the Microsoft Graph API at intervals or use the Microsoft Graph webhook to get notifications about changes in the user's mailbox.

**Example of a Webhook:**
You can create a webhook subscription to get notified when an email is received or changed.

### 4. **Scalability and Extensibility**

This system can be scaled horizontally by deploying it on cloud services like AWS or Azure and using managed Elasticsearch clusters.

For extensibility, you can add more routes and logic to support additional email providers by implementing IMAP or OAuth flows for those providers.

### 5. **Frontend Implementation**

For the frontend, you could create a simple React app with pages for:
- Adding and connecting an account.
- Viewing synced emails.

This could be done using fetch or axios to interact with the Node.js API.

This solution provides a solid foundation for an email client system using Node.js, OAuth, Elasticsearch, and Express.