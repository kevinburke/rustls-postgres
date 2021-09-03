# Adding Rustls to Postgres

See NOTES.md for my day-to-day notes.

There are six functions in the client and six functions in the server that we
need to implement - see NOTES.md for details.

### Implementation steps

1) Get Postgres compiling with stub functions that don't do anything.

2) Try to implement the functions.

3) Get it working on other operating systems, add docs, etc.

I'm going to push my changes to github.com/kevinburke/postgres.
