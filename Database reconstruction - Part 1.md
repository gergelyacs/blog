
One common challenge is convincing people that aggregate information can still qualify as personal data under the GDPR. By “aggregate information,” we refer to statistical summaries such as sums, medians, and means derived from a confidential dataset, or even the parameters of a trained machine learning model.

In this post, we’ll start with the simplest case: demonstrating how sensitive personal data can be reconstructed from the results of a series of SUM queries performed on a confidential dataset. In a follow-up post, we’ll explore how an attacker can strategically minimize the number of queries needed to reconstruct every record in the dataset.

For the sake of illustration, consider the following hospital dataset.

| <span style="color:green;">**#**</span> | <span style="color:red;">**Name**</span>       | <span style="color:green;">**ZIP**</span> | <span style="color:green;">**Gender**</span> | <span style="color:red;">**Blood sugar**</span> |
| --------------------------------------- | ---------------------------------------------- | ----------------------------------------- | -------------------------------------------- | ----------------------------------------------- |
| <span style="color:green;">1</span>     | <span style="color:red;">John Smith</span>     | <span style="color:green;">32453</span>   | <span style="color:green;">Male</span>       | <span style="color:red;">4.3</span>             |
| <span style="color:green;">2</span>     | <span style="color:red;">Jeremy Doe</span>     | <span style="color:green;">43813</span>   | <span style="color:green;">Male</span>       | <span style="color:red;">5.2</span>             |
| <span style="color:green;">3</span>     | <span style="color:red;">Susan Atkinson</span> | <span style="color:green;">43765</span>   | <span style="color:green;">Female</span>     | <span style="color:red;">6.1</span>             |
| <span style="color:green;">4</span>     | <span style="color:red;">Eve Brown</span>      | <span style="color:green;">32187</span>   | <span style="color:green;">Female</span>     | <span style="color:red;">3.2</span>             |
| <span style="color:green;">5</span>     | <span style="color:red;">Joseph Sky</span>     | <span style="color:green;">33745</span>   | <span style="color:green;">Male</span>       | <span style="color:red;">7.1</span>             |
| <span style="color:green;">6</span>     | <span style="color:red;">Alison Moon</span>    | <span style="color:green;">22983</span>   | <span style="color:green;">Female</span>     | <span style="color:red;">6.2</span>             |
Here, `ZIP` and `Gender` (in green) are considered public attributes, while `Name` and `Blood Sugar` (in red) are private. We show how statistical queries (such as sums or averages) over `Blood Sugar` can potentially expose an individual's blood sugar value - even when each query is computed across multiple records[^1].

Following SQL syntax, a query consists of a `WHERE` clause, which filters records from the dataset, and an aggregate function that summarizes a private attribute over the selected records. For example, the query `SELECT SUM("Blood sugar") WHERE Gender = "Male"` uses the predicate `Gender = "Male"` and the aggregate function `SUM`, returning the result 16.6 (the sum of blood sugar values for males: $4.3 + 5.2 + 7.1$).

It is not hard to see that answering the following queries reveal the blood sugar level of the second record (Jeremy Doe), even though each query covers at least two records.

1. Query 1: `SELECT SUM("Blood sugar") FROM Dataset`; Result: 32.1
2. Query 2: `SELECT SUM("Blood sugar") FROM Dataset WHERE Gender = "Female"`; Result: 15.5
3. Query 3: `SELECT SUM("Blood sugar") FROM Dataset WHERE ZIP > 32000 and ZIP < 35000 AND Gender = "Male"`; Result: 11.4

To see why, notice that Query 1 and 2 reveal the total blood sugar level of all males (subtract the result of Query 2 from that of Query 1), and Query 3 selects only Record 1 and 5. Therefore, taking the difference of the result of Query 3 and the total blood sugar of all males, we obtain
the exact blood sugar level of Record 2.

To make it more formal, let $x_i$ denote the unknown (private) Blood sugar value of record $i$. Each query along with its result can be represented by a linear equation as follows:

  1. Query 1: $x_1 + x_2 + x_3 + x_4 + x_5 + x_6 = 32.1$
  2. Query 2: $x_3 + x_4 + x_6 = 15.5$
  3. Query 3: $x_1 + x_5 = 11.4$
  
  Or, with matrix notation: $$\begin{aligned}

\mathbf{A}\mathbf{x} = \mathbf{b}

\end{aligned}$$ where $$\mathbf{A} = \begin{bmatrix}
1 & 1 & 1 & 1 & 1 & 1 \\
0 & 0 & 1 & 1 & 0 & 1 \\
1 & 0 & 0 & 0 & 1 & 0
\end{bmatrix}$$$$\mathbf{x} = (x_1, x_2, \ldots, x_6)^\mathsf{T}$$$$\mathbf{b} = (32.1, 15.5, 11.4)^\mathsf{T}$$
The task is to check if *any* $x_i$ can be unambiguously determined, or, in the worst case, whether the entire system of equations has a unique solution (i.e., all $x_i$ can be uniquely identified). If that is the case, the queries specified by matrix $\mathbf{A}$ cannot be answered without revealing the private attribute value of any record. 
## Which records are exposed?

Given a linear system of equations $\mathbf{A}\mathbf{x} = \mathbf{b}$, where $\mathbf{b} = \{b_1, \ldots, b_m\}$, $\mathbf{A} \in \mathbb{R}^{m,n}$, and row $i$ of $\mathbf{A}$ corresponds to query $q_i$. In general, an unknown $x_i$ can be unambiguously determined if the following conditions are met:

1. The system must be consistent, meaning that the vector $\mathbf{b}$ is a linear combination of the columns of $\mathbf{A}$. In other words, $\mathbf{b}$ must lie within the column space of $\mathbf{A}$. This is verified by checking if $\text{rank}(\mathbf{A}) = \text{rank}([\mathbf{A} \, | \, \mathbf{b}])$, where $[\mathbf{A} \, | \, \mathbf{b}]$ is the augmented matrix formed by appending $\mathbf{b}$ as a column to $\mathbf{A}$.

2. The $i$th column of $\mathbf{A}$ must be **linearly independent** of the remaining columns. To check this, remove the $i$th column from $\mathbf{A}$ and compute the rank of the resulting matrix. If the rank decreases, it confirms that the $i$th column is linearly independent.
  
In the above example, each query was answered faithfully, hence the linear system is consistent. To illustrate the second condition, let's reformulate $\mathbf{A}\mathbf{x} = \mathbf{b}$ as:
$$

x_1\cdot

\begin{bmatrix}

1 \\

0 \\

1

\end{bmatrix}

+

x_2\cdot

\begin{bmatrix}

1 \\

0 \\

0

\end{bmatrix}

+

x_3\cdot

\begin{bmatrix}

1 \\

1 \\

0

\end{bmatrix}

+

x_4\cdot

\begin{bmatrix}

1 \\

1 \\

0

\end{bmatrix}

+

x_5\cdot

\begin{bmatrix}

1 \\

0 \\

1

\end{bmatrix}

+

x_6\cdot

\begin{bmatrix}

1 \\

1 \\

0

\end{bmatrix}

=

\begin{bmatrix}

32.1 \\

15.5 \\

11.4

\end{bmatrix}

$$
This expression shows that vector $\mathbf{b}$ is a linear combination of the columns of the matrix $\mathbf{A}$, weighted by the unknowns $x_1, \ldots, x_6$.

Here, the contribution of the second column $(1, 0, 0)^\mathsf{T}$ to the linear combination $\mathbf{b}$ is unique and cannot be replicated by any other column or combination of columns from the matrix $\mathbf{A}$.

Indeed, adding this column to matrix
$$\begin{bmatrix}

1 & 1 & 1 & 1 & 1 \\

0 & 1 & 1 & 0 & 1 \\

1 & 0 & 0 & 1 & 0

\end{bmatrix}$$
increases its rank by one:

``` python
import numpy as np
from numpy.linalg import matrix_rank

# Define the matrix A and vector b
A = np.array([
[1, 1, 1, 1, 1, 1],
[0, 0, 1, 1, 0, 1],
[1, 0, 0, 0, 1, 0]
])

b = np.array([32.1, 15.5, 11.4])

for col in range(A.shape[1]):
	print (f"Unknown x_{col+1} can be uniquely determined:",
	matrix_rank(A) != matrix_rank(np.delete(A, row_num, axis=1))
```

which produces:

```
Unknown x_1 can be uniquely determined: False
Unknown x_2 can be uniquely determined: True
Unknown x_3 can be uniquely determined: False
Unknown x_4 can be uniquely determined: False
Unknown x_5 can be uniquely determined: False
Unknown x_6 can be uniquely determined: False
```

However, this uniqueness does not apply to the other columns which appear multiple times in the matrix:
$$

(x_1 + x_5)\cdot

\begin{bmatrix}

1 \\

0 \\

1

\end{bmatrix}

+

x_2\cdot

\begin{bmatrix}

1 \\

0 \\

0

\end{bmatrix}

+

(x_3 + x_4 + x_6)\cdot

\begin{bmatrix}

1 \\

1 \\

0

\end{bmatrix}

=

\begin{bmatrix}

32.1 \\

15.5 \\

11.4

\end{bmatrix}

$$
Since these three column vectors are linearly indepedent, we can uniquely determine their weights: $x_2$, the sum of $x_1$ and $x_5$, and the sum of $x_3$, $x_4$, and $x_6$. However, we cannot tell how these sums are shared among their respective variables. This implies that the original linear system as a whole still has an infinite number of solutions. 

Not surprisingly, *every* unknown $x_i$ can be unambiguously determined in a linear system if it is consistent and all columns are linearly indepedent: $\text{rank}([\mathbf{A}|\mathbf{b}]) = \text{rank}(\mathbf{A}) = n$, meaning $\mathbf{A}$ is square and full rank. In this case, the system has a unique solution given by $\mathbf{x} = \mathbf{A}^{-1}\mathbf{b}$.

If $\text{rank}([\mathbf{A}|\mathbf{b}]) = \text{rank}(\mathbf{A}) < n$, the system is underdetermined because there are fewer linearly independent queries (rows) than unknowns. As a result, the solution space typically has infinitely many solutions. Constraints like requiring $\mathbf{x}$ to have integer values or sparsity (at most $\text{rank}(\mathbf{A})$ non-zero entries in $\mathbf{x}$) can reduce the solution space to a finite set.
 
## Solving the equations

There are a couple of techniques to solve (or approximate) a linear system; any matrix decomposition technique (e.g., Gauss elimination) can be applied on the augmented matrix $\mathbf{A}^*$. Here, we rather use a more versatile optimization technique which provides an approximation of $\mathbf{x}$ irrespective of the rank of $[\mathbf{A}|\mathbf{b}]$, and can be extended with any extra (linear) constraints[^2]: $$\begin{aligned}

\mathbf{x}' = \arg\min_{\mathbf{x} \in \mathbb{R}^n} \quad & ||\mathbf{A}\mathbf{x} - \mathbf{b}||_2

% \textrm{such that} \quad & 0 \leq x_i \leq 1 \notag

\end{aligned}$$ where $||\cdot ||_2$ denotes the Euclidean distance ($L_2$-norm).

The solution of this optimization problem has a closed form $\mathbf{x}' = \mathbf{\hat{A}} \mathbf{b}$, where $\mathbf{\hat{A}}$ is the *pseudo-inverse* (or [Moore-Penrose inverse](https://en.wikipedia.org/wiki/Moore–Penrose_inverse)) of $\mathbf{A}$. If $\mathbf{A}$ is invertible, then $\mathbf{A}^{-1} = \mathbf{\hat{A}}$. The reconstruction error $||\mathbf{x}' - \mathbf{x}||_2$ depends on $\text{rank}(\mathbf{A})$ (the number of linearly indepedent queries). Exact reconstruction $(\mathbf{x}' = \mathbf{x})$ is only guaranteed if $\mathbf{A}$ has full rank and $\text{rank}(\mathbf{A}) = \text{rank}([\mathbf{A}|\mathbf{b}])$, as discussed earlier. This optimization technique is widely known as [*Ordinary Least Squares* (OLS)](https://en.wikipedia.org/wiki/Ordinary_least_squares#:~:text=In%20statistics%2C%20ordinary%20least%20squares,of%20the%20squares%20of%20the) or linear regression in machine learning terminology[^3]. OLS and pseudo-inverse computations are supported by many linear algebra libraries and machine learning frameworks, including [scikit-learn](hhttps://scikit-learn.org/1.5/modules/generated/sklearn.linear_model.LinearRegression.html), [torch](https://pytorch.org/docs/stable/generated/torch.linalg.lstsq.html), [numpy](https://numpy.org/doc/2.1/reference/generated/numpy.linalg.lstsq.html).

For example, in numpy:

``` python
# Solve the system using NumPy's least-squares method (since A is not square)
x_prime = np.linalg.lstsq(A, b, rcond=None)[0]

print ("Solution:", x_prime)
```

and the output:

``` python
Solution: [5.7 5.2 5.16666667 5.16666667 5.7 5.16666667]
```

where Jeremy Doe's blood sugar level ($x_2$) is accurately approximated. Notice that the sums $x_1 + x_5 = 11.4$ and $x_3+x_4+x_6 = 15.5$ are correct. OLS finds the solution with the smallest $L_2$-norm, which occurs when the sums are evenly distributed among their respective variables.

There is one potential caveat: the private attribute $\mathbf{x}$ may sometimes be binary in practice, such as indicating whether a patient has AIDS or not. Such an integer constraint turns the problem into an instance of Integer Linear Programming (ILP) which is NP-complete in general. Rather than tackling this computationally hard problem directly, we can round each element of $\mathbf{x}'$ (the OLS solution) to the nearest integer, obtaining a final approximation of the original vector $\mathbf{x}$.

Finally, while SUM and AVG are linear queries that can be efficiently audited using tools from linear algebra, auditing non-linear queries such as MEDIAN, MAX, and MIN is much more challenging. In fact, [checking disclosure](https://theory.stanford.edu/~nmishra/Papers/surveyQueryAuditingTechniquesDataPrivacy.pdf) for such queries may not even be feasible in polynomial time of the dataset size $n$. In these cases, we can resort to [heuristics such as SAT solvers](https://dl.acm.org/doi/10.1145/3287287).
# Notes

[^1]: *Seemingly, solving this linear system for $x_2$ without associating it with any flesh-and-bone individual (i.e., Jeremy Doe) should not imply any privacy breach according to GDPR. GDPR says that any information is personal if it is related to an identified or identifiable person. If the attacker can only access the public attributes (in blue), then it probably cannot link $x_2$ to Jeremy Doe. On the other hand, the attacker can always have some extra background knowledge which may help with singling out Jeremy's record even without accessing its \"Name\" attribute. For example, it may know from another source (like Facebook) that Jeremy went to the hospital and his ZIP starts with 438\*\*. If such additional knowledge is reasonably accessible, this post aims to demonstrate that query results themselves (e.g., the outcomes of Query 1, 2, and 3 above) can be considered personal or sensitive under the GDPR*.

[^2]: *For example, a linear constraint is when $\mathbf{x}$ comes from a bounded range. In that case, the optimization problem (OLS) remains a convex quadratic linear program, which is easy to solve with specialized solvers.*

[^3]: *Although OLS is polynomial in dataset size $n$, it becomes quite slow (and memory intensive) for large datasets. Iterative methods can find an accurate approximative solution faster with little overhead (such as Kaczmarz's method or any gradient-based technique)*