# Parallelize Cypress

{% embed url="https://github.com/testdouble/cypress-rails/issues/55" %}



Currently we are hitting up close to 20m build times for e2e tests in travis.

We _want_ to use knapsack pro to split this up - but the main problem is the support for this uses a direct call to the npm module cypress - while the rake task is needed to setup the db / server \(and not automatically called by the knapsack integration\).

It would be nice to solve this problem.

