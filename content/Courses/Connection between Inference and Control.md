---
title: Connection between Inference and Control
category: courses
tags:
  - cs285
draft: "true"
---
## Background Notes

1. Does reinforcement learning and optimal control provide a reasonable model of human behavior?
2. Is there a better explanation?
3. Can we derive optimal control, reinforcement learning, and planning as *probabilistic inference*?
4. How does this change our RL algorithms?

Goals:
- Understand the connection between inference and control
- Understand how specific RL algorithms can be instantiated in this framework
- Understand why this might be a good idea

- $\beta_t(s_t,a_t)=p(\mathcal{O}_{t:T}|s_t,a_t)$: backward message tells you what is the probability of being optimal now until the end of the trajectory given the state and action that you are in.
- $p(a_t|s_t, \mathcal{O}_{1:T})$ The policy is the probability of an action at timestep t given the state of timestep t and given the evidence that the entire trajectory from 1 to T is optimal
- $\alpha_t(s_t)=p(s_t|\mathcal{O}_{1:t-1})$ forward message says what's the probability of landing in particular state of $s_t$ if you are optimal through timestep t - 1  


$$\log p(\mathbf{x})\ge\mathbb{E}_{\mathbf{z}\sim q(\mathbf{z})}[\log p(\mathbf{x},\mathbf{z})-\log q(\mathbf{z})]$$


**Review**
- Reinforcement learning can be viewed ad inference in a graphical model
	- Value function is a backward message
	- Maximize reward and entropy (the bigger the rewards, the less entropy matters)
	- Variational inference to remove optimism
- Soft Q-learning
- Entropy-regularized policy gradient

Soft optimality suggested readings:

## Key points



## Explanation in my own words



## Why it matters



## Related



## Reference