# Metorial Agent Kit
Use Metorial like a pro, without reading the docs! 

This repo is your starting point.

## Motivation
This was inspired by recent Stanford research claiming that fine-tuning can be easily beaten by in-context and context driven work - this repo has ALL the context, right at your fingertips. [(link)](https://venturebeat.com/ai/fine-tuning-vs-in-context-learning-new-research-guides-better-llm-customization-for-real-world-tasks)

There are now THOUSANDS of AI Tools that can provide customer service and developer experience help, but NONE of them can run your code. At the same time, Claude Code, Cursor, and Codex are rapidly saturating the market, so why do developers need to read docs?

This project contains a clean and simple specialized agent that learns via ICL to provide improved handling of metorial usage questions, and to implement them in real time.  

## What's included:
+ Multi agent prompt kit
+ Subagent management
+ Q&A Manager
+ Drafting & Publishing Toolkit

## Demo
The following video shows dozens of Haiku 4-5 subagents ripping through the docs and building custom, targetted docs and example code, right on your computer. Try it yourself with the stpes below!

https://github.com/user-attachments/assets/demo-video-compressed.mp4

<video src="demo-video-compressed.mp4" controls></video>

## How to use this repo
1. Clone this repo 
`git@github.com:alexander-morris/metorial-agent-kit-yc-hackathon.git`

2. Open your favorite coding agent
```sh
cd metorial-agent-kit-yc-hackathon
claude --dangerously-skip-permissions 
```
Note: you can remove --dangerously-skip-permissions but it is a lot easier if you don't

3. Tell claude to run subagents to solve for your current challenges, e.g.:
`Work on building a metorial MCP integration for my local project in ~/my-project-that-is-not-an-mcp by converting it's API to metorial`

Boom - just like that, you'll see dozens of Haiku 4.5 subagents spin up and start working on it. 

Your agents will each do a small part of the larger work, whether it's research, code improvements, or more. Results will be published to a .md as well as relevant code folders inside this project folder.

Haiku 4.5 subagents are low-context, and will be narrowly scoped, which provides a major improvment  

## Using Tribe
***Note: We recommend using [Tribe](https://tribecode.ai) to use this repo, so you can see what all the subagents are doing.***

To get Tribe:
```sh
npx @_xtribe/cli@latest
tribe
```

This will show you all the agent processes as they run, and can help ensure you get the best results.

## Benchmarks & Comparison 
When run in parallel, this solution acheives a 10x performance improvement on speed for both research and complex coding tasks.
