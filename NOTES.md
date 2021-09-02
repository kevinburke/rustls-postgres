### September 1, 2021

Here's a patch from Daniel Gustafsson adding libnss support to Postgres. The
nice thing about this is that it abstracts (or maybe was already abstracted) the
SSL interfaces behind common functions, so it's easier for us to adopt the same
patterns for crustls as the third TLS library in use.

https://postgrespro.com/list/thread-id/2492594#1538bd8894fc723d97d5c41b9d4bb8feb9db0d43.camel@j-davis.com

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
