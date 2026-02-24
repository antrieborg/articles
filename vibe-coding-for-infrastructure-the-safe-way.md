# Vibe Coding for Infrastructure the Safe Way

[Summer Yue, Meta's director of AI alignment, told her OpenClaw agent to check her email and suggest what to delete.](https://www.ndtvprofit.com/technology/i-couldnt-stop-it-how-openclaw-tried-to-trash-meta-ai-alignment-directors-emails-11128276) The agent started mass-deleting everything in what she called a "speed run," ignoring her stop commands from her phone. She had to physically run to her Mac Mini to kill it. An AI safety researcher, whose job is making sure AI systems don't go rogue, couldn't stop an AI agent from nuking her inbox.

That was email. Now imagine it's your production Kubernetes cluster. Your Terraform state. Your Ansible playbooks running against live servers.

This isn't hypothetical. Engineers are already pasting LLM-generated infrastructure code into real systems every day. LLMs are the new Stack Exchange. The question isn't whether AI writes infra code. It's what happens when that code runs unsupervised against something that matters. In the worst case, the engineer pushes a hallucinated script straight to production. We've seen that. In the best case, they spin up a machine, try it, watch it fail, go back to the LLM to fix it, and repeat. That works, but it's slow. It's vibe coding for ops.

We thought this can be automated easily. It turned out to be more complicated than just vibing some Node.js code.

With frontend code, the solution space is broad and subjective. The LLM can always produce something that works or looks good enough. A button can be styled a hundred different ways and still be correct. With infrastructure, there's no "good enough." The service either starts or it doesn't. The cluster either converges or it doesn't. The firewall rule is either correct or traffic is blocked.

## The Problem

AI-generated infrastructure code looks correct. It passes syntax checks. It even passes linting. But when it runs on a real server, it fails in ways static analysis can't predict. Misconfigured services, missing dependencies, silent permission failures, race conditions during provisioning.

The surface area is massive. Enterprise infrastructure isn't just servers. It's Vault, Jenkins, networking, storage, CI/CD pipelines, container orchestration, each with its own quirks and failure modes. An LLM that writes a perfect Terraform module for AWS might generate something subtly broken for GCP. An Ansible playbook that works on Ubuntu might fail silently on AlmaLinux. Multiply that across every tool and platform in a typical enterprise and "sometimes correct" becomes a serious risk.

The problem isn't that LLMs are bad at writing infrastructure code. They're actually good at it. The problem is that "good" and "reliable" are not the same thing. An 85% success rate sounds great until you realize that means 15 out of every 100 deployments fail in ways you didn't expect.

And the engineers most at risk aren't beginners. They're the senior DevOps engineers who glance at LLM output, think "yeah, that's roughly what I would have written," and push it. Their confidence is calibrated to code they wrote, not code they reviewed. A junior engineer who doesn't fully understand the script will ask someone to check it. A senior engineer who pattern-matches it against ten years of experience will trust it and move on.

That's exactly what happened with Yue. She tested OpenClaw on a small inbox and it worked well. It earned her trust. So she pointed it at the real thing. The pattern is the same whether it's email or infrastructure: the tool works on the easy case, you extrapolate that it works generally, and the failure hits where the stakes are higher. Yue's job title is literally "director of alignment." If expertise in AI safety doesn't protect you from misplaced trust in AI output, expertise in Bash and CloudFormation won't either.

The LLM didn't learn infrastructure the way you did. It learned it from Stack Overflow answers, outdated docs, and other LLM outputs. The code converges on something that looks correct without being correct. Expertise becomes a liability when it makes you a faster, more confident skipper of the verification step.

## Convergence: Let It Fail Safely

The core idea is simple. Instead of demanding perfection from the LLM on the first try, or expecting humans to provision clusters and fix hallucinated code, or pushing it to the production team and hoping for the best, you build an environment where failure is safe, expected, and productive. You let the code fail, and you make the failure loop fast enough that convergence is cheap.

To make that work, the engine spins up ephemeral clusters of up to four real virtual machines. Not containers, not simulators. An Ansible task might need a control node and two managed nodes. A Terraform plan might need a target VM and a cloud endpoint. A load balancer test needs three or four VMs with distinct IPs. Every retry tears down the entire cluster and spins up a fresh one. No leftover state from a previous failed attempt. No patched-together environment. Clean slate every time.

Getting a single VM to boot in under one second was a challenge on its own. We went through different techniques: direct kernel boot, IP injection, microVMs. We also tried different hypervisors from Firecracker, to Kata Containers, to QEMU, before landing on something reliable. Subsecond VM launch matters because when every retry means a brand new cluster, slow boot times kill the entire loop. No one wants to spend 5 minutes spinning up a cluster to apply a config that takes 10 seconds. In practice, a 3-node nginx load balancer test (spin up three Ubuntu VMs with distinct IPs, generate the config for each role, verify everything) converges in about 38 seconds. Setting up the same thing manually with Vagrant takes 30 to 60 minutes.

The AI-generated code runs on the cluster. If it fails, the engine has two recovery paths.

The first is retry. The engine sends the full code and error logs back to the LLM and asks it to figure out what went wrong and produce a corrected version. This works well for things like missing packages or misconfigured services.

The second is troubleshooting. Instead of asking for a full rewrite, the engine works interactively with the LLM to diagnose the problem in real time. The LLM says "run ls -la /etc/nginx." The engine runs it and sends back the result. The LLM says "the directory doesn't exist, run mkdir -pv /etc/nginx." Step by step, the LLM and engine work together like an engineer SSHing into a broken server. The engine decides which path to take based on the nature of the failure.

Once the task converges, the engine synthesizes a single clean script and runs it from scratch on a fresh cluster to prove it works in one pass. The output is never a chain of fixes. It's a distilled script that works from zero.

## How to Use It

Antrieb is available as an MCP server, meaning it plugs directly into any MCP-compatible client like Claude Desktop, Cursor, or Continue. You describe what you want in natural language, and the engine handles the rest: environment design, code generation, execution, convergence. You can also submit code you generated with another LLM and ask the engine to deploy it and return findings. Yet, the simplest way to try Antrieb is to log in to [Grafana](https://dash.antrieb.sh) and there you can try examples selected from a curated set just by clicking.

<img width="1913" height="862" alt="Screenshot 2026-02-24 at 16-39-06 Set up HAProxy load balancing to 2 nginx backend n - Runs - Dashboards - Grafana" src="https://github.com/user-attachments/assets/6a1dc533-bf58-4d50-9ff5-52640c838af9" />


Every run is fully observable through Grafana. You can see logs, failure analysis, evaluation results, number of iterations, execution timings, which recovery path the engine chose, and why. Nothing is a black box. If a task took three rounds to converge, you can see exactly what failed in round one, what the LLM tried differently in round two, and what finally worked in round three.


## Try It. Break It.

Engineers are going to keep pasting LLM-generated infrastructure code. That's not a prediction, it's already happening. The real question isn't whether AI writes infra code. It's whether failures are discovered in minutes or in production.

The problem space is enormous. Every OS, tool, version, and configuration multiplies the surface area. No single model will get it right on the first pass. "Generate and hope" means the first real test is your staging cluster, or worse.

Antrieb collapses that loop. It spins up real machines, runs the code, tears everything down, and retries until it converges. What would normally take hours of manual provisioning and debugging becomes a controlled feedback cycle measured in seconds.

[We're still early. Some tasks break it. Some edge cases we haven't seen yet. Good. Try it. Break it. Tell us where it fails.](https://dash.antrieb.sh)

