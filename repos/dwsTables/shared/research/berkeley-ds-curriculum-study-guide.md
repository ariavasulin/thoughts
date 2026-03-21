---
title: "Berkeley Data Science Curriculum - Complete Knowledge Toolkit"
date: 2026-03-21
tags: [reference, data-science, berkeley, curriculum, machine-learning, statistics, probability]
---

# Berkeley Data Science Curriculum: Complete Knowledge Toolkit

A comprehensive, technically detailed reference of all topics and concepts covered across UC Berkeley's core data science curriculum. Organized as a knowledge toolkit for an expert data scientist.

---

## Data 8: Foundations of Data Science

**Textbook**: *Computational and Inferential Thinking* (Adhikari, DeNero, Wagner) — [inferentialthinking.com](https://inferentialthinking.com/)

### Core Programming & Data Manipulation
- Python 3, Jupyter notebooks, custom `datascience` library
- Table operations: `select`, `where`, `sort`, `group`, `pivot`, `join`, `apply`, `sample`
- NumPy arrays: element-wise operations, vectorization, broadcasting
- Aggregation: `np.mean`, `np.sum`, `np.std`, `np.percentile`

### Visualization
- Bar charts (`barh`), histograms (`hist`), scatter plots (`scatter`), line plots (`plot`)
- Distribution analysis, overlaid graphs

### Causality & Experimental Design
- Observational studies vs. randomized controlled trials (RCTs)
- Confounding variables, establishing causality
- John Snow cholera case study

### Probability & Sampling
- Empirical probability via simulation, Law of Large Numbers
- Conditional probability, independence
- Monty Hall problem
- Random sampling: `tbl.sample(n, with_replacement=False)`
- Empirical vs. sampling distributions

### Statistical Inference
- **Hypothesis testing**: null/alternative hypotheses, p-values, significance levels
- Test statistics: Total Variation Distance (TVD), difference in means
- Permutation tests, simulation-based null distributions
- Type I/II errors
- **Bootstrap**: resampling with replacement, bootstrap confidence intervals (percentile method)
- Standard error estimation
- Central Limit Theorem, Chebyshev's Inequality

### Prediction & Classification
- Correlation coefficient (r), R²
- Simple linear regression: least squares, slope/intercept, residuals
- Inference for regression: bootstrap slope, prediction intervals
- k-Nearest Neighbors (k-NN): Euclidean distance, train/test split, accuracy
- Bayes' Rule: prior, likelihood, posterior

---

## Data 100: Principles and Techniques of Data Science

**Website**: [ds100.org](https://ds100.org/)

### Data Manipulation with Pandas
- DataFrame: `groupby`, `agg`, `pivot_table`, `merge`, `sort_values`, `drop_duplicates`
- String operations, multi-index grouping, lambda aggregations

### Data Cleaning & EDA
- Missing values, duplicates, outliers, type conversions
- Summary statistics, correlation analysis, pattern detection

### Regular Expressions (Regex)
- Pattern matching, text extraction/substitution
- `re.findall()`, regex syntax and debugging

### Visualization
- Matplotlib, Seaborn, Plotly
- Histograms, scatter plots, box plots, pair plots

### Sampling & Probability
- Population inference, sampling strategies, sampling bias
- Random variables, probability distributions, expected value, variance

### Linear Regression & Loss Functions
- **SLR**: Y = β₀ + β₁X + ε
- **Loss functions**: L2 (MSE), L1 (MAE), cross-entropy
- **OLS**: matrix form Ŷ = Xθ, normal equations θ̂ = (XᵀX)⁻¹XᵀY
- Geometric interpretation: projection onto column space

### Optimization
- **Gradient descent**: batch, stochastic (SGD), mini-batch
- Update rule: θ := θ - α∇L(θ), learning rate selection, convergence

### Feature Engineering
- One-hot encoding, polynomial features, interaction terms
- `sklearn.pipeline.Pipeline`, `ColumnTransformer`
- Preventing data leakage in cross-validation

### Regularization
- **Ridge (L2)**: λ‖θ‖₂² — shrinks coefficients, closed-form solution
- **Lasso (L1)**: λ‖θ‖₁ — forces sparsity (feature selection), no closed-form
- **Elastic Net**: L1 + L2 combination
- Cross-validation for λ selection

### Bias-Variance Tradeoff
- Bias (underfitting) vs. variance (overfitting)
- Model complexity relationship, generalization

### Cross-Validation
- K-fold, leave-one-out (LOOCV), stratified, time-series CV
- Model evaluation, hyperparameter tuning

### Logistic Regression
- Sigmoid: σ(z) = 1/(1 + e⁻ᶻ)
- Binary cross-entropy loss, gradient descent optimization
- Coefficients as log-odds ratios

### PCA (Principal Component Analysis)
- Eigendecomposition of covariance matrix
- Principal components as orthogonal eigenvectors
- Variance explained, scree plots, dimensionality selection

### Clustering
- **k-Means**: Lloyd's algorithm, centroid initialization, convergence
- **Hierarchical**: agglomerative, linkage criteria (Ward, complete, average, single), dendrograms
- **Evaluation**: silhouette score, elbow method

### SQL & Databases
- SELECT, FROM, WHERE, ORDER BY, LIMIT
- JOINs (INNER, LEFT, RIGHT, FULL OUTER)
- GROUP BY, HAVING, aggregate functions
- Subqueries

### Ethics & Human Contexts
- Fairness, bias in models, privacy, societal impact

---

## Data 140 (Prob 140): Probability for Data Science

**Textbook**: *Probability for Data Science* (Adhikari, Pitman) — [textbook.prob140.org](https://textbook.prob140.org/)

### Fundamentals
- Outcome spaces, events, equally likely outcomes
- Birthday problem, hashing collisions, exponential approximation

### Calculating Chances
- Addition rule, multiplication principle
- Conditional probability, updating probabilities
- Inclusion-exclusion principle, derangements (matching problem)

### Random Variables & Distributions
- Functions on outcome spaces, PMFs
- Joint, marginal, conditional distributions
- Dependence and independence

### Discrete Distributions
- **Binomial**(n, p), **Hypergeometric**(N, G, n), **Poisson**(λ), **Geometric**(p), **Multinomial**
- Law of Small Numbers (Poisson approximation to binomial)
- Poissonizing binomial and multinomial

### Expectation & Variance
- E[X] = Σ x·P(X=x), additivity/linearity
- E[g(X)], expectations of functions
- Variance, SD, covariance properties
- Iterated expectation: E[Y] = E[E[Y|X]]

### Tail Bounds
- **Markov**: P(Y ≥ y) ≤ E[Y]/y
- **Chebyshev**: P(|X−μ| ≥ c) ≤ σ²/c²
- **Chernoff**: exponential tail bounds via MGF
- Heavy tails

### Markov Chains
- Transition matrices, one-step probabilities
- Steady-state/stationary distributions
- **Detailed balance**: π(i)P(i,j) = π(j)P(j,i)
- Reversibility, ergodicity
- **MCMC**: Metropolis-Hastings, Gibbs sampling
- Code breaking applications

### Continuous Distributions
- PDFs, CDFs, expectation via integration
- **Uniform**(a,b), **Exponential**(λ) — memoryless property
- **Normal/Gaussian**(μ,σ²), **Gamma**(α,λ), **Beta**(α,β)
- **Chi-squared**(n), **Rayleigh**, **Bivariate/Multivariate Normal**
- Symbolic calculus with SymPy

### Transformations of Random Variables
- Linear transformations, monotone functions
- Two-to-one functions, change of variables (Jacobian)

### Sums and Convolutions
- **Convolution formula**: f_S(s) = ∫ f_X(x)·f_Y(s−x)dx
- **Moment Generating Functions**: M_X(t) = E[e^{tX}]
- MGF uniqueness, multiplication for independent sums
- Sums of normal variables

### Convergence Theorems
- **Weak Law of Large Numbers** (WLLN)
- **Central Limit Theorem** (CLT)
- Probability Generating Functions (PGFs)
- Confidence intervals, finite population correction

### Estimation
- **MLE**: argmax of likelihood function
- **Bayesian inference**: prior → likelihood → posterior
- **Beta-Binomial conjugacy**: Beta(α,β) prior + Bin(n,p) likelihood → Beta(X+α, n−X+β) posterior

### Prediction & Regression
- **Conditional expectation as projection** (minimizes MSE)
- **Law of total variance**: Var(Y) = E[Var(Y|X)] + Var(E[Y|X])
- Least squares predictor: E[Y|X]
- **Bivariate normal**: regression line slope a* = ρ·σ_Y/σ_X
- **Multiple regression** in matrix notation

### Multivariate Normal
- Random vectors, mean vector, covariance matrix
- Linear combinations, independence (correlation=0 ⟹ independence for normals)

---

## CS 189 / EECS 189: Introduction to Machine Learning

**Website**: [eecs189.org](https://eecs189.org/)

### Classification Algorithms

**Linear Classifiers**:
- Perceptron: decision boundaries, online learning
- **SVMs**: hard/soft-margin, dual formulation, Lagrangian, hinge loss, kernel SVMs
- **Logistic regression**: MLE, cross-entropy, multinomial extension

**Probabilistic Methods**:
- **Gaussian Discriminant Analysis**: LDA, QDA
- **Naive Bayes**: conditional independence assumption, generative vs. discriminative

**Non-linear Methods**:
- **Decision trees**: ID3 (entropy/information gain), CART (Gini impurity), pruning
- **k-NN**: distance metrics, k-d trees, Voronoi diagrams, curse of dimensionality

### Ensemble Methods
- **Bagging**: bootstrap aggregating, variance reduction
- **Random Forests**: feature bagging, OOB error, variable importance
- **Boosting**: AdaBoost, weight resampling, exponential loss

### Regression
- Linear regression (OLS, normal equations), ridge (L2), lasso (L1)
- Polynomial regression, kernel ridge regression

### Kernel Methods
- **Kernel trick**: implicit feature mapping
- Kernels: polynomial, RBF/Gaussian, linear, sigmoid
- Kernel perceptron, kernel logistic regression, kernel PCA

### Dimensionality Reduction
- **PCA**: covariance eigendecomposition, maximum variance projection
- **SVD**: X = UDVᵀ, relationship to PCA, truncated SVD, low-rank approximation
- Random projection, Johnson-Lindenstrauss lemma

### Clustering
- k-Means (Lloyd's, k-means++), EM algorithm, GMMs
- Hierarchical clustering, dendrograms

### Neural Networks & Deep Learning

**Fundamentals**:
- Activations: sigmoid, ReLU, tanh, leaky ReLU
- Loss functions: MSE, cross-entropy, hinge
- Backpropagation, computational graphs, chain rule
- Regularization: L1/L2, dropout, batch normalization, early stopping
- Initialization: Xavier/Glorot, He
- Optimizers: SGD, momentum, Adam, AdamW, learning rate schedules

**Architectures**:
- **CNNs**: convolution, stride, padding, pooling, ResNets
- **RNNs**: hidden states, vanishing gradients, LSTM, GRU
- **Transformers**: self-attention, multi-head attention, positional encoding
- **Generative**: autoencoders, VAEs, GANs, LLMs

### Mathematical Foundations
- **Linear algebra**: eigenvalues/vectors, SVD, matrix norms, PSD matrices, projections
- **Optimization**: gradients, Jacobians, Hessians, convex optimization, Lagrangian duality, KKT conditions
- **Probability**: Bayes' rule, MLE, MAP, Bayesian decision theory

### Learning Theory
- Bias-variance decomposition
- VC dimension, shattering, growth functions
- Generalization bounds, PAC learning
- Overfitting/underfitting, model complexity

### Model Selection
- Train/validation/test splits
- K-fold cross-validation, LOOCV
- Hyperparameter tuning: grid search, random search

---

## Data 140 → Data 145: Evidence and Uncertainty

**Website**: [data145.org](https://data145.org/)
**Prerequisites**: Math 53, Data C100, Data C140 or EECS 126

### Convergence Theory
- **Convergence in distribution**: F_n(x) → F(x) at continuity points
- **Convergence in probability**: P(|X_n − X| > ε) → 0
- **Continuous Mapping Theorem**: g continuous, X_n →^d X ⟹ g(X_n) →^d g(X)
- **Slutsky's Theorem**: X_n →^d X, Y_n →^P c ⟹ X_n + Y_n →^d X + c
- **Delta Method**: √n(g(Y_n) − g(θ)) →^d N(0, [g'(θ)]²σ²)

### Maximum Likelihood Estimation (MLE)
- Likelihood, log-likelihood, score function S_n(θ)
- **Score has mean zero**: E_θ[ℓ'₁(θ; X)] = 0
- **Fisher Information**: I(θ) = Var_θ(ℓ'₁) = −E_θ[ℓ''₁]
- **Consistency**: via Jensen's inequality and WLLN
- **Asymptotic normality**: √n(θ̂ − θ₀) →^d N(0, 1/I(θ₀))
- **Cramér-Rao bound**: Var(T) ≥ 1/(nI(θ)) for unbiased T
- MLE is asymptotically efficient

### Decision Theory
- Loss functions (squared error), risk (MSE)
- **MSE = Bias² + Variance**
- Admissibility, inadmissibility
- **Minimax**: minimize max_θ R(θ; T)
- **Bayes risk**: minimize ∫R(θ;T)π(θ)dθ → posterior mean is Bayes estimator

### Bayesian Inference
- **Bayes' rule**: π(θ|x) ∝ f_θ(x)·π(θ)
- **Conjugate families**: Beta-Binomial, Gamma-Exponential, Normal-Normal
- Posterior mean as weighted average of MLE and prior mean
- **Bernstein-von Mises**: for large n, π(θ|X) ≈ N(θ̂_MLE, 1/(nI(θ̂)))
- **Jeffreys prior**: π(θ) ∝ √I(θ) — reparameterization invariant
- Hierarchical Bayes, empirical Bayes
- Gibbs sampler for hierarchical models
- Credible vs. confidence intervals

### Model Misspecification
- **KL divergence**: D_KL(g‖f) = E_g[log(g(X)/f(X))] ≥ 0 (Gibbs' inequality)
- MLE under misspecification → θ* = argmin_θ D_KL(g‖f_θ)
- **Sandwich estimator** for misspecified variance

### Nonparametric Methods
- Empirical CDF, WLLN for each x
- **Parametric bootstrap**: simulate from f_θ̂
- **Nonparametric bootstrap**: resample from empirical distribution
- Bootstrap CIs: normal, percentile, empirical/basic

### Goodness-of-Fit
- **Kolmogorov-Smirnov test**: D_n = sup_x |F_n(x) − F(x)|
- One-sample and two-sample KS tests

### Hypothesis Testing
- **Neyman-Pearson Lemma**: likelihood ratio test is most powerful
- **UMP tests**: monotone likelihood ratio property
- **Generalized LRT**: Λ = 2[ℓ(θ̂_full) − ℓ(θ̂_null)]
- **Wilks' Theorem**: Λ →^d χ²_r under H₀
- **Trinity of tests**: LRT, Wald, Score (Rao's) — all →^d χ²₁
- Chi-squared goodness-of-fit test

### Multiple Testing
- **FWER**: P(≥1 false positive)
- **Bonferroni**: test at α/m
- **Holm-Bonferroni**: step-down procedure, more powerful
- **FDR**: E[false positives / total rejections]
- **Benjamini-Hochberg**: p_(i) ≤ iα/m, controls FDR

### Advanced Topics
- **Sub-Gaussian variables**: E[exp(λ(X−μ))] ≤ exp(λ²σ²/2)
- **Hoeffding's inequality**: P(|X̄_n − μ| > t) ≤ 2exp(−2nt²/(b−a)²)
- **Exponential families**: f_θ(x) = h(x)exp{η(θ)·T(x) − A(θ)}
  - E[T(X)] = A'(θ), Var(T(X)) = A''(θ)
- Fisher Information Matrix (multivariate)
- Permutation testing
- Causal inference foundations
- Shannon entropy, mutual information

---

## Data 101: Data Engineering

**Website**: [data101.org](https://data101.org/)

### SQL (Advanced)
- Subqueries (correlated/uncorrelated), CTEs (recursive), window functions
- String manipulation: LIKE, SUBSTRING, REGEXP_REPLACE with POSIX regex
- Views (virtual vs. materialized), sampling techniques

### Relational Model & Algebra
- Primitive RA: selection (σ), projection (π), cross product (×), union, difference, rename
- Extended RA: grouping/aggregation (γ), outer joins
- SQL → relational algebra translation

### Data Definition & Constraints
- DDL (CREATE, ALTER, DROP), DML (INSERT, UPDATE, DELETE)
- PRIMARY KEY, UNIQUE, NOT NULL, DEFAULT, CHECK, FOREIGN KEY
- Referential integrity: CASCADE, SET NULL

### Query Performance & Optimization
- **B+ Trees**: logarithmic lookup, range queries, self-balancing
- **Hash indexes**: O(1) exact lookup only
- Index selection: cardinality, query patterns, maintenance costs
- Query planning: physical vs. logical plans, cost-based optimization
- I/O cost analysis: sequential vs. random reads

### Data Modeling
- Relations, tensors, dataframes
- Entity-Relationship diagrams, cardinality constraints
- Normalization, reducing redundancy
- PIVOT/UNPIVOT operations

### Data Preparation & Cleaning
- Granularity: roll up (coarser), drill down (finer)
- Outlier detection: z-score, MAD, winsorizing, trimming
- Imputation: mean/median, correlation-based, model-based, interpolation
- Entity resolution: blocking, matching, cleaning join keys

### Semi-Structured Data & NoSQL
- JSON, XML formats
- NoSQL: no joins, horizontal scalability, eventual consistency
- Partitioning vs. replication tradeoffs
- Key-value stores, document stores (MongoDB)
- MongoDB Query Language (MQL), aggregation pipelines

### Transactions & Concurrency
- ACID properties
- Strict Two-Phase Locking (2PL): shared/exclusive locks
- Isolation levels, deadlock detection
- COMMIT, ROLLBACK

### Parallel & Distributed Computing
- MapReduce paradigm, fault tolerance
- Distributed database systems, data partitioning

### Data Pipelines
- ETL vs. ELT, dbt
- ML pipelines: data validation, model training, prediction serving
- Event streaming, log analytics

### Business Intelligence & OLAP
- Data cubes, multidimensional analysis
- Data warehousing, BI fundamentals

### Graph Databases
- Property graph model, SQL implementation
- Path traversals via recursive CTEs

### Sampling
- Reservoir sampling (Algorithm R): O(n) single-pass, k/n probability

---

## Data 102: Data, Inference, and Decisions

**Website**: [data102.org](https://data102.org/)
**Textbook**: [data102.org/ds-102-book/](https://data102.org/ds-102-book/)

### Binary Decision-Making & Hypothesis Testing
- Type I/II errors, sensitivity, specificity, TPR, FPR, FNR, FDP
- ROC curves, AUC
- p-values, Neyman-Pearson lemma, likelihood ratio tests
- **Multiple testing**: FWER (Bonferroni), FDR (Benjamini-Hochberg)
- Permutation testing

### Decision Theory
- Loss functions, risk, minimax vs. Bayes risk
- Decision rules, admissibility

### Bayesian Inference & Modeling
- Prior, likelihood, posterior; MLE vs. MAP
- Conjugate priors
- **Hierarchical models**: hyperparameters, empirical Bayes, shrinkage
- **Graphical models**: Bayesian networks, d-separation, conditional independence
- **MCMC**: Monte Carlo integration, rejection/importance sampling, Gibbs, Metropolis-Hastings

### Prediction & GLMs
- **Generalized Linear Models**: exponential family, link functions (logit, probit, log)
- Logistic regression, Poisson regression
- Model checking: deviance, residual plots, AIC/BIC
- Uncertainty quantification: bootstrap, prediction intervals
- **Nonparametric**: kernel methods, k-NN, LOESS, splines
- **Neural networks**: feedforward, backpropagation, activation functions, interpretability
- **Ensemble methods**: decision trees, random forests, bagging, boosting

### Causal Inference
- Association vs. causation, Simpson's paradox, DAGs
- **Potential outcomes framework** (Rubin): ITE, ATE, CATE, SUTVA
- **RCTs**: randomization, A/B testing, power analysis
- **Observational studies**: unconfoundedness, propensity scores, matching
- **IPW** (inverse probability weighting), doubly robust estimation
- **Instrumental variables**: 2SLS, LATE, weak instruments
- Regression discontinuity, difference-in-differences, synthetic control

### Concentration Inequalities & Learning Theory
- Markov, Chebyshev, Hoeffding, Chernoff, Bennett, Bernstein
- Sub-Gaussian/sub-exponential variables
- **ERM**, PAC learning, VC dimension, Rademacher complexity
- Generalization bounds, uniform convergence
- Bias-variance tradeoff, regularization (ridge, lasso, elastic net)

### Multi-Armed Bandits
- Exploration-exploitation tradeoff, cumulative regret
- ε-greedy, **UCB** (Upper Confidence Bound), **Thompson Sampling**
- Contextual bandits

### Reinforcement Learning
- **MDPs**: states, actions, rewards, transitions, discount factors, policies
- Value functions, Q-functions, Bellman equations
- **Dynamic programming**: value iteration, policy iteration
- **RL algorithms**: TD learning, Q-learning, SARSA
- Monte Carlo Tree Search (MCTS)

### Privacy & Fairness
- **Differential privacy**: ε-DP, (ε,δ)-DP, sensitivity
- Mechanisms: Laplace, Gaussian, exponential
- Composition theorems, privacy budget
- **Fairness**: demographic parity, equalized odds, calibration, individual fairness

---

## Cross-Course Concept Map

| Concept | Data 8 | Data 100 | Data 140 | CS 189 | Data 145 | Data 101 | Data 102 |
|---|---|---|---|---|---|---|---|
| Hypothesis Testing | ✓ | ✓ | | | ✓✓ | | ✓✓ |
| Bootstrap | ✓ | ✓ | | | ✓✓ | | ✓ |
| Linear Regression | ✓ | ✓✓ | ✓ | ✓✓ | ✓ | | ✓ |
| Bayesian Inference | ✓ | | ✓✓ | ✓ | ✓✓ | | ✓✓ |
| MLE | | | ✓ | ✓ | ✓✓ | | ✓ |
| PCA/SVD | | ✓ | | ✓✓ | | | |
| Neural Networks | | | | ✓✓ | | | ✓ |
| Causal Inference | ✓ | | | | ✓ | | ✓✓ |
| SQL | | ✓ | | | | ✓✓ | |
| Markov Chains | | | ✓✓ | | | | ✓ |
| Decision Theory | | | | ✓ | ✓✓ | | ✓✓ |
| Multiple Testing | | | | | ✓✓ | | ✓✓ |
| Clustering | | ✓ | | ✓✓ | | | ✓ |
| Regularization | | ✓✓ | | ✓✓ | | | ✓ |
| RL/Bandits | | | | | | | ✓✓ |
| Privacy/Fairness | | ✓ | | | | ✓ | ✓✓ |

✓ = covered, ✓✓ = deep coverage
