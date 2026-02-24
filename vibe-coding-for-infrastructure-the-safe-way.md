# Vibe Coding for Infrastructure the Safe Way

Most engineers are already copying infra code from LLMs. LLMs are the new Stack Exchange. The question is what happens with that code. In the worst case, the engineer pushes a hallucinated script straight to production. We've seen that. In the best case, the engineer copies it, spins up a machine or cluster, tries it, and when it fails, goes back to the LLM to fix it. Rinse and repeat. It works, but it's time consuming. This is vibe coding for ops. This can be automated easily. So we thought.

I started building a vibe coding engine for ops and infra during the Christmas holidays (with help from a few friends). It was supposed to take maximum two weeks. It turned out to be more complicated than just vibing some Node.js code.

With frontend code, the solution space is broad and subjective. The LLM can always produce something that works or looks good enough. A button can be styled a hundred different ways and still be correct. With infrastructure, there's no "good enough." The service either starts or it doesn't. The cluster either converges or it doesn't. The firewall rule is either correct or traffic is blocked.

## The Problem

AI-generated infrastructure code looks correct. It passes syntax checks. It even passes linting. But when it runs on a real server, it fails in ways static analysis can't predict. Misconfigured services, missing dependencies, silent permission failures, race conditions during provisioning.

Gartner predicts that the next great infrastructure failure may not be caused by hackers or natural disasters but by a well-intentioned engineer and a flawed automation script.

The surface area is massive. Enterprise infrastructure isn't just servers. It's Vault, Jenkins, networking, storage, CI/CD pipelines, container orchestration, each with its own quirks and failure modes. An LLM that writes a perfect Ansible playbook for Ubuntu might generate something subtly broken for AlmaLinux. Multiply that across every tool and platform in a typical enterprise and "sometimes correct" becomes a serious risk.

The problem isn't that LLMs are bad at writing infrastructure code. They're actually impressive. The problem is that "impressive" and "reliable" are not the same thing. An 85% success rate sounds great until you realize that means 15 out of every 100 deployments fail in ways you didn't expect.

And the engineers most at risk aren't beginners. They're the senior DevOps engineers who glance at LLM output, think "yeah, that's roughly what I would have written," and push it. Their confidence is calibrated to code they wrote, not code they reviewed. A junior engineer who doesn't fully understand the playbook will ask someone to check it. A senior engineer who pattern-matches it against ten years of experience will trust it and move on. The LLM didn't learn Ansible the way they did. It learned it from Stack Overflow answers, outdated docs, and other LLM outputs. The code converges on something that looks correct without being correct. Expertise becomes a liability when it makes you a faster, more confident skipper of the verification step.

## Convergence: Let It Fail Safely

We took a different approach. Instead of demanding perfection from the LLM on the first try, expecting humans to provision clusters and fix hallucinated code, or simply pushing it to the production team and hoping for the best, we built an environment where failure is safe, expected, and productive.

The engine spins up ephemeral clusters of up to four real virtual machines, not containers, not simulators. An Ansible task might need a control node and two managed nodes. A load balancer test needs three or four VMs with distinct IPs. Every retry tears down the entire cluster and spins up a fresh one. No leftover state from a previous failed attempt. No patched-together environment. Clean slate every time. Getting a single VM to boot in under one second was a challenge on its own. We went through different techniques: direct kernel boot, IP injection, microVMs. We also tried different hypervisors from Firecracker, to Kata Containers, to QEMU, before landing on something reliable. Subsecond VM launch matters because when every retry means a brand new cluster, slow boot times kill the entire loop. No one wants to spend 5 minutes spinning up a cluster to apply a config that takes 10 seconds.

The AI-generated code runs on the cluster. If it fails, the engine has two recovery paths.

The first is retry. The engine sends the full code and error logs back to the LLM and asks it to analyze what went wrong and produce a corrected version. This works well for straightforward issues like missing packages or misconfigured services.

The second is troubleshooting. Instead of asking for a full rewrite, the engine works interactively with the LLM to diagnose the problem in real time. The LLM says "run ls -la /etc/nginx." The engine runs it and sends back the result. The LLM says "the directory doesn't exist, run mkdir -pv /etc/nginx." Step by step, the LLM and engine work together like an engineer SSHing into a broken server. The engine decides which path to take based on the nature of the failure.

Behind the scenes, this isn't a single AI doing everything. Multiple specialized LLMs collaborate: one designs the test environment, one generates code, one evaluates results, one troubleshoots, one decides whether to retry or investigate further. Each has a narrow job and limited context.

Once the task converges, the engine synthesizes a single, clean, verified script and runs it from scratch on a fresh cluster to prove it works in one pass. The output is never a chain of fixes. It's a distilled script that works from zero.

## How to Use It

Antrieb is available as an MCP server, meaning it plugs directly into any MCP-compatible client like Claude Desktop, Cursor, or Continue. You describe what you want in natural language, and the engine handles everything: environment design, code generation, execution, convergence. Alternatively, you can submit code generated with another LLM and ask the engine to deploy it and return findings. The simplest way to try Antrieb is simply to log into Grafna. And within it, you can try examples selected from a curated set just by clicking. 

Every run is fully observable through Grafana. You can see logs, failure analysis, evaluation results, number of iterations, execution timings, which recovery path the engine chose, and why. Nothing is a black box. If a task took three rounds to converge, you can see exactly what failed in round one, what the LLM tried differently in round two, and what finally worked in round three.

## What Testers Are Saying

We've been running this as a research project with a small group of DevOps engineers. Here's what they're reporting.

One tester wrote: "As a DevOps Engineer I am very careful not to deploy unverified code into production. If a sandbox is not available then I am left parsing the code line by line and then scheduling an additional code review by a peer to look for errors. Even this strategy only works 90% of the time and is still time consuming. A tool such as this will allow me to provide less manual oversight to the AI generated code and, hopefully, get to production more quickly and more safely."

Another described a specific test: "The 3-node nginx load balancer test spun up three Ubuntu VMs with distinct IPs, generated a script that configured each role correctly, and verified it all in 38 seconds. The Ansible test also set up a proper 3-node cluster (1 controller + 2 targets). Doing this manually with Vagrant would take me 30-60 minutes."

## Try It. Break It.

Engineers are going to keep pasting LLM-generated infrastructure code. That's not a prediction — it's already happening. The real question isn't whether AI writes infra code. It's whether failures are discovered in minutes or in production.

The problem space is enormous. Every OS, tool, version, and configuration multiplies the surface area. No single model will get it right on the first pass. "Generate and hope" means the first real test is your staging cluster — or worse.

Antrieb collapses that loop. It spins up real machines, runs the code, tears everything down, and retries until it converges. What would normally take hours of manual provisioning and debugging becomes a controlled feedback cycle measured in seconds.

We're still early. Some tasks break it. Some edge cases we haven't seen yet. Good. Try it. Break it. Tell us where it fails.

[https://dash.antrieb.sh]
