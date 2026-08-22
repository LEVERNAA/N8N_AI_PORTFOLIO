# RSS to AI Summary Email Workflow

## Problem
Keeping up with new articles/updates from an RSS feed takes time — reading everything manually is slow and easy to fall behind on.

## What it does
This n8n workflow automatically:
1. Checks an RSS feed on a schedule for new posts
2. Sends each new post to an AI model to generate a short summary
3. Sends the summary as an email

## Who it's for
Anyone who wants to stay updated on a blog, news source, or topic without reading every full article — e.g. content creators, researchers, or busy professionals.

## Tools used
- **n8n** — workflow automation
- **RSS Feed node** — to detect new content
- **AI model** — to generate summaries
- **Email node (SMTP/Gmail)** — to deliver the result

## How it works (step by step)
1. Trigger: n8n checks the RSS feed on a set schedule
2. New item detected: the workflow grabs the title, link, and content
3. AI summarization: the content is sent to the AI model with a prompt to summarize it clearly and briefly
4. Email: the summary is formatted and sent to the configured email address

## Demo
[Link to Loom video demo — add once recorded]

## What I'd improve next
- Support multiple RSS feeds at once
- Add filtering by keyword/topic before summarizing
- Send to Slack/Telegram as an alternative to email

