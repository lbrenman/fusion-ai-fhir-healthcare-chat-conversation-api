# Amplify Fusion FHIR Healthcare AI Chat Conversation API

An Amplify Fusion FHIR Healthcare AI chat API that maintains conversation context across multiple turns using (Neon) PostgreSQL for message history storage. It has the following features:

* Conversation management using API generated converation id and OAuth 2.0 client id for conversation isolation. This is also useful for prompt/response audit logging which is often required in finance or healthcare applications
* Implements an LLM Agent Tool loop using a highly structured prompt and FHIR APIs via OpenAPI/HTTP Client
* Developed with Anthropic LLM model claude-sonnet-4-20250514
* Fallback LLM is Amazon Bedrick model us.anthropic.claude-sonnet-4-6
* Supports OAuth 2.0 for front end security
  * OAuth 2.0 - Extracts the Client Id and user roles from the Authorization header jwt token. Client Id is used for conversation management. Role is used for LLM tool approval
  * You can find a document for configuring KeyCloak for Fusion API's [here](https://gist.github.com/lbrenman/69317b109e0db85771ae29a2fab890c8)
* A sample FHIR Backend with FHIR R4 API and Dashboard can ne found [here](https://github.com/lbrenman/fhir-codespace).
* A sample web app that supports OAuth Client Credentials, Authcode with PKCE or API Key can be found [here](https://github.com/lbrenman/fusion-ai-chat-web-app-multi-auth). Note that you must use OAuth 2 PKCE for this API.

To get started, import the [AI_HCChatConversation.zip](https://github.com/lbrenman/fusion-ai-fhir-healthcare-chat-conversation-api/blob/main/AI_HCChatConversation.zip) project export into your Fusion tenant and enter credentials for all connections and activate. Project also includes a cron job integration that can purge old conversations if activated

* [Demo Video](https://youtu.be/NUGrLLQjRH0)
* [How It Was Built](https://youtu.be/4zNGkrP8Mbs)

## API

### POST /v1/prompt

Submit a prompt and receive a response from the LLM with full conversation context.

#### Request Body

```json
{
  "prompt": "What is my name?",
  "conversationId": "abc-123",
  "modelRequested": "gpt-4o"
}
```

> Note that `modelRequested` has no effect in this application

| Field | Type | Required | Description |
|---|---|---|---|
| `prompt` | string | Yes | The user's message to send to the LLM |
| `conversationId` | string | No | ID of an existing conversation. If omitted, a new conversation is created |
| `modelRequested` | string | No | The model to use for this request (e.g. `gpt-4o`, `claude-sonnet-4-6`). If omitted or unavailable, the server uses its default model |

#### Response Body

```json
{
  "message": "Your name is Leor.",
  "conversationId": "abc-123",
  "modelUsed": "gpt-4o"
}
```

| Field | Type | Description |
|---|---|---|
| `message` | string | The assistant's reply |
| `conversationId` | string | The conversation ID (use this in subsequent requests to maintain context) |
| `modelUsed` | string | The model that was actually used. May differ from `modelRequested` if the requested model was unavailable or not recognized |

#### Error Response

```json
{
  "message": "Error description"
}
```

## How It Works

Each turn follows this flow:

```
Client sends prompt + optional conversationId
       ↓
1. Upsert user message into DB
       ↓
2. Fetch full message array from DB
       ↓
3. Send full message array to OpenAI
       ↓
4. Upsert assistant response into DB
       ↓
5. Return response + conversationId to client
```

By passing the full message history to OpenAI on every request, the model has complete conversational context — allowing it to answer questions like "What is my name?" based on something the user said several turns earlier.

## Database

### PostgreSQL Schema

```sql
REATE TABLE conversations (
  conversation_id  VARCHAR(255) NOT NULL,
  client_id        VARCHAR(255) NOT NULL,
  created_at       TIMESTAMP DEFAULT NOW(),
  updated_at       TIMESTAMP DEFAULT NOW(),
  messages         JSONB NOT NULL DEFAULT '[]',
  PRIMARY KEY (conversation_id, client_id)
);
```

### Upsert Message

```sql
INSERT INTO conversations (conversation_id, client_id, messages)
VALUES (:conversationId, :clientId, CAST(:messageArray AS jsonb))
ON CONFLICT (conversation_id, client_id)
DO UPDATE SET
  messages = CAST(:messageArray AS jsonb),
  updated_at = NOW();
```

### Retrieve Message History

```sql
SELECT messages FROM conversations 
WHERE conversation_id = :conversationId
AND client_id = :clientId;
```

### Delete Old Conversations

```sql
DELETE FROM conversations
WHERE updated_at < NOW() - INTERVAL '24 hours';
```

Change '24 hours' to whatever TTL makes sense for your use case, e.g. '7 days', '1 hour', etc.

A schedule triggered integration is included for cleanup using the `Delete Old Conversations` SQL above

## Conversation ID Management

The client is responsible for managing conversation IDs.

- If no `conversationId` is provided, the server generates a new one and returns it in the response
- Pass the returned `conversationId` in all subsequent requests to maintain context
- A null or missing `conversationId` starts a fresh conversation with no prior context

## Sample Prompts

* Hello

  Hello! I'm your clinical assistant. I can help you access patient information from our FHIR system. What patient information are you looking for today?

* What is my role?

  Your role is nurse. As a nurse, you have access to patient search functionality to help you find patients by name.

  or

  You are logged in as a doctor. This gives you access to patient records and other clinical information through our system. How can I assist you with patient care today?


* get patient pat-004

  You do not have permission to access that resource.

  or

  Patient Information
  Name: James Williams
  Patient ID: pat-004
  Medical Record Number: MRN-004
  Status: Active

  Demographics
  Gender: Male
  Date of Birth: January 30, 1962
  Age: ~62 years old

  Contact Information
  Phone: 503-555-1004 (home)
  Address: Portland, OR, US

  Care Team
  Managing Organization: Organization/org-cedar
  General Practitioner: Practitioner/prac-johnson

* what can you do?

  I am a clinical assistant. Please describe what patient information you are looking for and I will help you.

* what can i do?

  As a nurse, you can search for patients by their first and last name. For example, you can ask me to "find patient John Smith" or "search for patient Mary Johnson" and I'll help you locate their records.

  or

  As a doctor, I can help you access the following patient information:

  Available Actions
  🔍 Patient Search & Retrieval
  Search patients by name - Find patients using their first and last name
  Get patient by ID - Retrieve detailed patient records using their FHIR Patient ID
  How to Use
  Just tell me what you're looking for in natural language, such as:

  "Find patient John Smith"
  "Get patient pat-123"
  "Show me information for Sarah Johnson"
  What patient information would you like to access?

* get patient James Williams

  James Williams

  Basic Demographics
  Patient ID: pat-004
  MRN: MRN-004
  Gender: Male
  Date of Birth: January 30, 1962
  Status: Active
  Contact Information
  Phone: 503-555-1004 (home)
  Address: Portland, OR, US
  Care Team
  Managing Organization: Organization/org-cedar
  General Practitioner: Practitioner/prac-johnson


* get patient Margaret Thompson

  Patient Information
  Margaret Thompson

  Basic Demographics
  Patient ID: pat-001
  MRN: MRN-001
  Gender: Female
  Date of Birth: March 12, 1955
  Status: Active
  Contact Information
  Phone: 916-555-1001 (home)
  Address: Sacramento, CA, US
  Care Team
  Managing Organization: Organization/org-mercy
  General Practitioner: Practitioner/prac-chen


* what can you help me with

  Clinical Assistant Capabilities
  I can help you access patient information through the following services:

  🔍 Patient Search
  Find patients by name - Search for patients using their first and last name
  Retrieve patient records - Get detailed patient information using their FHIR Patient ID
  💊 Medication Management
  Patient medications by ID - Retrieve all medications for a patient using their Patient ID
  Patient medications by name - Get medication information by searching with patient's first and last name
  📋 Available Information
  When retrieving patient data, you can access:

  Patient demographics and contact information
  Current and past medication requests
  Medication status (active, completed, stopped)
  Prescribing physician information
  Dosage instructions and timing
  Prescription dates
  How to get started: Simply tell me what patient information you're looking for. For example:

  "Find patient John Smith"
  "Get medications for patient pat-123"
  "Show me Sarah Johnson's medication list"
  What would you like to look up today?

* get medication for patient pat-001

  Medications for Patient pat-001 (Margaret Thompson)
  Active Medications
  Sertraline 50 MG Oral Tablet

  Status: Active
  Prescribed: June 30, 2025
  Prescriber: Dr. Chen
  Dosage: Take as directed, once daily
  RxNorm Code: 372614

* get medication for patient Margaret Thompson

  Medications for Margaret Thompson
  Active Medications
  💊 Sertraline 50 MG Oral Tablet
  Status: ✅ Active
  Prescribed: June 30, 2025
  Prescriber: Dr. Chen
  Dosage Instructions: Take as directed, once daily
  RxNorm Code: 372614
  Last Updated: May 21, 2026
  Total Active Medications: 1

  Margaret Thompson currently has one active medication prescription for Sertraline, which is commonly used to treat depression and anxiety disorders.

* what is my role

  You are logged in as a doctor. As a doctor, you have access to:

  Patient records by ID
  Patient search by name
  Patient medication information
  How can I help you access patient information today?

* get medication for last patient

  Medication Records for Michael Taylor (pat-010)
  Total Medications: 3

  1. Atorvastatin 10 MG Oral Tablet
  Status: Completed
  Prescribed: June 1, 2025
  Prescriber: Practitioner/prac-nguyen
  Dosage: Take as directed - Once daily
  RxNorm Code: 314200

  2. Metformin 500 MG Oral Tablet ⭐ Active
  Status: Active
  Prescribed: March 6, 2024
  Prescriber: Practitioner/prac-nguyen
  Dosage: Take as directed - Once daily
  RxNorm Code: 860975

  3. Albuterol 0.83 MG/ML Inhalation Solution ⭐ Active
  Status: Active
  Prescribed: June 16, 2023
  Prescriber: Practitioner/prac-nguyen
  Dosage: Take as directed - Once daily
  RxNorm Code: 310798

  Note: ⭐ indicates currently active medications

* Make appointment for Michael Taylor, pat-010 for June 20, 2026 at 9am for 30 minutes

  Appointment Successfully Created ✅
  Appointment Details:

  Patient: Michael Taylor (ID: pat-010)
  Date: June 20, 2026
  Time: 9:00 AM - 9:30 AM (30 minutes)
  Service Type: Follow-up
  Status: Booked
  Appointment ID: 60be3e0b5601
  The appointment has been confirmed and the patient has been marked as accepted.