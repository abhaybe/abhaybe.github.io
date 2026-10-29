---
layout: post
title: "LLM-Guided RL: Help When Stuck, and Reward Shaping Written by an LLM"
description: "Two cheap ways to put an LLM inside RL training. With PPO on sparse-reward MiniGrid tasks, success rose from 5% to 90% on DoorKey and from 0% to 30% on MultiRoom, for about ten cents of API calls per run."
reading: 7 min read
---

Sparse-reward RL is mostly wasted time. On a task like MiniGrid's DoorKey, a PPO agent has to find a key, pick it up, walk to a locked door, open it, and reach a goal, and it gets exactly one reward signal: +1 at the end. Until it stumbles onto that sequence by chance, there is nothing to learn from. In my runs, plain PPO solved DoorKey-8x8 only 5% of the time after 300K steps.

A language model, on the other hand, knows what a key is for. But LLMs make poor policies: they hallucinate, don't explore systematically, and are slow and expensive to query every step.

For a project in MIT's 6.7920 (Deep Reinforcement Learning), I tried a middle ground: a normal PPO agent does all the learning, and an LLM helps in two narrow, cheap places. This post covers what I did, what worked, and what I would not trust in my own results.

## Two ways to use the LLM

**1. Help when stuck.** If the agent goes k = 50 steps without any reward, we pause and ask the LLM for a single action. The prompt contains the task description, the agent's position and facing direction, what it is carrying, the object directly in front of it, its last 10 actions, and the list of valid actions. The suggested action replaces the agent's own action for that one step, and the stuck counter resets. We allow at most 3 help calls per episode, and in practice the LLM fires on fewer than 2% of environment steps.

Responses are parsed with a strict `ACTION: <name>` format, then a keyword search, then a small synonym map; the 4% that still fail fall back to the agent's own action.

**2. Reward shaping written by the LLM.** Once, before training starts, we give the LLM the task description, a short description of the environment API (agent position, what it's carrying, how to query grid cells and door state), two example shaping functions, and a list of constraints (no imports, no file or network access). It returns a Python function `shaped_reward(env, prev_carrying)` that gets called every step and added to the environment reward:

r′ₜ = rₜ + λ · φ(sₜ), with λ = 1.

The generated code goes through an AST parse, a check for blocked operations, execution with restricted builtins, and a test call; if any step fails, we fall back to a simple default. After that there are no more API calls.

The two methods differ in kind. Help-when-stuck is reactive and local: it fixes one bad moment. Reward shaping is proactive and global: it turns one distant reward into a trail of intermediate ones.

## Setup

The agent is PPO (Stable-Baselines3) with a ~20K-parameter CNN over MiniGrid's 7×7 egocentric view, with identical hyperparameters across conditions and 300K environment steps per run. The LLM was `gpt-4o-mini`. The three tasks are **Empty-8x8** (walk to a goal; a sanity check), **DoorKey-8x8** (key, door, goal; short but strictly sequential), and **MultiRoom-N6** (six connected rooms; long horizon). The four conditions are PPO alone, + help-when-stuck, + LLM-written shaping, and both. Each policy was evaluated on 50 episodes with deterministic actions.

## Results

<div class="table-wrap" markdown="1">

| Task | PPO | Help | Shaping | Both |
|---|---|---|---|---|
| Empty | 100% | 100% | 100% | 100% |
| DoorKey | 5% | 82% | 86% | **90%** |
| MultiRoom | 0% | 0% | 28% | **30%** |

</div>
<p class="caption">Success rate after 300K steps, 50 evaluation episodes. Tasks are Empty-8x8, DoorKey-8x8, and MultiRoom-N6. On Empty-8x8, PPO converged in 150K steps and PPO + help in 120K.</p>

A few things stand out.

**DoorKey is where help-when-stuck shines.** Going from 5% to 82% with only an occasional hint makes sense once you look at where the agent gets stuck. DoorKey's stuck states are easy to read: facing a locked door with a key in hand, or standing next to a key with empty hands. The LLM recognizes those situations and gives the obvious next action. The most common suggestions were turn right (31%), toggle (24%), and pickup (18%), and 72% of suggestions were followed by a reward within 10 steps.

**MultiRoom is where it fails.** Help-when-stuck alone scored 0% on MultiRoom, the same as the baseline. Being stuck in a six-room maze is ambiguous from a 7×7 view: the LLM can't see the map, so "turn right" is no better than a guess. What MultiRoom needs is a dense signal over a long horizon, and only the shaping condition provided one (28%).

**The two methods are complementary.** Combining them gave the best result on both hard tasks (90% and 30%). Shaping gives the agent a sense of direction throughout training; help-when-stuck occasionally gets it out of a specific hole. In combined runs the help mode also fired less often, since the agent was getting shaped reward and so was less often "stuck" by the 50-step definition.

**Even on the easy task, hints speed things up.** On Empty-8x8, everything converges to 100%, but help-when-stuck got there in 120K steps instead of 150K.

The whole thing cost about $0.05–0.10 in API calls per training run.

## Limitations and what I'd do next

This was a class project, and the numbers above are less solid than a results table makes them look. Here is what I would want a careful reader to know, roughly in order of how much it matters.

**Everything is a single seed.** Each cell in the table is one training run. RL results on sparse-reward tasks are notoriously high-variance across seeds, so the gaps between 82%, 86%, and 90% on DoorKey are well within what seed noise could produce. The big gaps (5% vs. 80–90%, 0% vs. ~30%) are more likely to be real, but I can't claim even those with confidence without 5–10 seeds per condition and confidence intervals.

**Evaluation used the best checkpoint, not the final one.** Each condition was scored using the best model seen during training, as picked by periodic evaluation. That is an optimistic estimate, especially for a noisy method: a run that briefly spiked and then collapsed would still look good. The fair comparison is final-checkpoint performance, or best-checkpoint selection on a separate set of evaluation seeds from the ones used for reporting.

**The shaping isn't potential-based.** I added the LLM's bonus directly to the reward. Ng, Harada, and Russell (1999) showed that shaping preserves the optimal policy only if it has the form F(s, s′) = γΦ(s′) − Φ(s) for some potential function Φ over states. Arbitrary bonuses can change what the optimal policy is. My setup has a concrete version of this problem: the door bonus is paid on *every step* the door is open, not once when it opens. With 100-step episodes, an agent that opens the door and then wanders around can collect more shaped reward than the +1 for reaching the goal. The 90% success rate suggests the agent didn't exploit this here, but nothing in the setup prevents it. The fix is straightforward: have the LLM write Φ (for example, "has key" + "door open" + negative distance to the next subgoal) and compute the bonus as γΦ(s′) − Φ(s).

**Help-when-stuck makes those transitions off-policy.** The override happens inside an environment wrapper, so PPO's rollout buffer records the action the policy sampled (and its log-probability), while the environment actually executed the LLM's action. The policy gets credit or blame for an action it didn't take. With fewer than 2% of steps affected the damage is probably small, but it's a real bias in an algorithm that assumes on-policy data. Better options are to store the executed action with its probability under the current policy and let PPO's clipping bound the importance ratio, to mask overridden steps out of the policy loss and use them only for the value function, or to treat hints as demonstrations with a separate imitation term.

**The DoorKey example in the prompt was too close to the answer.** One of the two worked examples I gave the LLM was already a DoorKey-style function (+0.15 for picking up a key, +0.15 for an open door). So on DoorKey, the "LLM-written" shaping is partly a hand-written shaping passed through the LLM. MultiRoom is the cleaner test of whether the LLM can come up with useful structure on its own. A fair version would use examples from unrelated tasks.

Next steps, in order: rerun every condition with 5–10 seeds and report final-checkpoint means with intervals; switch to potential-based shaping and check whether the MultiRoom gains survive; fix the off-policy handling of overrides; use neutral prompt examples; and only then try harder environments and other LLMs.

## Takeaway

The idea I'd keep is the division of labor: the LLM supplies prior knowledge in small, cheap doses, either once at the start (as code) or rarely during training (as a hint), and the RL agent still does the learning. Which dose helps depends on the task. Hints work when stuck states are easy to interpret, and shaping works when the problem is a long horizon. Together they took sparse-reward tasks that PPO couldn't solve to ones it mostly could, for $5–10 of API calls per hundred training runs. Whether those effect sizes hold up under multiple seeds and principled shaping is the experiment I'd want to run next.

Code and the full report are at [github.com/abhaybe/babyai-llm](https://github.com/abhaybe/babyai-llm).
