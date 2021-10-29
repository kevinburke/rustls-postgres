# Adding Rustls to Postgres

See NOTES.md for my day-to-day notes.

There are six functions in the client and six functions in the server that we
need to implement - see NOTES.md for details.

### Implementation steps

1) Get the Postgres frontend compiling with stub functions that don't do
anything. The backend will still compile using NSS (ie we'll compile in both
libraries until we have them both working.)

2) Try to implement the functions.

3) Get it working on other operating systems, add docs, etc.

I'm going to push my changes to github.com/kevinburke/postgres.

### Getting the code up and running

The first thing you want to do is apply [Daniel Gustafsson's NSS
patches][patches] to `tip` of the upstream project.

0. Install prerequisites - we bootstrap using NSS, so you'll need a valid check
   out of that for now.

```
brew install nspr nss
```

    Clone github.com/rustls/rustls-ffi and run:

    ```
    make clean && make && make install DESTDIR='./target'
    ```

1. Check out github.com/postgres/postgres and (as of October 29) pull the
   latest commit on the master branch.

2. Find the latest patchset in `patchsets` here or download from the mailing
   list.

3. Apply each patch from the NSS patchset:

    for patch in ../../kevinburke/postgres-rustls/patchsets/2021-10-29-gustafsson-mailing-list/*.patch; do git am $patch; done;

4. Apply the latest changes from the `rustls-4` branch on the
github.com/kevinburke/postgres remote, on top of this patch, rebasing down to
one commit if you need.

5. Run "make clean" to clear out any artifacts that may have been remaining from
   the previous build

6. Compile, make, install. This will install binaries into $HOME/pq. Note you
will have to change some of these paths - the ones that say e.g. /Users/kevin.

```
LDFLAGS="-L/usr/local/opt/nss/lib -L/usr/local/opt/nspr/lib -L/Users/kevin/src/github.com/rustls/rustls-ffi/target/lib" \
    CFLAGS="-framework Security" \
    CPPFLAGS="-framework Security -I/usr/local/opt/nss/include -I/usr/local/opt/nspr/include -I/Users/kevin/src/github.com/rustls/rustls-ffi/target/include" \
    ./configure --prefix=$HOME/pq --with-ssl=rustls && \
    gmake && \
    PATH=/bin:$PATH gmake install
```

[patches]: https://postgrespro.com/list/thread-id/2492594#1538bd8894fc723d97d5c41b9d4bb8feb9db0d43.camel@j-davis.com
