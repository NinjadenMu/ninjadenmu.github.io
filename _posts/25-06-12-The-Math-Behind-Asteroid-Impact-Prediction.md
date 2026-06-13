---
title:  "The Math Behind Asteroid Impact Prediction - Will the Flat-o-Saurs Go Extinct?"
date:   2025-06-11
categories: 
  - applied-computer-science
  - high-school
author: Jaden Mu
mathjax: true
---
65 million years ago, the dinosaurs were wiped out by an asteroid.  The Flat-o-saurs are like the dinosaurs, but they live on a 2D planet called plane-et (which makes the math a bit cleaner in this article).  Dr. Bi-ceratops, the 2D cousin of the 3D Triceratops, has set up an asteroid detection and monitoring system to try to avoid his cousin's tragic fate.

Oh No!  Dr. Bi-ceratops has detected an asteroid!  In this paper, I'll explain how Dr. Bi-ceratops' detection and monitoring system works, and how he can calculate the probability of the newly discovered asteroid hitting plane-et.   

To find asteroids that might be a threat, Dr. Bi-ceratops has set up cameras pointed at the night sky.  Sophisticated software analyzes the images from these cameras, and flags fast moving objects (which are likely to be asteroids).

![Fast Moving Asteroid in Scan](/assets/images/asteroids/detection.gif)

Because the cameras only capture an image, they can calculate the angle of the asteroid relative to plane-et, but can't (easily) find the distance of the asteroid to plane-et (the image has no depth information associated with it.)

![Illustration of Localization Problem](/assets/images/asteroids/localization_problem.png)

Before Dr. Bi-ceratops can predict whether the asteroid will hit plane-et, he needs more information about the asteroid. To fully describe an asteroid in 2-D space, Dr. Bi-ceratops needs its $$x$$ and $$y$$ coordinates, as well as its $$x$$ and $$y$$ velocities. From his one observation, all Dr. Bi-ceratops knows is $$\theta$$. He knows that $$x$$ and $$y$$ are related to $$\theta$$ by $$x = r \cos(\theta)$$ and $$y = r \sin(\theta)$$ where $$r$$ is the distance from plane-et to the asteroid. Unfortunately, he still doesn't know $$r$$, $$v_x$$, or $$v_y$$.

Luckily, Dr. Bi-ceratops can make multiple observations!  

![Illustration of Multiple Observations](/assets/images/asteroids/multiple_observations.png)

How can he use these multiple observations to help him find the unknown $$r, v_x$$, and $$v_y$$?  He can start with an estimate, and refine it until the estimated asteroid is consistent with every observation.

Let's generate an asteroid with random $$r, v_x$$, and $$v_y$$ at $$t=0$$, and simulate its motion according to Newton's laws of motion.

![Illustration of Residuals of Propagated Orbit vs. Observations](/assets/images/asteroids/residuals.png)

Hmm...this specific combination of $$r, v_x$$, and $$v_y$$ is not consistent with all the observations (it has large residuals).  This means we probably haven't found the values that best characterize the asteroid.  However, if we find some combination of $$r, v_x$$, and $$v_y$$ at $$t=0$$ that has small residuals, we'll know we've found a good characterization of the asteroid.

Formally, we want to find the combination of $$r_{0}, v_{x_0}, v_{y_0}$$ which when propagated forward in time using Newton's laws of motion, minimize the weighted squared residuals relative to all observations.

Let's define the following cost function *Q*:

$$Q=\hat{e}^TW\hat{e}$$ for the residual vector $$\hat{e}=[e_0, e_1, ..., e_i, ... e_n]^T$$, where $$e_i$$ is the residual for observation $$i$$.

As you can see, $$Q$$ is a **quadratic** function of the residuals, although the form is a bit funky because $e$ is a vector.  $$W$$ is a weighting matrix, since some residuals may be less important than others to minimize.  If all residuals are weighted the same, $$W$$ would be the identity matrix.

So, Dr. Bi-ceratops needs to find the initial state of the asteroid which when propagated forward in time, minimizes $$Q$$.  How can $$Q$$ be minimized?  Harkening back to Calculus 101, Dr. Bi-ceratops vaguely remembers that a minima of a function occurs when its derivative (or in this case, gradient) is 0.  P.S.: Typically, we wouldn't know if the critical point was a min or max, but it turns out this cost function is highly convex in practice.

Let's define the initial state of the asteroid as $$X_0$$, where $$X_0$$ = $$[r_0, v_{x_0}, v_{y_0}]^T$$.  Note that $$X$$ is the entire characterization of the asteroid, not just its $$x$$ coordinate.

Formally, we need to find the value of $$X_0$$ where $$\frac{\delta Q}{\delta x_0}=0$$.  To do this, we'll start with an initial random (and probably bad) estimate of $$X_0$$.  Then, we'll refine $$X_0$$ by repeatedly changing it by some small $$\delta x$$ until we've found the $$X_0$$ where $$\frac{\delta Q}{\delta X_0}=0$$.

Now, how do we find the right $$\delta x$$ to change $$X_0$$ by?  Let's examine what happens to the residuals when we change $$X_0$$ by some small $$\delta x$$.  We assume $$e(X_0)$$ is locally linear (remember, $$e$$ represents our residuals).  Then, $$e(X_0+\delta x)\approx e(X_0) + B\delta x$$, where $$B$$ is the Jacobian $$\frac{\delta Q}{\delta x_0}$$.  If this is a bit confuzzling, scroll down and Dr. Bi-ceratops will explain it better.

![A Better Explanation of the Math Above](/assets/images/asteroids/better_explanation.png)

This in turn means that $$Q(X_0 + \delta x)=e(X_0+\delta x)^TWe(X_0+\delta x)$$

$$= e^TWe + 2\delta x^TB^TWe + \delta x^TB^TWB\delta x$$ after expansion.

Whew! That expansion took me quite a while. But it was worth it, because if you look closely at the expanded expression, we see that the change in $$Q$$ with respect to a small change in $$X_0$$ is:

$$2B^TWe + B^TWB\delta x$$

Wait a second! A small change in $$Q$$ with respect to a small change in $$X_0$$? That's $$\frac{\delta Q}{\delta X_0}$$!

We need that to be 0, so we try:

$$2B^TWe + B^TWB\delta x = 0$$

which rearranges to:

$$\delta x = -C^{-1}B^TWe$$

where $$C = B^TWB$$

Note that $$C$$ is also the second-order partial derivatives of Q (C is the Hessian of Q).

So it turns out that:

$$\delta x = -C^{-1}B^TWe$$

This means that by updating $$X_0$$ by $$-C^1B^TWe$$ repeatedly, we'll eventually refine our characterization of the asteroid into something that's compatible with all observations!

![Illustration of minimized residuals](/assets/images/asteroids/small_residuals.png)

So we've found a $$X_0$$ that is the best characterization of the asteroid.  However, because there's some uncertainty in our measurements, and because the $$X_0$$ we found still doesn't lead to a path that fits perfectly with all observations,$$X_0$$ should actually be represented as a multivariate probability distribution centered at the best singular $$X_0$$ value we found.  From now on, I'll use $$X_0$$ to represent the probability distribution, and $$X_{best}$$ to represent the singular value we found previously.

Let's assume $$X_0$$ is Gaussian.  A multivariate Gaussian takes the form: 
![Form of Multivariate Gaussian](/assets/images/asteroids/multivariate_gaussian.png)
$\Sigma$ represents the covariance matrix of the distribution.  

We already know $$\mu$$ is $$X_{best}$$, so what we need to find now is the covariance $$\Sigma$$.

Intuitively, some $$X$$ ~ $$X_0$$ with a high cost ($$Q$$) has a lower probability than a $$X$$ with a low cost, since it is less compatible with the observations.  

We'll formalize this idea as $$Q(X)\propto-\log(P(X)) + C$$ for some constant $$C$$.  This means that a $$X$$ with a high cost has a probability approaching 0, and a $$X$$ with a low cost has a probability approaching 1.

Rearranging, $$P(X)\propto e^{-Q(X)}$$.

After some further calculation, we can use this to extract a covariate matrix for $$X_0$$.  

Then, we'll be able to get a distribution for something like $$(v_{x_0}, v_{y_0})$$ that looks like the following:
![Illustration of Confidence Ellipses](/assets/images/asteroids/ellipses.png)

If Dr. Bi-ceratops were a 3 dimensional creature instead of a 2 dimensional creature, he'd be able to see that the actual distribution looks like a 3D ellipsoid describing the uncertainty in $$r_0, v_{x_0}$$, and $$v_{y_0}$$!

Now that the asteroid is pinpointed, it's pretty easy*** to calculate the probability of impact with plane-et.  Just sample a bunch of asteroids from the probability distribution, simulate their paths using Newton's laws of motion, and check if they collide with plane-et, and we're done!

![Dinosaurs saying Thank You :)](/assets/images/asteroids/dino_thank_you.png)

\* What I described for calculating probabilities is really slow cause you have to sample a lot of asteroids - there are much better and faster ways that I would write about but the flat-o-saurs are after me and I don’t have tim-

