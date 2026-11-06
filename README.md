## Requirements

The following are strictly required before setting up this automation.

- Slack account.
- Permission to add apps to your slack workspace.
- N8n cloud account.
- LLM API key.
- GitHub account.


## 1. Setting Up From Scratch

This first section will detail how to set up this automation completely from scratch. If you have the automation set up already but want to start a new project skip to section 2.


### 1.1 Slack App

*NOTE: Any time you update your slack app settings you will need to save changes and/or reinstall it to your workspace for the changes to persist.*

The first step is to set up a slack app. Go to: [the slack api](https://api.slack.com/apps/) home page and create a new app. You want to select the create a basic app option, then select the workspace you want the app to work in.

Copy and paste the `slack-bot-manifest.json` file into the app manifest section of the app settings and save changes.

#### 1.1.1 Permissions

You need the following bot scopes enabled:

| Scope | Reason |
|-------|--------|
| canvases:read | Allows the bot to access the generated huddle summary. |
| files:read | Required for the bot to be able to listen for files shared events. |
| chat:write | Allows the bot to write confirmation emails to the slack channel. |
| channels:read | Allows the bot to read information about a public slack channel. |
| channels:history | Allows the bot to read chat messages in a public channel. |
| groups:read | Allows the bot to read information about a private slack channel. |
| groups:history | Allows the bot to read chat messages in a private channel. |

You need the following bot event subscription:

| Scope | Reason |
|-------|--------|
| file_shared | Allows the bot to listen for a file being shared in a channel. |

#### 1.1.2 Authentication

You may need permission from your workspace admin to add the app to your workspace, then once it's added you'll need to invite it to any channels you want it to watch.

In the app settings ensure socket mode is **off**.

Go to your n8n cloud workspace and create a new **Slack API** credential. The access token is your **Bot User OAuth Token** which is found in the 'OAuth and Permissions' tab of the app settings; it will be prefixed with `xoxb-`. The other fields you can leave as default.

Once your credential is set up, create a new n8n workflow. Within the workflow create a slack trigger node for a file shared. Add in your newly created bot credential and select either 'Watch Whole Workspace'.

Publish your workflow then copy the **Production URL** under the 'webhook URL' tab of the trigger node. Navigate back to your app settings, go to the 'Event Subscriptions' section and ensure 'Enable Events' is set to **on**. In the 'Request URL' box directly underneath add in the copied production URL and it should give you a green tick and text saying 'verified'.

*NOTE: If the URL doesn't get verified here then the automation will not work. Some things which may cause this:*
- *Leading or trailing whitespace in your URL.*
- *Copying the test URL instead of the production URL.*
- *An error in setting up your n8n bot credential.*
- *You added a signature secret to your n8n bot credential.*

To test everything is working correctly connect your slack trigger node in n8n to a slack 'send message' node and set it to send a message to the channel your bot is in. Then upload a file to that chat in slack and you should see a message come through almost instantly.

*NOTE: If at this point you start getting infinite message triggers, that's ok. Just unpublish the workflow and check you don't have the slack trigger node set to listen for any event or a message being posted.*


### 1.2 N8N

Delete any exisiting nodes in your n8n workflow so that you are starting from a blank canvas. Copy the full JSON file found in `n8n-workflows/transcription.json` and paste it into your n8n workflow.

Then create 2 new n8n workflows and copy into them the contents of the `n8n-workflows/setup.json` and `n8n-workflows/transcription-error-handler.json` files.

#### 1.2.1 Data Tables

You will need to go to your n8n cloud dashboard and create a new data table found under the data tables tab. Create three new columns:
- `channel_id` (string)
- `project_id` (string)
- `last_upload_ts` (number)

This data table does 2 jobs:
1. It links a specific project to a channel so that multiple channels can be assigned to different projects.
2. When slack posts the transcription summary to the channel, it does so twice (who knows why?) and this data table allows the workflow to stop any duplicate runs which occur within ten seconds of the first trigger.

#### 1.2.2 Credentials

For each slack node (*visible by the slack icon on the node*) in all 3 workflows ensure the credentials are set using the slack API credential you created in section 1.1.2. For the 'Get Summary Text' HTTP request node in the transcription workflow, add in the same slack API credential ensuring you select *Predefined Credential Type -> Slack API* from the dropdown list.

Add an Open AI API credential to the 'Open AI Chat Model' node attached to the 'Create Issues' LLM node. If you have an LLM API credential from a different provider then delete the OpenAI node and replace it with the one you need and add in your credential. If you don't have an LLM API credential then you will need to navigate back to the credentials tab of the n8n dashboard and create a new one, then add it in to the node.


### 1.3 GitHub

To set up the GitHub integration for the workflow you will need a GitHub account which can be invited to any project you want to connect the automation to. If you have a busy workspace it is recommended you create a new account for this.

#### 1.3.1 Token

In the GitHub browser click on your profile icon and navigate to settings. At the bottom of the left hand navigation bar is a section named 'developer settings'. In this tab you need to naviagte to '*Personal Access Tokens -> Tokens (classic)*' and create a new classic token. Set the expiration to never and make sure you add in the full **project** and **repo** scopes, otherwise you will not be able to access the necessary information to allow the workflow to compare new tickets against those already created.

*NOTE: Make sure the token you create is a classic one. Fine grained tokens will not work even with full permissions.*

Once you have your token, copy it (*NOTE: If you exit the tab without copying you will have to regenerate the token in order to copy it again.*) and create a new n8n credential and select the GitHub API option. Add your username and under access token paste in your classic token value.

After creating your token add it to the 'Get Existing Project Tickets' and 'Add Issues to Project' nodes in the transcription workflow. Add the same GitHub credential to all the HTTP nodes in the setup workflow.

#### 1.3.2 Connecting to a Project

In a slack channel which the slack app has been invited to, run the `/setup` command by typing it into the chat box. You should receive a reply in the channel with a form to fill out. Into that form copy the URL from the home page of the project you want to connect and the repository you want to connect.

*NOTE: In GitHub you can connect a project board to many repos, but only the one you enter here will be scanned for duplicate tickets. Support for multiple repos connected to the same project is not currently vailable*


## 2. Setting Up a New Project

If the automation is already set up for your workspace then setup is simple:

- Invite the app to the channel you want to use for the automation.
- Follow step 1.3.2.


## Summary

You are now set up and ready to use the automation. Host a short huddle meeting with AI transcription turned on to test everything is working correctly.

If you hit any issues or bugs then please open a GitHub issue and include as much detail about what went wrong as possible. I will investigate and get back to you as soon as I can.