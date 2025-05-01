# Data Science Tips
> log of list data science tips I learned when learning data science.

## Panda category
To reduce memory usage, for data that is repeatable or categorical, use dtype category.
For example table below

```table
name | status
andrew | married
james | single
barbara | married
```

the status can be converted to category, so the memory usage will be reduced.
```python
import pandas as pd

df = pd.DataFrame([
    {'name': 'andrew', 'status': 'married'},
    {'name': 'james', 'status': 'single'},
    {'name': 'barbara', 'status': 'married'}
], dtype={'status': 'category'})

# or can be also
df['status'] = df['status'].astype('category')

# check the byte size of the column
print(df['status'].nbytes)
```

