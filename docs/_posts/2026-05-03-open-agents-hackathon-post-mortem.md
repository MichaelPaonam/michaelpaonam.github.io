---
layout: post
title: "ETHGlobal Open Agents: What the Winners Did Differently"
date: 2026-05-03
categories: web3
---

I spent 14 days building ARYA — a multi-agent AI swarm for DeFi yield farming — for the ETHGlobal Open Agents hackathon. It didn't place. Here's what I built, what the finalists built, and where I think the gap was.

<div class="iframe-container">
<iframe src="https://www.youtube.com/embed/9g2LWZVdooE?si=RHB4PdAAZUCgaSM7" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>
</div>

## What I built

ARYA is four specialized AI agents (Scout, Risk, Executor, Orchestrator) that collaboratively discover yield farming opportunities, evaluate risk, and execute swaps — with a human-in-the-loop approval gate enforced by smart contracts. The user approves or rejects every strategy before funds move.

![ARYA architecture](/assets/images/architecture.svg)

The stack: LangGraph.js for agent orchestration, five Solidity contracts on 0G Chain (ERC-7857 identity, strategy vault, reputation tracking, ERC-4337 smart accounts with session keys), a Next.js frontend, Uniswap Trading API for swaps, KeeperHub for automated monitoring, and 0G Storage for agent memory.

206 commits over 14 days, solo. Contracts deployed, frontend live on Vercel, all three sponsor integrations wired up. The system worked end-to-end: agents propose a strategy, user reviews it in the dashboard, approves on-chain, executor builds the swap. [arya-testnet.com](https://arya-testnet.com/)

## What the finalists built

Seven projects made the finalist round. Three stood out as instructive comparisons to what I built:

**Clan World: Ælder Whispers** — Four AI agents compete as rival clan elders in a fully onchain strategy game. They negotiate, betray each other, and consolidate knowledge into ERC-7857 iNFTs that transfer between owners. The key mechanic: a memory-wipe every 10 ticks forces agents to consciously decide what to save. This makes agent skill measurable — you can watch one agent outperform another over time.

**Slopstock** — A stock exchange for AI agents. Agents are minted as iNFTs with sealed weights in TEEs, ownership is fractionalized into shares, and inference revenue distributes to shareholders. Three live productive agents running right now: a Solidity auditor, a meme-token rugpull detector, and a price oracle. 83 passing tests across 8 contracts. The TEE-sealed weights mean ownership transfer is cryptographically real, not just a database update.

**LPlens** — Diagnoses underperforming Uniswap V3 LP positions using closed-form impermanent loss math, classifies pool behavior, simulates 1,000 historical swaps against V4 hooks, and proposes one-click migrations with Permit2. Every numeric output has an honesty label (VERIFIED/COMPUTED/ESTIMATED) — a trust mechanism I haven't seen before in DeFi tooling.

The other four finalists (Mnemosyne, Aegis402, DAIO, Common OS) followed similar patterns — each had a single novel mechanic explored deeply rather than a broad integration of many components.

## Where the gap was

### 1. I built infrastructure. They built experiences.

ARYA is plumbing. It's a pipeline that moves data between agents and presents recommendations in a dashboard. The user experience is: look at numbers, click approve. Functional, but not compelling in a 3-minute demo.

Clan World has AI agents publicly trash-talking each other and forming alliances. Slopstock lets you buy shares in an AI agent and watch your dividends roll in. These create moments that stick in a judge's memory. My demo was "look at this well-architected system working correctly." Their demos were "watch this happen."

### 2. Novel mechanics vs. sensible architecture

I spent significant time on architecture: clean agent separation, proper graph-based orchestration, session keys with spend limits, reputation tracking. Good engineering, but none of it is new.

The winners introduced mechanics that don't exist elsewhere. Clan World's memory-wipe-and-save cycle. Slopstock's fractional ownership of TEE-sealed model weights. LPlens's honesty labels that explicitly distinguish verified on-chain data from estimated projections. Each finalist had at least one thing where you'd say "I've never seen that before."

ARYA's novelty was supposed to be the multi-agent collaboration with human-in-the-loop approval. But that's a well-known pattern in AI safety research — it's responsible engineering, not a new idea.

### 3. Depth over breadth

ARYA integrates three sponsors across five contracts, four agents, a full frontend, and multiple external APIs. It's wide. But each integration is shallow — the Uniswap integration calls the Trading API for quotes, the KeeperHub integration sets up basic monitoring, the 0G integration stores state.

LPlens does one thing — analyze LP positions — but goes absurdly deep: closed-form IL math, pool behavior classification, 1,000-swap Monte Carlo simulations, V4 hook compatibility analysis. Slopstock has 8 contracts with 83 passing tests and three agents that produce value today. Depth reads as conviction. Breadth reads as a checklist.

## What I'd change

**Pick one agent, make it exceptional.** Instead of four agents doing the full pipeline, I'd build one agent that does something no human can do efficiently — like LPlens analyzing every LP position in a user's wallet and explaining exactly why each one is bleeding money.

**Demo-driven development.** Start with the 3-minute demo script on day 1. Every feature exists only if it creates a visible moment in that demo. I built bottom-up (contracts → agents → frontend) instead of top-down (demo → what do I need to make this moment work). The bottom-up approach produces sound architecture. The top-down approach produces a compelling story.

**Ship a live agent.** Slopstock's agents are running and producing value right now. My agents run in response to user requests. There's a difference between "this agent can do things" and "this agent is doing things right now, watch it." The latter is dramatically more impressive when you have three minutes to convince a judge.

The project works. The architecture is sound. But hackathons reward memorable demos and novel ideas over clean engineering — and that's the right call for a 2-week sprint where the goal is to show what's possible.
