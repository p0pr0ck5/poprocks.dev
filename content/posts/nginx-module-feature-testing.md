---
title: "Nginx Module Feature Testing"
date: 2018-06-02T12:40:33-08:00
---

Until (somewhat) recently, Nginx development was somewhat of an adventurous journey. Official documentation was largely nonexistent; Evan Miller’s decade-old guide was the often-referenced canonical source of material. Publication of an authoritative development guide came only a few years ago, significantly lowering the bar to entry for third party developers. It’s an excellent source of material, but it doesn’t cover every aspect of authoring and extending Nginx, meaning that complex or uncommon features still require a bit of blog browsing, copypasta, and diving into the Nginx source to figure out what’s going on.

Module feature testing is one of these aspects. Writing simple Nginx modules is straightforward, but discussion of module config files is often glossed over or ignored entirely. The canonical development guide does a reasonable job of touching on some of the functionality and definitions available to config files, but it’s short on examples, and it ignores a crucial aspect of developing complex modules or integrations with third party libraries: feature testing.


Module feature testing is the process by which the Nginx configure script examines the build environment to determine if a given feature can successfully be integrated into Nginx. Unlike traditional autoconf tooling, the Nginx configure script is comprised of a number of hand-written shell tooling, the majority of which features tests various OS facilities (compiler data, libc environment, TCP stack), as well as required library features (zlib, OpenSSL, PCRE, etc).

Feature testing is performed by the auto/feature script. In short, this script attempts to build (and possibly run) a small executable based on the definition of several shell variables defined prior to the invocation of the feature script. The test executable is defined as follows:

```bash
if test -n "$ngx_feature_path"; then
    for ngx_temp in $ngx_feature_path; do
        ngx_feature_inc_path="$ngx_feature_inc_path -I $ngx_temp"
    done
fi

cat << END > $NGX_AUTOTEST.c

#include <sys/types.h>
$NGX_INCLUDE_UNISTD_H
$ngx_feature_incs

int main(void) {
    $ngx_feature_test;
    return 0;
}

END
```

There are a few shell variables used to build this out:

* `$ngx_feature_path` is a whitespace separated list of library header path locations to pass to the testing compiler
* `$ngx_feature_incs` typically is defined by header includes or global definitions needed for the executable
* `$ngx_feature_test` fills out the auto test main function. This value doesn’t have to be defined, depending on the nature of the feature test

`auto/feature` invokes the compiler based on the above definitions, walking through a standard preprocessing-compiling-assembling-linking lifecycle. If the executable does not successfully compile, the feature is determined not to be found. Note that a missing feature doesn’t mean the configure step must fail, particularly for core or OS-specific feature tests.

If the test executable is successfully compiled, the feature availability then depends on the value of `$ngx_feature_run`. Generally this value is either “yes” or “no”; a few other options (“bug” and “value”) can be passed to modify the test behavior, but I haven’t seen these used in practice. When the value is “yes”, the test executable will be run; auto/feature then treats a non-zero return value as a failure. Forcing the test executable to run is useful when validating complex or version-dependent behavior that cannot be determined by a successful build lifecycle.

Regardless of the behavior defined by `$ngx_feature_run`, the end result of auto/feature is the definition of `$ngx_found`: “yes” or “no”. From here, feature testers can determine the behavior of Nginx as needed, either aborting, passing a warning, or dressing in drag and doing the hula.

Let’s take a look at a basic feature test built into the core Nginx project: a test for the `sched_setaffinity` function executed on Unix-like environments:

```bash
ngx_feature="sched_setaffinity()"
ngx_feature_name="NGX_HAVE_SCHED_SETAFFINITY"
ngx_feature_run=no
ngx_feature_incs="#include <sched.h>"
ngx_feature_path=
ngx_feature_libs=
ngx_feature_test="cpu_set_t mask;
                  CPU_ZERO(&mask);
                  sched_setaffinity(0, sizeof(cpu_set_t), &mask)"
. auto/feature
```

`$ngx_feature_run` is defined as no, so configure will simply attempt to compile an executable. Note also the definition of `$ngx_feature_name`, which is then passed by the configure script and made available at compile time; Nginx can use this to alter the behavior of various core/OS-specific functionality as needed.

Let’s now consider a slightly more complicated feature test from the Nginx libmodsecurity connector:

```bash
# If $MODSECURITY_INC is specified, lets use it. Otherwise lets try
# the default paths
#
if [ -n "$MODSECURITY_INC" -o -n "$MODSECURITY_LIB" ]; then
    # explicitly set ModSecurity lib path
    ngx_feature="ModSecurity library in \"$MODSECURITY_LIB\" and \"$MODSECURITY_INC\" (specified by the MODSECURITY_LIB and MODSECURITY_INC env)"
    ngx_feature_path="$MODSECURITY_INC"
    ngx_modsecurity_opt_I="-I$MODSECURITY_INC"
    ngx_modsecurity_opt_L="-L$MODSECURITY_LIB $YAJL_EXTRA"

    if [ $NGX_RPATH = YES ]; then
        ngx_feature_libs="-R$MODSECURITY_LIB -L$MODSECURITY_LIB -lmodsecurity $YAJL_EXTRA"
    elif [ "$NGX_IGNORE_RPATH" != "YES" -a $NGX_SYSTEM = "Linux" ]; then
        ngx_feature_libs="-Wl,-rpath,$MODSECURITY_LIB -L$MODSECURITY_LIB -lmodsecurity $YAJL_EXTRA"
    else
        ngx_feature_libs="-L$MODSECURITY_LIB -lmodsecurity $YAJL_EXTRA"
    fi

    . auto/feature

    if [ $ngx_found = no ]; then
        cat << END
        $0: error: ngx_http_modsecurity_module requires the ModSecurity library and MODSECURITY_LIB is defined as "$MODSECURITY_LIB" and MODSECURITY_INC (path for modsecurity.h) "$MODSECURITY_INC", but we cannot find ModSecurity there.
END
        exit 1
    fi
else
    # auto-discovery
    ngx_feature="ModSecurity library"
    ngx_feature_libs="-lmodsecurity"

    . auto/feature

    if [ $ngx_found = no ]; then
        ngx_feature="ModSecurity library in /usr/local/modsecurity"
        ngx_feature_path="/usr/local/modsecurity/include"
        if [ $NGX_RPATH = YES ]; then
            ngx_feature_libs="-R/usr/local/modsecurity/lib -L/usr/local/modsecurity/lib -lmodsecurity"
        elif [ "$NGX_IGNORE_RPATH" != "YES" -a $NGX_SYSTEM = "Linux" ]; then
            ngx_feature_libs="-Wl,-rpath,/usr/local/modsecurity/lib -L/usr/local/modsecurity/lib -lmodsecurity"
        else
            ngx_feature_libs="-L/usr/local/modsecurity/lib -lmodsecurity"
        fi

        . auto/feature

    fi
fi



if [ $ngx_found = no ]; then
 cat << END
 $0: error: ngx_http_modsecurity_module requires the ModSecurity library.
END
 exit 1
fi
```

This module config script optionally takes advantage of predefined environmental variables to search for the library, and attempts multiple path searches if a custom include/library path is undefined. This is one of the features of config scripts that I enjoy. Being able to define the feature (or module) configuration using a scripting language, instead of being restricted to a DSL, allows for a lot of flexibility and creativity on the part of the module author, lowering the barrier to entry to consumers.

Feature testing and module configuration is not a particularly sexy or exciting part of Nginx extension and module development, but it’s necessary in complex integration environments, and developers are given a lot of freedom. It’s a blast to play with. I would love to see feature testing treated as a first-class citizen in Nginx documentation and code maintenance.
