# ChatGPT Ads (OpenAI) pixel via GTM

A workflow that lets an LLM agent install the ChatGPT Ads measurement pixel and its conversion events on **any custom-built website** through Google Tag Manager.

The agent reads OpenAI's pixel docs, works out the business type (purchase, lead generation, subscription, booking, content), chooses the events, reads the site's real dataLayer, reuses the container's existing triggers, and builds the tags in a separate GTM workspace. It never publishes.

## What is here

| Path | Use it when |
|---|---|
| [`chatgpt-ads-pixel-gtm-prompt.md`](chatgpt-ads-pixel-gtm-prompt.md) | You use any LLM agent. One self-contained file: paste it in and fill the three inputs. |
| [`skill/chatgpt-ads-pixel-gtm/`](skill/chatgpt-ads-pixel-gtm/) | You use Claude Code. `SKILL.md` is the workflow, `reference.md` holds the pixel facts and tag code. |

Both contain the same workflow. The prompt is the two skill files joined together.

## Requirements

- A GTM MCP server connected to the agent, with edit access to the container.
- A browser tool, so the agent can open the website and read `dataLayer`.

## Inputs the agent asks for

1. Website URL
2. GTM account name and container name
3. OpenAI Pixel ID (Ads Manager → Conversions)

It stops and asks if any of the three is missing.

## Install the skill (Claude Code)

Copy the folder into your personal skills directory:

```bash
cp -r skill/chatgpt-ads-pixel-gtm ~/.claude/skills/
```

Then ask Claude Code to set up the ChatGPT Ads pixel for a website.

## Conventions the workflow follows

- Names: `EA - <Platform> - <event>`, for example `EA - OpenAI - order_created`.
- The Pixel ID lives only in the constant variable `EA - OpenAI - pixel_id`.
- Existing triggers and variables are reused; a trigger is created only when none exists.
- Amounts are sent as integers in minor units (129.99 USD → `12999`), as OpenAI requires.
- Work happens in a workspace named `EA - OpenAI Pixel`. Publishing is always left to you.

## Source docs

- https://developers.openai.com/ads/measurement-pixel
- https://developers.openai.com/ads/supported-events
