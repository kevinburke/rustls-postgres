# Patchsets

Daniel Gustafsson's work is attached as a series of patchsets to the email, this
directory is designed to check in those changes so that we can follow them over
time.

An example command you can use to apply these patchsets is:

```bash
cd /path/to/postgres/postgres
for patch in ../../kevinburke/postgres-rustls/patchsets/2021-08-10-gustafsson-mailing-list/*.patch; do
    git am $patch
done
```
