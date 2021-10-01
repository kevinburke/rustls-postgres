### September 16, 2021

Including unistd.h, _or_ sleeping for 1 second, causes psql to send the right
sequence - with UTF8 and the Q at the beginning.

Sleeping for 1 second makes the output much more reasonable. We see the 335
bytes being read in rustls, but then we see the same preamble being sent later.

### September 14, 2021

Here's the startup sequence of reads and writes from OpenSSL, which is
negotiating TLS 1.3.

"Err 2" here is `SSL_ERROR_WANT_READ`, in which case we set 0 error on the
socket and return 0 bytes read.

```
pgtls_read_pending 0 bytes
pgtls_read_pending 0 bytes
pgtls_write 81 bytes, got err 0
pgtls_write result errno 0, returned n 81
pgtls_read_pending 0 bytes
pgtls_read -1 bytes, got err 2, ecode 0
pgtls_read result errno 0, returned n 0
pgtls_read_pending 0 bytes
pgtls_read 335 bytes, got err 0, ecode 0
pgtls_read result errno 0, returned n 335
pgtls_write 200 bytes, got err 0
pgtls_write result errno 0, returned n 200
pgtls_read_pending 0 bytes
pgtls_read 120 bytes, got err 0, ecode 0
pgtls_read result errno 0, returned n 120
pgtls_read -1 bytes, got err 2, ecode 0
pgtls_read result errno 0, returned n 0
```

OpenSSL compilation instructions:

```
LDFLAGS="-L/usr/local/opt/openssl@1.1/lib" CPPFLAGS="-I/usr/local/opt/openssl@1.1/include" ./configure --prefix=$HOME/pq-master --with-openssl && gmake && PATH=/bin:$PATH gmake install
```

why does pgtls_write on Rust want to write/write only 60 bytes?

on master, with the same printing instructions, we write:

"Quserkevindatabasepostgresapplication_namepsqlclient_encodingUTF8"

on Rust branch we write:

^@^@^@<^@^C^@^@user^@kevin^@database^@postgres^@application_name^@psql^@^@

### September 7, 2021

Needed to also implement these functions in the frontend

```
Undefined symbols for architecture x86_64:
  "_PQgetssl", referenced from:
     -exported_symbol[s_list] command line option
  "_PQsslAttribute", referenced from:
     -exported_symbol[s_list] command line option
  "_PQsslAttributeNames", referenced from:
     -exported_symbol[s_list] command line option
  "_PQsslStruct", referenced from:
     -exported_symbol[s_list] command line option
```

Got everything compiling with shim functions! `psql` hangs when I try to connect
to a Postgres database using TLS. OK, great.

`pgtls_open_client` for OpenSSL invokes SSL_connect which is an OpenSSL
function. The methods on it can be found here

/usr/local/Cellar/openssl@1.1/1.1.1l/include/openssl/ssl.h

SSL_connect source code in ssl/ssl_lib.c in openssl

NSS calls this function

```
conn->pr_fd = PR_ImportTCPSocket(conn->sock);
```

which seems to imply that Postgres already has a socket available.

I keep running into this error:

```
gcc -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Werror=vla -Wendif-labels -Wmissing-format-attribute -Wimplicit-fallthrough=3 -Wcast-function-type -Wformat-security -fno-strict-aliasing -fwrapv -fexcess-precision=standard -Wno-format-truncation -Wno-stringop-truncation -O2 -I/usr/local/Cellar/nss/3.69.1/include/nss -I/usr/local/Cellar/nspr/4.32/include/nspr -I/usr/local/Cellar/nspr/4.32/include/nspr -pthread -D_REENTRANT -D_THREAD_SAFE  -dynamiclib -install_name '/Users/kevin/pq/lib/libpq.5.dylib' -compatibility_version 5 -current_version 5.15 -exported_symbols_list exports.list -multiply_defined suppress -o libpq.5.15.dylib  fe-auth-scram.o fe-connect.o fe-exec.o fe-lobj.o fe-misc.o fe-print.o fe-protocol3.o fe-secure.o fe-trace.o legacy-pqsignal.o libpq-events.o pqexpbuffer.o fe-auth.o fe-secure-common.o fe-secure-rustls.o -L../../../src/port -L../../../src/common -lpgcommon_shlib -lpgport_shlib -isysroot /Library/Developer/CommandLineTools/SDKs/MacOSX10.15.sdk -L/usr/local/opt/nss/lib -L/usr/local/opt/nspr/lib -L/Users/kevin/src/github.com/rustls/rustls-ffi/target/lib  -L/usr/local/Cellar/nss/3.69.1/lib -L/usr/local/Cellar/nspr/4.32/lib -lnss3 -lnssutil3 -lsmime3 -lssl3 -lplds4 -lplc4 -lnspr4 -L/usr/local/Cellar/nspr/4.32/lib -lplds4 -lplc4 -lnspr4 -lcrustls -Wl,-dead_strip_dylibs   -lcrustls -lm
Undefined symbols for architecture x86_64:
  "_SecRandomCopyBytes", referenced from:
      __ZN4ring4rand6darwin4fill17hef096156cdbb5e22E in libcrustls.a(ring-059d35c0cff8849f.ring.4rb6t1i9-cgu.13.rcgu.o)
      __ZN4ring2ec7suite_b5ecdsa7signing12EcdsaKeyPair3new17h59971278221fac36E in libcrustls.a(ring-059d35c0cff8849f.ring.4rb6t1i9-cgu.14.rcgu.o)
  "_kSecRandomDefault", referenced from:
      __ZN4ring4rand6darwin4fill17hef096156cdbb5e22E in libcrustls.a(ring-059d35c0cff8849f.ring.4rb6t1i9-cgu.13.rcgu.o)
      __ZN4ring2ec7suite_b5ecdsa7signing12EcdsaKeyPair3new17h59971278221fac36E in libcrustls.a(ring-059d35c0cff8849f.ring.4rb6t1i9-cgu.14.rcgu.o)
ld: symbol(s) not found for architecture x86_64
collect2: error: ld returned 1 exit status
```

### September 3, 2021

Daniel Gustafsson sent a patch via email for the `_exit` issue I was running
into:

```
diff --git a/src/interfaces/libpq/Makefile b/src/interfaces/libpq/Makefile
index 1fe6155e07..3a72369eaa 100644
--- a/src/interfaces/libpq/Makefile
+++ b/src/interfaces/libpq/Makefile
@@ -101,7 +101,7 @@ SHLIB_EXPORTS = exports.txt

 PKG_CONFIG_REQUIRES_PRIVATE = libssl libcrypto

-all: all-lib libpq-refs-stamp
+all: all-lib

 # Shared library stuff
 include $(top_srcdir)/src/Makefile.shlib
```

Must set autoconf to 2.69 precisely.

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

### pgtls_read_pending

?

### pgtls_get_peer_certificate_hash

### pgtls_verify_peer_name_matches_certificate_guts

### pgtls_read

rustls_connection_read_tls
