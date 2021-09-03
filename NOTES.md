### September 2, 2021

Need to run "make clean" before switching to the NSS branch from master or vice
versa.

There's an error with `cp` in Rust coreutils: https://github.com/uutils/coreutils/issues/2631

Compiling the NSS branch with:

```
LDFLAGS="-L/usr/local/opt/nss/lib -L/usr/local/opt/nspr/lib" CPPFLAGS="-I/usr/local/opt/nss/include -I/usr/local/opt/nspr/include" ./configure --prefix=$HOME/pq --with-ssl=nss && gmake && PATH=/bin:$PATH gmake install check
```

I am running into the following error:

```
gmake[5]: Entering directory '/Users/kevin/src/github.com/postgres/postgres/src/port'
gmake[5]: Nothing to be done for 'all'.
gmake[5]: Leaving directory '/Users/kevin/src/github.com/postgres/postgres/src/port'
gmake -C ../../../src/common all
gmake[5]: Entering directory '/Users/kevin/src/github.com/postgres/postgres/src/common'
gmake[5]: Nothing to be done for 'all'.
gmake[5]: Leaving directory '/Users/kevin/src/github.com/postgres/postgres/src/common'
! nm -A -u libpq.5.15.dylib 2>/dev/null | grep -v __cxa_atexit | grep exit
libpq.5.15.dylib: _exit
gmake[4]: *** [Makefile:123: libpq-refs-stamp] Error 1
gmake[4]: Leaving directory '/Users/kevin/src/github.com/postgres/postgres/src/interfaces/libpq'
gmake[3]: *** [Makefile:17: install-libpq-recurse] Error 2
gmake[3]: Leaving directory '/Users/kevin/src/github.com/postgres/postgres/src/interfaces'
gmake[2]: *** [Makefile:42: install-interfaces-recurse] Error 2
gmake[2]: Leaving directory '/Users/kevin/src/github.com/postgres/postgres/src'
gmake[1]: *** [GNUmakefile:11: install-src-recurse] Error 2
gmake[1]: Leaving directory '/Users/kevin/src/github.com/postgres/postgres'
```

### September 1, 2021

Here's a patch from Daniel Gustafsson adding libnss support to Postgres. The
nice thing about this is that it abstracts (or maybe was already abstracted) the
SSL interfaces behind common functions, so it's easier for us to adopt the same
patterns for crustls as the third TLS library in use.

https://postgrespro.com/list/thread-id/2492594#1538bd8894fc723d97d5c41b9d4bb8feb9db0d43.camel@j-davis.com

For ease of looking at the diff I've opened the most recent patch set as a PR
which you can view here. https://github.com/kevinburke/postgres/pull/2

### Server side

#### Functions that need to be implemented

On the server side (these are all in backend/libpq/be-secure.c)

```
be_tls_init(bool isServerStart)
be_tls_destroy(void)
be_tls_open_server(Port *port)
be_tls_close(Port *port)
be_tls_read(Port *port, void *ptr, size_t len, int *waitfor)
be_tls_write(Port *port, void *ptr, size_t len, int *waitfor)(Port *port, void *ptr, size_t len, int *waitfor)
```

I believe that we'd need the following functions in `crustls` to support each of
these:

#### be_tls_init

Load certs from file:

rustls_client_config_builder_load_roots_from_file

#### be_tls_open_server

rustls_server_connection_new

#### be_tls_read

rustls_connection_read_tls or maybe rustls_connection_read

#### be_tls_write

rustls_connection_write_tls or maybe rustls_connection_write

#### be_tls_close

rustls_connection_free

### Client Side

These are the functions we need to implement, which are in fe-secure.c and
fe-secure-nss.c.

```c
pgtls_init_library(bool do_ssl, int do_crypto)
pgtls_init(PGconn *conn, bool do_ssl, bool do_crypto)
pgtls_open_client(PGconn *conn)
pgtls_close(PGconn *conn)

/*
 *	Read data from a secure connection.
 *
 * On failure, this function is responsible for appending a suitable message
 * to conn->errorMessage.  The caller must still inspect errno, but only
 * to determine whether to continue/retry after error.
 */
ssize_t pgtls_read(PGconn *conn, void *ptr, size_t len)

/*
 * pgtls_write
 *		Write data on the secure socket
 *
 */
ssize_t pgtls_write(PGconn *conn, const void *ptr, size_t len)
```

I think that we need these to implement these functions. The main guide for how
to implement these is tests/client.c in the rustls-ffi codebase.

### pgtls_init_library

I'm not sure that we need this, we can do void like nss does.

### pgtls_init

Just sets various fields on the `*PGConn` object.

### pgtls_close

rustls_connection_free

### pgtls_open_client

rustls_client_connection_new

### pgtls_write

rustls_connection_write
rustls_connection_write_tls

### pgtls_read

rustls_connection_read_tls
