Data pages cannot written to disk until the log records describing those changes are written first.

Buffer pool is the in-memory cache of data pages.
An update transaction is taken place
First we take an exclusive lock at the table level and then we take an exclusive lock at the page level.

