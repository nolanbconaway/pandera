# pandera

Validating pandas data structures for people seeking correct things.

A light-weight and flexible validation package for
<a href="http://pandas.pydata.org" target="_blank">pandas</a> data structures,
built on top of
<a href="https://github.com/keleshev/schema" target="_blank">schema</a>,
a powerful Python data structure validation tool. API inspired by
<a href="https://tmiguelt.github.io/PandasSchema" target="_blank">pandas-schema</a>.


## Why?

Because pandas data structures hide a lot of information, and explicitly
validating them in production-critical or reproducible research settings is
a good idea.

And it also makes it easier to review pandas code :)


## Install

```
pip install pandera
```


## Example Usage

### `DataFrameSchema`

```python
import pandas as pd

from pandera import Column, DataFrameSchema, PandasDtype, Validator


# validate columns
schema = DataFrameSchema([
    Column("column1", PandasDtype.Int, Validator(lambda x: 0 <= x <= 10)),
    Column("column2", PandasDtype.Float, Validator(lambda x: x < -1.2)),
    Column("column3", PandasDtype.String,
           Validator(lambda x: x.startswith("value_")))
])

df = pd.DataFrame({
    "column1": [1, 4, 0, 10, 9],
    "column2": [-1.3, -1.4, -2.9, -10.1, -20.4],
    "column3": ["value_1", "value_2", "value_3", "value_2", "value_1"]
})

validated_df = schema.validate(df)
print(validated_df)

#     column1  column2  column3
#  0        1     -1.3  value_1
#  1        4     -1.4  value_2
#  2        0     -2.9  value_3
#  3       10    -10.1  value_2
#  4        9    -20.4  value_1
```

#### Errors

If the dataframe does not pass validation checks, `pandera` provides useful
error messages. An `error` argument can also be supplied to `Validator` for
custom error messages.

```python
simple_schema = DataFrameSchema([
    Column("column1", PandasDtype.Int,
           Validator(lambda x: 0 <= x <= 10, error="range checker [0, 10]"))
])

# validation rule violated
fail_check_df = pd.DataFrame({
    "column1": [-20, 5, 10, 30],
})

simple_schema.validate(fail_check_df)

# schema.SchemaError: series failed element-wise validator 0:
# <lambda>: range checker [0, 10]
# failure cases: {0: -20, 3: 30}


# column name mis-specified
wrong_column_df = pd.DataFrame({
    "foo": ["bar"] * 10,
    "baz": [1] * 10
})

simple_schema.validate(wrong_column_df)

#  SchemaError: column 'column1' not in dataframe
#     foo  baz
#  0  bar    1
#  1  bar    1
#  2  bar    1
#  3  bar    1
#  4  bar    1
```


### `SeriesSchema`

```python
from pandera import SeriesSchema

# specify multiple validators
schema = SeriesSchema(
    PandasDtype.String, [
        Validator(lambda x: "foo" in x),
        Validator(lambda x: x.endswith("bar")),
        Validator(lambda x: len(x) > 3)])

schema.validate(pd.Series(["1_foobar", "2_foobar", "3_foobar"]))

#  0    1_foobar
#  1    2_foobar
#  2    3_foobar
#  dtype: object
```


### Aggregate Validators

If you need to make basic statistical assertions about a column, use the
`element_wise=False` keyword argument. The signature of validators then becomes
`pd.Series -> bool|pd.Series[bool]`, where the function has access to the
Series API:

```python
schema = DataFrameSchema([
    Column("a", PandasDtype.Int,
           [Validator(lambda s: s.mean() > 5, element_wise=True),
            Validator(lambda s: s.median() >= 6)])
])

df = pd.DataFrame({"a": [4, 4, 5, 6, 6, 7, 8, 9]})
schema.validate(df)
```

To specify validators that apply both element- and series-wise, supply
a list of `Validators` with the appropriate `element_wise` argument (default
is `element_wise=True`.

```python
schema = DataFrameSchema([
    Column("a", PandasDtype.Int,
           [Validator(lambda s: s.mean() > 5, element_wise=False),
            Validator(lambda x: x > 0)])
])
```


### `validate_input`

Decorator that validates input pandas DataFrame/Series before entering the
wrapped function.

```python
from pandera import validate_input


df = pd.DataFrame({
    "column1": [1, 4, 0, 10, 9],
    "column2": [-1.3, -1.4, -2.9, -10.1, -20.4],
})

in_schema = DataFrameSchema([
    Column("column1", PandasDtype.Int, Validate(lambda x: 0 <= x <= 10)),
    Column("column2", PandasDtype.Float, Validate(lambda x: x < -1.2)),
])


# provide the argument name as a string
@validate_input(in_schema, "dataframe")
def preprocessor(dataframe):
    dataframe["column4"] = dataframe["column1"] + dataframe["column2"]
    return dataframe

# or integer representing index in the positional arguments. If second argument
# is not specified, assumes that the first argument is dataframe/series.
@validate_input(in_schema, 0)
def preprocessor(dataframe):
    ...

preprocessed_df = preprocessor(df)
print(preprocessed_df)

#  Output:
#     column1  column2  column3  column4
#  0        1     -1.3  value_1     -0.3
#  1        4     -1.4  value_2      2.6
#  2        0     -2.9  value_3     -2.9
#  3       10    -10.1  value_2     -0.1
#  4        9    -20.4  value_1    -11.4
```


### `validate_output`

The same as `validate_input`, but this decorator checks the output
DataFrame/Series of the decorated function.

```python
from pandera import validate_output


# assert that all elements in "column1" are zero
out_schema = DataFrameSchema([
    Column("column1", PandasDtype.Int, lambda x: x == 0)])


# by default assumes that the pandas DataFrame/Schema is the only output
@validate_output(out_schema)
def zero_column_1(df):
    df["column1"] = 0
    return df


# you can also specify in the index of the argument if the output is list-like
@validate_output(out_schema, 1)
def zero_column_1_arg(df):
    df["column1"] = 0
    return "foobar", df


# or the key containing the data structure to verify if the output is dict-like
@validate_output(out_schema, "out_df")
def zero_column_1_dict(df):
    df["column1"] = 0
    return {"out_df": df, "out_str": "foobar"}


# for more complex outputs, you can specify a function
@validate_output(out_schema, lambda x: x[1]["out_df"])
def zero_column_1_custom(df):
    df["column1"] = 0
    return ("foobar", {"out_df": df})


zero_column_1(preprocessed_df)
zero_column_1_arg(preprocessed_df)
zero_column_1_dict(preprocessed_df)
zero_column_1_custom(preprocessed_df)
```

Tests
-----

```
pip install pytest
pytest tests
```

Issues
------

Go <a href="https://github.com/cosmicBboy/pandera/issues" target="_blank">here</a>
to submit feature requests or bugfixes.
