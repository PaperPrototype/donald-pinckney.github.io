---
layout: bookpost
title: Feature Scaling
date: 2018-11-15
categories: TensorFlow
isEditable: true
editPath: books/tensorflow/src/ch2-linreg/2018-11-15-feature-scaling.md
subscribeName: TensorFlow
---

<script type="text/x-mathjax-config">
MathJax.Hub.Config({
  TeX: { equationNumbers: { autoNumber: "AMS" } }
});
</script>

# Feature Scaling

In chapters [2.1](/books/tensorflow/book/ch2-linreg/2017-12-03-single-variable.html), [2.2](/books/tensorflow/book/ch2-linreg/2017-12-27-optimization.html), [2.3](/books/tensorflow/book/ch2-linreg/2018-03-21-multi-variable.html) we used the gradient descent algorithm (or variants of) to minimize a loss function, and thus achieve a line of best fit. However, it turns out that the optimization in chapter 2.3 was much, much slower than it needed to be. While this isn’t a big problem for these fairly simple linear regression models that we can train in seconds anyways, this inefficiency becomes a much more drastic problem when dealing with large data sets and models.

## Example of the Problem

First, let’s look at a concrete example of the problem, by again considering a synthetic data set. Like in chapter 2.3 I generated a simple [synthetic data set][synthetic-data] consisting of 2 independent variables \\(x_1\\) and \\(x_2\\), and one dependent variable \\(y = 2x_1 + 0.013x_2\\). However, note that the range for \\(x_1\\) is 0 to 10, but the range for \\(x_2\\) is 0 to 1000. Let's call this data set \\(D\\). A few sample data points look like this:

| \\(x_1\\) | \\(x_2\\) | \\(y\\)
|-----|---------------------|-|
|7.36|    1000|        34.25|
|9.47|    0|            19.24|
|0.52|    315.78|        3.50|
|1.57|    315.78|        11.02|
|6.31|    263.15|        13.93|
|1.57|    526.31|        10.21|
|0.52|    105.26|        3.41|
|4.21|    842.10|        16.27|
|2.63|    894.73|        19.04|
|3.68|    210.52|        4.60|
| ... | ... | ... |

For simplification I have excluded a constant term from this synthetic data set, and when training models I "cheated" by not training the constant term \\(b\\) and just setting it to 0, so as to simplify visualization. The model that I used was simply:

```python
y_predicted = tf.matmul(A, x) # A is a 1x2 matrix
```

Since this is just a simple application of the ideas from chapter 2.3, one would expect this to work fairly easily. However, when using a `GradientDescentOptimizer`, training goes rather poorly. Here is a plot showing the training progress of `A[0]` and the loss function side-by-side:

![Training progress without scaling][plot1]

This optimization was performed with a learning rate of 0.0000025, which is about the largest before it would diverge. The loss function quickly decreases at first, but then quickly stalls, and decreases quite slowly. Meanwhile, \\(A_0\\) is increasing towards the expected value of approximately 2.0, but very slowly. Within 5000 iterations this model fails to finish training.

Now let's do something rather strange: take the data set \\(D\\), and create a new data set \\(D'\\) by dividing all the \\(x_2\\) values by 100. The resulting data set \\(D'\\) looks roughly like this:

| \\(x_1'\\) | \\(x_2'\\) | \\(y'\\)
|-----|---------------------|-|
|7.36|    10|        34.25|
|9.47|    0|            19.24|
|0.52|    3.1578|        3.50|
|1.57|    3.1578|        11.02|
|6.31|    2.6315|        13.93|
|1.57|    5.2631|        10.21|
|0.52|    1.0526|        3.41|
|4.21|    8.4210|        16.27|
|2.63|    8.9473|        19.04|
|3.68|    2.1052|        4.60|
| ... | ... | ... |

A crucial note is that while \\(D'\\) is technically different from \\(D\\), it contains exactly the same information: one can convert between them freely, by dividing or multiplying the second column by 100. In fact, since this transformation is linear, and we are using a linear model, we can train our model on \\(D'\\) instead. We would just expect to obtain a value of 1.3 for `A[1]` rather than 0.013. So let's give it a try!

The first interesting observation is that we can use a much larger learning rate. The largest learning rate we could use with \\(D\\) was 0.0000025, but with \\(D'\\) we can use a learning rate of 0.01. And when we plot `A[0]` and the loss function for both \\(D\\) and \\(D'\\) we see something pretty crazy:

![Training progress with scaling][plot2]

While training on \\(D\\) wasn't even close to done after 5000 iterations, training on \\(D'\\) seems to have completed nearly instantly. If we zoom in on the first 60 iterations, we can see the training more clearly:

![Training progress with scaling zoom][plot3]

So this incredibly simple data set transformation has changed a problem that was untrainable within 5000 iterations to one that can be trained practically instantly with 50 iterations. What is this black magic, and how does it work?

## Visualizing Gradient Descent with Level Sets

One of the best ways to gain insight in machine learning is by visualization. As seen in chapter 2.1 we can visualize loss functions using plots. Since we have 2 parameters `A[0]` and `A[1]` of the loss function, it would be a 3D plot. We used this for visualization in chapter 2.1, but frankly it's a bit messy looking. Instead, we will use [level sets][level_sets] (also called contour plots), which use lines to indicate where the loss function has a constant value. An example is easiest, so here is the contour plot for the loss function for \\(D'\\), the one that converges quickly:

![Contour plot of D'][contour2]

Each ellipse border is where the loss function has a particular constant value (the innermost ones are labeled), and the red X marks the spot of the optimal values of `A[0]` and `A[1]`. By convention the particular constant values of the loss function are evenly spaced, which means that contour lines that are closer together indicate a "steeper" slope. In this plot, the center near the X is quite shallow, while far away is pretty steep. In addition, one diagonal axis of the ellipses is steeper than the other diagonal axis.

We can also plot how `A[0]` and `A[1]` evolve during training on the contour plot, to get a feel for how gradient descent is working:

![Contour plot of D' with training][contour2_dots]

Here, we can see that the initial values for `A[0]` and `A[1]` start pretty far off, but gradient descent quickly zeros in towards the X. As a side note, the line connecting each pair of dots is perpendicular to the line of the level set: see if you can figure out why.

So if that is what the contour plot looks like for the "good" case of \\(D'\\), how does it look for \\(D\\)? Well, it looks rather strange:

![Contour plot of D][contour1]

Don't be fooled: like the previous plot the level sets also form ellipses, but they are so stretched that they are nearly straight lines at this scale. This means that vertically (`A[0]` is constant and we vary `A[1]`) there is substantial gradient, but horizontally (`A[1]` is constant and we vary `A[0]`) there is practically no slope. Put another way, gradient descent only knows to vary `A[1]`, and (almost) doesn't vary `A[0]`.  We can test this hypothesis by plotting how gradient descent updates `A[0]` and `A[1]`, like above. Since gradient descent makes such little progress though, we have to zoom in a lot to see what is going on:

![Contour plot of D][contour1_dots]

We can clearly see that gradient descent applies large updates to `A[1]` (a bit too large, a smaller learning rate would have been a bit better) due to the large gradient in the `A[1]` direction. But due to the (comparatively) tiny gradient in the `A[0]` direction very small updates are done to `A[0]`. Gradient descent quickly converges on the optimal value of `A[1]`, but is very very far away from finding the optimal value of `A[0]`.

Let's take a quick look at what is going on mathematically to see why this happens. The model we are using is:
\\[
    y'(x, A) = Ax = a_1 x_1 + a_2 x_2
\\]
Here, \\(a_1\\) is in the role of `A[0]` and \\(a_2\\) is `A[1]`. We substitute this into the loss function to get:
\\[
    L(a_1, a_2) = \\sum_{i=1}^m (a_1 x_1^{(i)} + a_2 x_2^{(i)} - y^{(i)})^2
\\]
Now if we differentiate \\(L\\) in the direction of \\(a_1\\) and \\(a_2\\) separately, we get:
\\[
    \\frac{\\partial L}{\\partial a_1} = \\sum_{i=1}^m 2(a_1 x_1^{(i)} + a_2 x_2^{(i)} - y^{(i)})x_1^{(i)} \\\\
    \\frac{\\partial L}{\\partial a_2} = \\sum_{i=1}^m 2(a_1 x_1^{(i)} + a_2 x_2^{(i)} - y^{(i)})x_2^{(i)}
\\]
Here we can see the problem: the inner terms of the derivatives are the same, except one is multiplied by \\(x_1^{(i)}\\) and the other by \\(x_2^{(i)}\\). If \\(x_2\\) is on average 100 times bigger than \\(x_1\\) (which it is in the original data set), then we would expect \\(\\frac{\\partial L}{\\partial a_2}\\) to be roughly 100 times bigger than \\(\\frac{\\partial L}{\\partial a_1}\\). It isn't exactly 100 times larger, but with any reasonable data set it should be close. Since the derivatives in the directions of \\(a_1\\) and \\(a_2\\) are scaled completely differently, gradient descent fails to update both of them adequately.

The solution is simple: we need to rescale the input features before training. This is exactly what happened when we mysteriously divided by 100: we rescaled \\(x_2\\) to be comparable to \\(x_1\\). But we should work out a more methodical way of rescaling, rather than randomly dividing by 100.

## Rescaling Features

This wasn't explored above, but there are really two ways that we potentially need to rescale features. Consider an example where \\(x_1\\) ranges between -1 and 1, but \\(x_2\\) ranges between 99 and 101: both of these features have (at least approximately) the same [standard deviation][std_wiki], but \\(x_2\\) has a much larger [mean][mean_wiki]. On the other hand, consider an example where \\(x_1\\) still ranges between -1 and 1, but \\(x_2\\) ranges between -100 and 100. This time, they have the same mean, but \\(x_2\\) has a much larger standard deviation. Both of these situations can make gradient descent and related algorithms slower and less reliable. So, our goal is to ensure that all features have the same mean and standard deviation.

> **Note:** There are other methods to measure how different features are, and then subsequently rescale them. For a look at more of them, feel free to consult the [Wikipedia article][scaling_wiki]. However, after implementing the technique presented here, any other technique is fairly similar and easy to implement if needed.

Without digressing too far into statistics, let's quickly review how to calculate the mean \\(\\mu\\) and standard deviation \\(\\sigma\\). Suppose we have already read in our data set in Python into a Numpy vector / matrix, and have all the values for the feature \\(x_j\\), for each \\(j\\). Mathematically, the mean for feature \\(x_j\\) is just the average:
\\[
    \\mu_j = \\frac{\\sum_{i=1}^m x_j^{(i)}}{m}
\\]
In Numpy there is a convenient `np.mean()` function we will use.

The standard deviation \\(\\sigma\\) is a bit more tricky. We measure how far each point \\(x_j^{(i)}\\) is from the mean \\(\\mu_j\\), square this, then take the mean of all of this, and finally square root it:
\\[
    \\sigma_j = \\sqrt{\\frac{\\sum_{i=1}^m (x_j^{(i)} - \\mu_j)^2 }{m}}
\\]
Again, Numpy provides a convenient `np.std()` function.

> **Note:** Statistics nerds might point out that in the above equation we should divide by \\(m - 1\\) instead of \\(m\\) to obtain an unbiased estimate of the standard deviation. This might well be true, but in this usage it does not matter since we are only using this to rescale features relative to each other, and not make a rigorous statistical inference. To be more precise, doing so would involve multiplying each scaled feature by only a constant factor, and will not change any of their standard deviations or means relative to each other.

Once we have every mean \\(\\mu_j\\) and standard deviation \\(\\sigma_j\\), rescaling is easy: we simply rescale every feature like so:

\\[
\\begin{equation} \\label{eq:scaling}
    x_j' = \\frac{x_j - \\mu_j}{\\sigma_j}
\\end{equation}
\\]

This will force every feature to have a mean of 0 and a standard deviation of 1, and thus be scaled well relative to each other.

> **Note:** Be careful to make sure to perform the rescaling at both training time and prediction time. That is, we first have to perform the rescaling on the whole training data set, and then train the model so as to achieve good training performance. Once the model is trained and we have a new, never before seen input \\(x\\), we also need to rescale its features to \\(x'\\) because the trained model only understands inputs that have already been rescaled (because we trained it that way).

## Implementation and Experiments

Feature scaling can be applied to just about any multi-variable model. Since the only multi-variable model we have seen so far is multi-variable linear regression, we'll look at implementing feature scaling in that context, but the ideas and the code are pretty much the same for other models. First let's just copy-paste the original multi-variable linear regression code, and modify it to load the [new synthetic data set][synthetic-data]:

```python
import numpy as np
import tensorflow as tf
import pandas as pd
import matplotlib.pyplot as plt

# First we load the entire CSV file into an m x n matrix
D = np.matrix(pd.read_csv("linreg-scaling-synthetic.csv", header=None).values)

# Make a convenient variable to remember the number of input columns
n = 2

# We extract all rows and the first n columns into X_data
# Then we flip it
X_data = D[:, 0:n].transpose()

# We extract all rows and the last column into y_data
# Then we flip it
y_data = D[:, n].transpose()



# Define data placeholders
x = tf.placeholder(tf.float32, shape=(n, None))
y = tf.placeholder(tf.float32, shape=(1, None))

# Define trainable variables
A = tf.get_variable("A", shape=(1, n))
b = tf.get_variable("b", shape=())

# Define model output
y_predicted = tf.matmul(A, x) + b

# Define the loss function
L = tf.reduce_sum((y_predicted - y)**2)

# Define optimizer object
optimizer = tf.train.AdamOptimizer(learning_rate=0.1).minimize(L)

# Create a session and initialize variables
session = tf.Session()
session.run(tf.global_variables_initializer())

# Main optimization loop
for t in range(2000):
    _, current_loss, current_A, current_b = session.run([optimizer, L, A, b], feed_dict={
        x: X_data,
        y: y_data
    })
    print("t = %g, loss = %g, A = %s, b = %g" % (t, current_loss, str(current_A), current_b))

```

If you run this code right now, you might see output like the following:

```
...
t = 1995, loss = 0.505296, A = [[1.9920335  0.01292674]], b = 0.089939
t = 1996, loss = 0.5042, A = [[1.9920422  0.01292683]], b = 0.0898404
t = 1997, loss = 0.5031, A = [[1.9920509  0.01292691]], b = 0.0897419
t = 1998, loss = 0.501984, A = [[1.9920596  0.01292699]], b = 0.0896435
t = 1999, loss = 0.50089, A = [[1.9920683  0.01292707]], b = 0.0895451
```

Note that if you run this code multiple times you will get different results each time due to the random initialization. The synthetic data was generated with the equation \\(y = 2x_1 + 0.013x_2 + 0\\). So after 2000 iterations of training we are getting close-ish to the correct values, but it's not fully trained. So let's implement feature scaling to fix this.

The first step is to compute the mean and standard deviation for each feature in the training data set. Add the following after `X_data` is loaded:

```python
means = X_data.mean(axis=1)
deviations = X_data.std(axis=1)
```

Using `axis=1` means that we compute the mean for each feature, averaged over all data points. That is, `X_data` is a \\(2 \\times 400\\) matrix, `means` is a \\(2 \\times 1\\) matrix (vector). If we had used `axis=0`, `means` would be a \\(1 \\times 400\\) matrix, which is not what we want.

Now that we know the means and standard deviations of each feature, we have to use them to transform the inputs via Equation \\((\\ref{eq:scaling})\\). We have two options: we could implement the transformation directly using `numpy` to transform `X_data` before training, or we could include the transformation within the TensorFlow computations. But since we need to do the transformation again when we want to predict output given new inputs, including it in the TensorFlow computations will be a bit more convenient.

Just like before we setup `x` as a placeholder, so this code is identical:

```python
# Define data placeholders
x = tf.placeholder(tf.float32, shape=(n, None))
```

Now we want to transform `x` according to Equation \\((\\ref{eq:scaling})\\). We can define a new TensorFlow value `x_scaled`:

```python
# Apply the rescaling
x_scaled = (x - means) / deviations
```

One important concept is that when this line of code runs, `means` and `deviations` have already been computed: they are actual matrices. So to TensorFlow they are **constants**: they are neither trainable variables that will be updated during optimization, nor are they placeholders that need to be fed values later. They have already been computed, and TensorFlow just uses the already computed values directly.

Also, note that since `x` and `means` are compatibly sized matrices, the subtraction (and division) will be done separately for each feature, automatically. So while Equation \\((\\ref{eq:scaling})\\) technically says how to scale one feature individually, this single line of code scales all the features.

Now, everywhere that we use `x` in the model we should now use `x_scaled` instead. For us the only code that needs to change is in defining the model's prediction:

```python
# Define model output, using the scaled features
y_predicted = tf.matmul(A, x_scaled) + b
```

Note that when we run the session we do *not* feed values to `x_scaled`, because we want to feed unscaled values to `x` which will then automatically get scaled when `x_scaled` is computed based on what is fed to `x`. We instead continue to feed values to `x` as before.

That's all that we need to implement for feature scaling, it's really pretty easy. If we run this code now we should see:

```
...
t = 1995, loss = 1.17733e-07, A = [[6.0697703 3.9453504]], b = 16.5
t = 1996, loss = 1.17733e-07, A = [[6.0697703 3.9453504]], b = 16.5
t = 1997, loss = 1.17733e-07, A = [[6.0697703 3.9453504]], b = 16.5
t = 1998, loss = 1.17733e-07, A = [[6.0697703 3.9453504]], b = 16.5
t = 1999, loss = 1.17733e-07, A = [[6.0697703 3.9453504]], b = 16.5
```

The fact that none of the trained weights are updating anymore and that the loss function is very small is a good sign that we have achieved convergence. But the weights are not the values we expect... shouldn't we get `A = [[2, 0.013]], b = 0`, since that is the equation that generated the synthetic data? Actually, the weights we got *are* correct because the model is now being trained on the rescaled data set, so we get different weights at the end. See Challenge Problem 3 below to explore this more.

# Concluding Remarks

We've seen how to implement feature scaling for a simple multi-variable linear regression model.  While this is a fairly simple model, the principles are basically the same for applying feature scaling to more complex models (which we will see very soon). In fact, the code is pretty much identical: just compute the means and standard deviations, apply the formula to `x` to compute `x_scaled`, and anywhere you use `x` in the model, just use `x_scaled` instead.

Whether or not feature scaling helps is dependent on the problem and the model. If you have having trouble getting your model to converge, you can always implement feature scaling and see how it affects the training. While it wasn't used in this chapter, using TensorBoard (see [chapter 2.2](/books/tensorflow/book/ch2-linreg/2017-12-27-optimization.html)) is a great way to run experiments to compare training with and without feature scaling.

# Challenge Problems

<!-- <ol>
    <li>**Coding:** Take one of the challenge problems from [the previous chapter](/books/tensorflow/book/ch2-linreg/2018-03-21-multi-variable.html#challenge-problems), and implement it with and without feature scaling, and then compare how training performs.</li>
</ol> -->

1. **Coding:** Take one of the challenge problems from [the previous chapter](/books/tensorflow/book/ch2-linreg/2018-03-21-multi-variable.html#challenge-problems), and implement it with and without feature scaling, and then compare how training performs.
2. **Coding:** Add TensorBoard support to the code from this chapter, and use it to explore for yourself the effect feature scaling has. Also, you can compare with different optimization algorithms.
3. **Theory:** One problem with feature scaling is that we learn different model parameters. In this chapter, the original data set was generated with \\(y = 2x_1 + 0.013x_2 + 0\\), but the parameters that the model learned were \\(a_1' = 6.0697703, a_2' = 3.9453504, b' = 16.5\\). Note that I have called these learned parameters \\(a_1'\\) rather than \\(a_1\\) to indicate that these learned parameters are for the *rescaled* data.<br/><br />One important use of linear regression is to explore and understand a data set, not just predict outputs. After doing a regression, one can understand through the learned weights how severely each feature affects the output. For example, if we have two features \\(A\\) and \\(B\\) which are used to predict rates of cancer, and the learned weight for \\(A\\) is much larger than the learned weight for \\(B\\), this indicates that occurrence of \\(A\\) is more correlated with cancer than occurrence of \\(B\\) is, which is interesting in its own right. Unfortunately, performing feature scaling destroys this: since we have rescaled the training data, the weight for \\(A\\) (\\(a_1'\\)) has lost its relevance to the values of \\(A\\) in the real world. But all is not lost, since we can actually recover the "not-rescaled" parameters from the learned parameters, which we will derive now.<br/><br/>First, suppose that we have learned the "rescaled" parameters \\(a_1', a_2', b'\\). That is, we predict the output \\(y\\) to be: \\[\\begin{equation}\\label{eq:start}y = a_1' x_1' + a_2' x_2' + b'\\end{equation}\\]where \\(x_1', x_2'\\) are the rescaled features. Now, we *want* a model that uses some (yet unknown) weights \\(a_1, a_2, b\\) to predict the same output \\(y\\), but using the not-rescaled features \\(x_1, x_2\\). That is, we want to find \\(a_1, a_2, b\\) such that this is true:\\[\\begin{equation}\\label{eq:goal}y = a_1 x_1 + a_2 x_2 + b\\end{equation}\\]To go about doing this, substitute for \\(x_1'\\) and \\(x_2'\\) using Equation \\((\\ref{eq:scaling})\\) into Equation \\((\\ref{eq:start})\\), and then compare it to Equation \\((\\ref{eq:goal})\\) to obtain 3 equations: one to relate each scaled and not-scaled parameter. Finally, you can use these equations to be able to compute the "not-scaled" parameters in terms of the learned parameters. You can check your work by computing the not-scaled parameters in terms of the learned parameters for the synthetic data set, and verify that they match the expected values.

# Complete Code

The [complete example code is available on GitHub](https://github.com/donald-pinckney/donald-pinckney.github.io/blob/src/books/tensorflow/src/ch2-linreg/code/feature_scaling.py), as well as directly here:

```python
import numpy as np
import tensorflow as tf
import pandas as pd
import matplotlib.pyplot as plt

# First we load the entire CSV file into an m x n matrix
D = np.matrix(pd.read_csv("linreg-scaling-synthetic.csv", header=None).values)

# Make a convenient variable to remember the number of input columns
n = 2

# We extract all rows and the first n columns into X_data
# Then we flip it
X_data = D[:, 0:n].transpose()

# We extract all rows and the last column into y_data
# Then we flip it
y_data = D[:, n].transpose()

# We compute the mean and standard deviation of each feature
means = X_data.mean(axis=1)
deviations = X_data.std(axis=1)

print(means)
print(deviations)

# Define data placeholders
x = tf.placeholder(tf.float32, shape=(n, None))
y = tf.placeholder(tf.float32, shape=(1, None))

# Apply the rescaling
x_scaled = (x - means) / deviations

# Define trainable variables
A = tf.get_variable("A", shape=(1, n))
b = tf.get_variable("b", shape=())

# Define model output, using the scaled features
y_predicted = tf.matmul(A, x_scaled) + b

# Define the loss function
L = tf.reduce_sum((y_predicted - y)**2)

# Define optimizer object
optimizer = tf.train.AdamOptimizer(learning_rate=0.1).minimize(L)

# Create a session and initialize variables
session = tf.Session()
session.run(tf.global_variables_initializer())

# Main optimization loop
for t in range(2000):
    _, current_loss, current_A, current_b = session.run([optimizer, L, A, b], feed_dict={
        x: X_data,
        y: y_data
    })
    print("t = %g, loss = %g, A = %s, b = %g" % (t, current_loss, str(current_A), current_b))

```

[synthetic-data]: /books/tensorflow/book/ch2-linreg/code/linreg-scaling-synthetic.csv
[plot1]: /books/tensorflow/book/ch2-linreg/assets/scaling_plot1.png
[plot2]: /books/tensorflow/book/ch2-linreg/assets/scaling_plot2.png
[plot3]: /books/tensorflow/book/ch2-linreg/assets/scaling_plot3.png
[contour1]: /books/tensorflow/book/ch2-linreg/assets/contour1.png
[contour2]: /books/tensorflow/book/ch2-linreg/assets/contour2.png
[contour1_dots]: /books/tensorflow/book/ch2-linreg/assets/contour1_dots.png
[contour2_dots]: /books/tensorflow/book/ch2-linreg/assets/contour2_dots.png
[level_sets]: https://en.wikipedia.org/wiki/Level_set
[mean_wiki]: https://en.wikipedia.org/wiki/Expected_value
[std_wiki]: https://en.wikipedia.org/wiki/Standard_deviation
[scaling_wiki]: https://en.wikipedia.org/wiki/Feature_scaling
