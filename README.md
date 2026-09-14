# Slack Recipe AI Automation --- n8n

## Overview

This project is a Slack-based AI recipe assistant built with n8n.

A user sends a recipe request in a Slack channel, n8n receives the
message automatically, sends it to an OpenAI chat model, cleans the
generated response, and posts the recipe back to the same Slack channel.

Example:

**User** \> give me a simple pasta recipe for 2 people

**Bot** \> Simple tomato-basil pasta --- ingredients for 2 people: \> -
Spaghetti: 180--200 g \> - Tomatoes: 400 g \> - Olive oil: 2 tbsp \> -
Garlic: 2 cloves \> ...

Once published, the workflow runs automatically. The user does not need
to click **Execute workflow**.

------------------------------------------------------------------------

## Workflow Architecture

``` text
Slack
  |
  v
Slack Trigger
  |
  v
IF — Ignore bot messages
  | True
  v
Basic LLM Chain
  ^
  |
OpenAI Chat Model
  |
  v
Code in JavaScript
  |
  v
Slack — Send a message
```

## Nodes Used

### 1. Slack Trigger

Purpose: Starts the workflow whenever a new message is posted in the
recipe Slack channel.

Configuration used:

-   Event: **New Message Posted to Channel**
-   Watch Whole Workspace: **Off**
-   Channel: **By ID**
-   Recipe channel ID: `C0C16D2EJEB`

The Slack app/bot must be invited to the channel.

For development, n8n provides a Test webhook URL containing:

``` text
/webhook-test/
```

For the published workflow, Slack Event Subscriptions must use the
Production URL containing:

``` text
/webhook/
```

The Production Request URL should show **Verified** in Slack.

------------------------------------------------------------------------

### 2. IF --- Prevent Bot Reply Loops

Purpose: Prevents the Slack bot from triggering the workflow again when
it posts its own response.

Condition:

``` text
{{ $json.bot_id }}
```

Operator:

``` text
does not exist
```

Behavior:

-   Human Slack message: `bot_id` normally does not exist → **True**
-   Bot-generated message containing `bot_id` → **False**

Only the **True** output is connected to the Basic LLM Chain.

The False output is left unconnected.

This prevents a loop such as:

``` text
User → Bot → Bot sees its own reply → Bot replies again → ...
```

------------------------------------------------------------------------

### 3. Basic LLM Chain

Purpose: Converts the user's Slack message into a recipe/ingredient
request for the AI model.

The Slack message can be read from the event data. In our captured Slack
event, the message was available inside the Slack block structure.

Example expression used in the prompt:

``` text
{{ $json.blocks[0].elements[0].elements[0].text }}
```

Example prompt:

``` text
List all the products and ingredients needed to make {{ $json.blocks[0].elements[0].elements[0].text }}.

Return a clear bulleted list with approximate quantities.
Keep it short and practical. If the recipe name is unclear,
ask the user to rephrase it.
```

The Basic LLM Chain is connected to an **OpenAI Chat Model**.

------------------------------------------------------------------------

### 4. OpenAI Chat Model

Purpose: Generates the recipe response.

The OpenAI model is connected to the **Model** input of the Basic LLM
Chain.

The OpenAI credentials are stored securely inside n8n credentials and
should never be hard-coded into workflow fields or documentation.

------------------------------------------------------------------------

### 5. Code in JavaScript

Purpose: Cleans the AI response before it is sent to Slack.

Example:

``` javascript
// Get the Basic LLM Chain response
let text = $json.text ?? '';

// Convert escaped newlines into real line breaks
text = text.replace(/\\n/g, '\n');

// Normalize Windows line endings
text = text.replace(/\r\n/g, '\n');

// Remove trailing spaces
text = text.replace(/[ \t]+$/gm, '');

// Remove excessive blank lines
text = text.replace(/\n{3,}/g, '\n\n');

// Normalize common bullet characters
text = text.replace(/^[•●▪]\s*/gm, '• ');

// Remove whitespace around the final response
text = text.trim();

// Log the cleaned response for debugging
console.log('Cleaned Slack text:', text);

// Return the cleaned text to the next node
return {
  json: {
    text: text
  }
};
```

The output passed to the Slack node is:

``` text
$json.text
```

------------------------------------------------------------------------

### 6. Slack --- Send a Message

Purpose: Sends the AI-generated recipe back into Slack.

Configuration:

-   Resource: **Message**
-   Operation: **Send**
-   Send Message To: **Channel**
-   Channel: **By ID**
-   Channel ID: `C0C16D2EJEB`
-   Message Type: **Simple Text Message**

Message Text:

``` text
{{ $json.text }}
```

The **Append n8n Attribution** option should be disabled if you do not
want this footer:

``` text
Automated with this n8n workflow
```

------------------------------------------------------------------------

## Slack Configuration

The Slack app needs:

1.  A bot user.
2.  Appropriate OAuth permissions for reading channel messages and
    posting messages.
3.  The Bot User OAuth Token configured in the n8n Slack credential.
4.  Event Subscriptions enabled.
5.  The n8n Production webhook configured as the Slack Request URL.
6.  The bot invited to the recipe channel.

The Slack credential created in n8n was named:

``` text
Slack AI CRED
```

Never put the actual Slack token in a README or workflow documentation.

------------------------------------------------------------------------

## Development vs Production

### Testing

During development, the Slack Trigger can use the n8n Test URL:

``` text
https://your-n8n-domain/webhook-test/...
```

n8n must be actively listening for a test event.

This mode is temporary and normally requires starting a test execution.

### Production

For continuous Slack automation, use:

``` text
https://your-n8n-domain/webhook/...
```

The workflow must be **Published**.

Slack's Event Subscriptions Request URL should point to this Production
URL and show **Verified**.

After publishing, users can simply send messages in Slack. There is no
need to click **Execute workflow**.

------------------------------------------------------------------------

## Final Production Flow

``` text
User sends recipe request in Slack
            |
            v
      Slack Trigger
            |
            v
    Is bot_id missing?
        /       \
      Yes        No
       |          |
       v          Stop
 Basic LLM Chain
       |
       v
 OpenAI Chat Model
       |
       v
 JavaScript Cleanup
       |
       v
 Send Slack Message
       |
       v
 User receives recipe
```

------------------------------------------------------------------------

## Example

User sends:

``` text
give me a simple pasta recipe for 2 people
```

The workflow automatically processes the request and the Slack bot
responds with ingredients and approximate quantities.

No manual n8n execution is required in production.

------------------------------------------------------------------------

## Troubleshooting

### Slack Trigger works only when Execute Workflow is clicked

The workflow is probably still using the Test webhook.

Check that:

-   Workflow is Published.
-   Slack Event Subscriptions uses the Production `/webhook/` URL.
-   Slack shows the Request URL as Verified.

### Bot keeps replying to itself

Check the IF node.

It should test:

``` text
{{ $json.bot_id }}
```

with:

``` text
does not exist
```

Only the True output should continue to the AI.

### Slack shows `undefined`

Check the Code node output and Slack Message Text.

The Code node should return:

``` text
text
```

and the Slack node should use:

``` text
{{ $json.text }}
```

### Slack adds an n8n footer

Disable **Append n8n Attribution** in the Slack Send Message node.

------------------------------------------------------------------------

## Current Limitation --- Conversation Memory

The current workflow uses a **Basic LLM Chain**, so every Slack message
is essentially processed as a new request.

For example:

``` text
User: Give me a pasta recipe for 2 people.
Bot: [recipe]

User: Make it for 4 people.
```

The second message may not reliably understand that **"it"** refers to
the previous pasta recipe.

A future enhancement is to replace the Basic LLM Chain with an **AI
Agent** and add a memory component, using a Slack user/channel/thread
identifier as the conversation session key.

That would make the automation behave more like a continuous chat
assistant.

------------------------------------------------------------------------

## Security Notes

-   Never commit Slack Bot OAuth Tokens to GitHub.
-   Never commit OpenAI API keys.
-   Store secrets using n8n Credentials.
-   Keep bot permissions limited to what the workflow requires.
-   Keep the bot-message IF filter enabled in production.

------------------------------------------------------------------------

## Status

The completed production workflow successfully:

-   Receives Slack messages automatically.
-   Restricts processing to the configured recipe channel.
-   Prevents the bot from responding to itself.
-   Sends recipe requests to OpenAI.
-   Cleans AI output with JavaScript.
-   Posts the result back to Slack.
-   Runs automatically after publishing.

**Next planned improvement:** conversational memory.
