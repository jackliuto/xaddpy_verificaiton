# Python Implementation of XADD

This repository implements the Python version of XADD (eXtended Algebraic Decision Diagrams) which was first introduced in [Sanner et al. (2011)](https://arxiv.org/pdf/1202.3762.pdf); you can find the original Java implementation from [here](https://github.com/ssanner/xadd-inference). 

Our Python XADD code uses [Sympy](https://github.com/sympy/sympy) for symbolically maintaining all variables and related operations, and [PULP](https://github.com/coin-or/pulp) is used for pruning unreachable paths.  Note that we only check linear conditionals.  If you have Gurobi installed and configured in the conda environment, then PULP will use Gurobi for solving (MI)LPs; otherwise, the default solver ([CBC](https://github.com/coin-or/Cbc)) is going to be used.

Note that the implementation for [EMSPO](https://proceedings.mlr.press/v162/jeong22a/jeong22a.pdf) --- Exact symbolic reduction of linear Smart Predict+Optimize to MILP (Jeong et al., ICML-22) --- has been moved to the branch [emspo](https://github.com/jihwan-jeong/xaddpy/tree/emspo).

You can find the implementation for the [CPAIOR-23](https://ssanner.github.io/papers/cpaior23_dblpsve.pdf) work --- A Mixed-Integer Linear Programming Reduction of Disjoint Bilinear Programs via Symbolic
Variable Elimination --- in [examples/dblp](examples/dblp).

## Installation

**Load your Python virtual environment then type the following commands for package installation**

```shell
pip install xaddpy

# Optional: if you want to use Gurobi for the 'reduce_lp' method
# that prunes out unreachable partitions using LP solvers
pip install gurobipy    # If you have a license
```

## Using xaddpy

You can find useful XADD usecases in the [xaddpy/tests/test_bool_var.py](xaddpy/tests/test_bool_var.py) and [xaddpy/tests/test_xadd.py](xaddpy/tests/test_xadd.py) files. Here, we will first briefly discuss different ways to build an initial XADD that you want to work with. 

### Loading from a file

If you know the entire structure of an initial XADD, then you can create a text file specifying the XADD and load it using the `XADD.import_xadd` method. It's important that, when you manually write down the XADD you have, you follow the same syntax rule as in the example file shown below.

Below is a part of the XADD written in [xaddpy/tests/ex/bool_cont_mixed.xadd](xaddpy/tests/ex/bool_cont_mixed.xadd):
```
...
        ( [x - y <= 0]
            ( [2 * x + y <= 0]
                ([x])
                ([y])
            )
            (b3
                ([2 * x])
                ([2 * y])
            )
        )
...
```
Here, `[x-y <= 0]` defines a decision expression; its true branch is another node with the decision `[2 * x + y <= 0]`, while the decision of the false branch is a Boolean variable `b3`. Similarly, if `[2 * x + y <= 0]` holds true, then we get the leaf value `[x]`; otherwise, we get `[y]`. Inequality decisions and leaf values are wrapped with brackets, while you can directly put the variable name in the case of a Boolean decision. A Sympy `Symbol` object will be created for each unique variable.

To load this XADD, you can do the following:
```python
from xaddpy import XADD
context = XADD()
fname = 'xaddpy/tests/ex/bool_cont_mixed.xadd'

orig_xadd = context.import_xadd(fname)
```
Following the Java implementation, we call the instantiated XADD object `context`. This object maintains and manages all existing/new nodes and decision expressions. For example, `context._id_to_node` is a Python dictionary that stores mappings from node IDs (int) to the corresponding `Node` objects. For more information, please refer to the constructor of the `XADD` class.

To check whether you've got the right XADD imported, you can print it out.
```python
print(f"Imported XADD: \n{context.get_repr(orig_xadd)}")
```
The `XADD.get_repr` method will return `repr(node)` and the string representation of each XADD node is implemented in [xaddpy/xadd/node.py](xaddpy/xadd/node.py). Beware that the printing method can be slow for a large XADD.

### Recursively building an XADD
Another way of creating an initial XADD node is by recursively building it with the `apply` method. A very simple example would be something like this:

```python
from xaddpy import XADD
import sympy as sp

context = XADD()

x_id = context.convert_to_xadd(sp.Symbol('x'))
y_id = context.convert_to_xadd(sp.Symbol('y'))

sum_node_id = context.apply(x_id, y_id, op='add')
comp_node_id = context.apply(sum_node_id, y_id, op='min')

print(f"Sum node:\n{context.get_repr(sum_node_id)}\n")
print(f"Comparison node:\n{context.get_repr(comp_node_id)}")
```
You can check that the print output shows
```
Sum node:
( [x + y] ) node_id: 9

Comparison node:
( [x <= 0] (dec, id): 10001, 10
         ( [x + y] ) node_id: 9 
         ( [y] ) node_id: 8 
)
```
which is the expected outcome!

Check out a much more comprehensive example demonstrating the recursive construction of a nontrivial XADD from here: [pyRDDLGym/XADD/RDDLModelXADD.py](https://github.com/ataitler/pyRDDLGym/blob/01955ee7bca2861124709c116f419f2927c04a89/pyRDDLGym/XADD/RDDLModelXADD.py#L124).

### Directly creating an XADD node
Finally, you might want to build a constant node, an arbitrary decision expression, and a Boolean decision directly. To this end, let's consider building the following XADD: 

```
([b]
    ([1])
    ([x + y <= 0]
        ([0])
        ([2])
    )
)
```

To do this, we will first create an internal node whose decision is `[x + y <= 0]`, the low   and the high branches are `[0]` and `[2]` (respectively). Using Sympy's `S` function (or you can use `sympify`), you can turn an algebraic expression involving variables and numerics into a symbolic expression. Given this decision expression, you can get its unique index using `XADD.get_dec_expr_index` method. You can use the decision ID along with the ID of the low and high nodes connected to the decision to create the corresponding decision node, using `XADD.get_internal_node`.

```python
import sympy as sp
from xaddpy import XADD

context = XADD()

# Get the unique ID of the decision expression
dec_expr: sp.Basic = sp.S('x + y <= 0')
dec_id, is_reversed = context.get_dec_expr_index(dec_expr, create=True)

# Get the IDs of the high and low branches: [0] and [2], respectively
high: int = context.get_leaf_node(sp.S(0))
low: int = context.get_leaf_node(sp.S(2))
if is_reversed:
    low, high = high, low

# Create the decision node with the IDs
dec_node_id: int = context.get_internal_node(dec_id, low=low, high=high)
print(f"Node created:\n{context.get_repr(dec_node_id)}")
```

Note that `XADD.get_dec_expr_index` returns a boolean variable `is_reversed` which is `False` if the canonical decision expression of the given decision has the same inequality direction. If the direction has changed, then `is_reversed=True`; in this case, low and high branches should be swapped.

Another way of creating this node is to use the `XADD.get_dec_node` method. This method can only be used when the low and high nodes are terminal nodes containing leaf expressions.

```python
dec_node_id = context.get_dec_node(dec_expr, low_val=sp.S(2), high_val=sp.S(0))
```

Note also that you need to wrap constants with the `sympy.S` function to turn them into `sympy.Basic` objects.

Now, it remains to create a decision node with the Boolean variable `b` and connect it to its low and high branches. 

```python
b = sp.Symbol('b', bool=True)
dec_b_id, _ = context.get_dec_expr_index(b, create=True)
```

First of all, you need to provide `bool=True` as a keyword argument when instantiating a Sympy `Symbol` corresponding to a Boolean variable. When you instantiate multiple Boolean variables, you can use `b1, b2, b3 = sp.symbols('b1 b2 b3', bool=True)`. If you didn't specify `bool=True`, then the variable won't be recognized as a Boolean variable in XADD operations!

Once you have the decision ID, we can finally link this decision node with the node created earlier. 

```python
high: int = context.get_leaf_node(sp.S(1))
node_id: int = context.get_internal_node(dec_b_id, low=dec_node_id, high=high)
print(f"Node created:\n{context.get_repr(node_id)}")
```
And we get the following print outputs.
```
Output:
Node created:
( [b]   (dec, id): 2, 9
        ( [1] ) node_id: 1 
        ( [x + y <= 0]  (dec, id): 10001, 8
                ( [0] ) node_id: 0 
                ( [2] ) node_id: 7 
        )  
) 
```

### XADD Operations

#### XADD.apply(id1: int, id2: int, op: str)
You can perform the `apply` operation to two XADD nodes with IDs `id1` and `id2`. Below is the list of the supported operators (`op`):

**Non-Boolean operations**
- 'max', 'min'
- 'add', 'subtract'
- 'prod', 'div'

**Boolean operations**
- 'and'
- 'or'

**Relational operations**
- '!=', '=='
- '>', '>='
- '<', '<='

#### XADD.unary_op(node_id: int, op: str) (unary operations)
You can also apply the following unary operators to a single XADD node recursively (also check `UNARY_OP` in [xaddpy/utils/global_vars.py](xaddpy/utils/global_vars.py)). In this case, an operator will be applied to each and every leaf value of a given node. Hence, the decision expressions will remain unchanged.

- 'sin, 'cos', 'tan'
- 'sinh', 'cosh', 'tanh'
- 'exp', 'log', 'log2', 'log10', 'log1p'
- 'floor', 'ceil'
- 'sqrt', 'pow'
- '-', '+'
- 'sgn' (sign function... sgn(x) = 1 if x > 0; 0 if x == 0; -1 otherwise)
- '~' (negation)

The `pow` operation requires an additional argument specifying the exponent.

#### XADD.evaluate(node_id: int, bool_assign: dict, cont_assign: bool, primitive_type: bool)
When you want to assign concrete values to Boolean and continuous variables, you can use this method. An example is provided in the `test_mixed_eval` function defined in [xaddpy/tests/test_bool_var.py](xaddpy/tests/test_bool_var.py).

As another example, let's say we want to evaluate the XADD node defined a few lines above.
```python
x, y = sp.symbols('x y')

bool_assign = {b: True}
cont_assign = {x: 2, y: -1}

res = context.evaluate(node_id, bool_assign=bool_assign, cont_assign=cont_assign)
print(f"Result: {res}")
```

In this case, `b=True` will directly leads to the leaf value of `1` regardless of the assignment given to `x` and `y` variables. 

```python
bool_assign = {b: False}
res = context.evaluate(node_id, bool_assign=bool_assign, cont_assign=cont_assign)
print(f"Result: {res}")
```

If we change the value of `b`, we can see that we get `2`. Note that you have to make sure that all symbolic variables get assigned specific values; otherwise, the function will return `None`. 

#### XADD.substitute(node_id: int, subst_dict: dict)
If instead you want to assign values to a subset of symbolic variables while leaving the other variables as-is, you can use the `substitute` method. Similar to `evaluate`, you need to pass in a dictionary mapping Sympy `Symbol`s to their concrete values.

For example,

```python
subst_dict = {x: 1}
node_id_after_subs = context.substitute(node_id, subst_dict)
print(f"Result:\n{context.get_repr(node_id_after_subs)}")
```
which outputs
```
Result:
( [b]   (dec, id): 2, 16
        ( [1] ) node_id: 1 
        ( [y + 1 <= 0]  (dec, id): 10003, 12
                ( [0] ) node_id: 0 
                ( [2] ) node_id: 7 
        )  
) 
```
as expected. 

#### XADD.collect_vars(node_id: int)
If you want to extract all Boolean and continuous Sympy variables existing in an XADD node, you can use this method.

```python
var_set = context.collect_vars(node_id)
print(f"var_set: {var_set}")
```
```
Output:
var_set: {y, b, x}
```

This method can be useful to figure out which variables need to have values assigned in order to evaluate a given XADD node.

#### XADD.make_canonical(node_id: int)
This method gives a canonical order to an XADD that is potentially unordered. Note that the `apply` method already calls `make_canonical` when the `op` is one of `('min', 'max', '!=', '==', '>', '>=', '<', '<=', 'or', 'and')`.


## Citation

Please use the following bibtex for citations:

```
@InProceedings{pmlr-v162-jeong22a,
  title = 	 {An Exact Symbolic Reduction of Linear Smart {P}redict+{O}ptimize to Mixed Integer Linear Programming},
  author =       {Jeong, Jihwan and Jaggi, Parth and Butler, Andrew and Sanner, Scott},
  booktitle = 	 {Proceedings of the 39th International Conference on Machine Learning},
  pages = 	 {10053--10067},
  year = 	 {2022},
  volume = 	 {162},
  series = 	 {Proceedings of Machine Learning Research},
  month = 	 {17--23 Jul},
  publisher =    {PMLR},
}
```