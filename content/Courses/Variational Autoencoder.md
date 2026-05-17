---
title: Variational Autoencoder
category: courses
tags:
  - cs285
draft: "true"
---
## Background

Encoder:
$$q_\phi(z|x)=\mathcal{N}(\mu_\phi(x),\sigma_\phi(x))$$
Decoder:
$$p_\theta(x|z)=\mathcal{N}(\mu_\theta(z),\sigma_\theta(z))$$
![[Pasted image 20260516091627.png]]
why does this work?
$$\mathcal{L}_i=\mathbb{E}_{z\sim q_\phi(z|x_i)}[\log p_\theta(x_i|z)]-D_{KL}(q_\phi(z|x_i)||p(z))$$
The encoder has a very strong incentive to produce z that look like samples from the prior because if it doesn't do that then that KL Divergence term will be very large and that will incur a large penalty, this explains why samples from the encoder will be within that unit variance prior, it doesn't by itself explain why any sample from the unit variance prior will be close to something that is being encoded the argument for that has to do with efficiency the encoder also wants to have a variance close to one because that's what the prior has. 

So the encoder wants to be pretty Frugal in its use of the Latent space which means that it really wants to use every piece of the latent space, if there's some piece of the Latent space that's unused, it'll be better for the encoder to expand into those spaces and increase its variance so that its variance can be closer to one. So as a result, you end up with a mapping between these $z$ and $x$ where pretty much every $z$ that you sample from the unit variance, prior will map to some valid $x$.

**Representation learning**

why we expect these $z$ to be better state representations than the states themselves? Because a variational auto encoder learns these $z$ representations that satisfy an independent Gaussian prior meaning every dimension of z is independent of every other dimension should lead to better disentanglement of the underlying factors of variation than the images themselves.

**Conditional models**
$$\mathcal{L}_i=\mathbb{E}_{z\sim q_\phi(z|x_i,y_i)}[\log p_\theta(y_i|x_i,z)+\log p(z|x_i)]+\mathcal{H}(q_\phi(z|x_i,y_i))$$


Learn state space model and plan in the latent space

## Key points



## Explanation in my own words



## Why it matters



## Related



## Reference