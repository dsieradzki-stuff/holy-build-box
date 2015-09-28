# Tutorial 5: Library variants

[<< Back to Tutorial 4: Tweaking the application's build system](TUTORIAL-4-TWEAKING-APPS.md) | [Tutorial index](README.md#tutorials) | [Skip to Tutorial 6: Introducing additional static libraries >>](TUTORIAL-6-ADDITIONAL-STATIC-LIBS.md)

Holy Build Box provides 3 different variants of its static libraries, each compiled with different compilation flags.

Your application must also be compiled with matching compilation flags. The activation script of each variant sets `CFLAGS`, `CXXFLAGS` and other environment variables, which most applications' build system will automatically pick up.

## Available variants

 * **`nopic`**

   This is the variant that you have worked with so far in these tutorials. This is a good default variant to use. No special compilation options are used.

   Compilation flags: `-O2 -fvisibility=hidden`<br>
   Activation command: `/hbb_nopic/activate-exec`<br>
   Activation source script: `/hbb_nopic/activate`

 * **`pic`**

   This variant is compiled with `-fPIC`. This variant is useful if you plan on linking static libraries into dynamic libraries, e.g. when compiling Ruby native extensions or NPM native extensions.

   Compilation flags: `-O2 -fvisibility=hidden -fPIC`<br>
   Activation command: `/hbb_pic/activate-exec`<br>
   Activation source script: `/hbb_pic/activate`

 * **`deadstrip_hardened_pie`**

   This variant is compiled with security hardening flags and with dead-code elimination flags. This variant is especially suitable for compiling binaries for use in production environments. See [Security hardening binaries](SECURITY-HARDENING-BINARIES.md) for more information.

   **Warning**: the enabled security features are [not compatible with `-O3`](SECURITY-HARDENING-BINARIES.md).

   The dead-code elimination flags allow the compiler to eliminate unused code, which makes your binaries as small as possible.

   Compilation flags: `-O2 -fvisibility=hidden -ffunction-sections -fdata-sections -fstack-protector -fPIE -D_FORTIFY_SOURCE=2 -Wl,--gc-sections -pie -Wl,-z,relro`<br>
   Activation command: `/deadstrip_hardened_pie/activate-exec`<br>
   Activation source script: `/deadstrip_hardened_pie/activate`

## Example: compiling Nginx with the `deadstrip_hardened_pie`

Let's see what happens if we compile Nginx with the `deadstrip_hardened_pie` variant. The Nginx binary that we compiled in [tutorial 4](TUTORIAL-4-TWEAKING-APPS.md) was 2.7 MB after stripping its debugging symbols:

    $ strip --strip-all nginx
    $ ls -lh nginx
    -rwxr-xr-x 1 hongli hongli 2,7M sep 28 13:43 nginx*

It also wasn't compiled with any security hardening flags:

    $ docker run -t -i --rm \
      -v `pwd`/nginx:/exe:ro \
      phusion/holy-build-box-64:latest \
      /hbb_deadstrip_hardened_pie/activate-exec \
      hardening-check -b /exe
    ...
    Position Independent Executable: no, normal executable!
    Stack protected: no, not found!
    Fortify Source functions: no, only unprotected functions found!
    Read-only relocations: no, not found!
    Immediate binding: no, not found! (ignored)

Modify the compilation script to load `/hbb_deadstrip_hardened_pie/activate` instead of `/hbb_nopic/activate`. Change this line...

    source /hbb_nopic/activate

...to:

    source /hbb_deadstrip_hardened_pie/activate

Then invoke the compilation script:

    docker run -t -i --rm \
      -v `pwd`:/io \
      phusion/holy-build-box-64:latest \
      bash /io/compile.sh

Let's take a look at the Nginx binary now:

    $ strip --strip-all nginx
    $ ls -lh nginx
    -rwxr-xr-x 1 hongli hongli 2,6M sep 28 13:44 nginx*

The Nginx binary is now 2.6 MB. We saved about 100 KB.

We can also see that the security hardening flag are enabled inside the binary:

    $ docker run -t -i --rm \
      -v `pwd`/nginx:/exe:ro \
      phusion/holy-build-box-64:latest \
      /hbb_deadstrip_hardened_pie/activate-exec \
      hardening-check -b /exe
    ...
    Position Independent Executable: yes
    Stack protected: yes
    Fortify Source functions: yes (some protected functions found)
    Read-only relocations: yes
    Immediate binding: no, not found! (ignored)

**Tip**: when using the `deadstrip_hardened_pie`, we recommend that you call `hardening-check` from your compilation script after compilation is finished.

## The entire compilation script

~~~bash
#!/bin/bash
set -e

# Activate Holy Build Box environment.
source /hbb_nopic/activate

set -x

# Extract and enter source
tar xzf /io/nginx-1.8.0.tar.gz
cd nginx-1.8.0

# Compile
sed -i 's|-lssl -lcrypto|-lssl -lcrypto -lz -ldl|' auto/lib/openssl/conf
./configure --without-http_rewrite_module --with-http_ssl_module --with-ld-opt="$LDFLAGS"
make
make install

# Verify result
hardening-check -b /usr/local/nginx/sbin/nginx

# Copy result to host
cp /usr/local/nginx/sbin/nginx /io/
~~~

## Conclusion

You have now learned about the different library variants. Next up, you will learn what to do if your application has additional dependencies that are not included in Holy Build Box.

[Tutorial 6: Introducing additional static libraries >>](TUTORIAL-6-ADDITIONAL-STATIC-LIBS.md)