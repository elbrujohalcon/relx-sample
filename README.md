# relx-sample
A repo to demonstrate a relx issue

## The Problem
In this repo you can see that we have a very minimal OTP application.
The only particular thing is that we have an `extra_folder` that we want to include in the release.
Since relx overlays don't accept wildcards, the only way to add that folder (and its subfolders and files) into the release is to use the following overlay...

```erlang
{overlay, [{copy, "your/folder/", "parent/folder/in/the/release"}]}.
```

With that `copy` overlay, `your/folder` will be included in the release under `parent/folder/in/the/release` (i.e. it will end up being `parent/folder/in/the/release/folder`).

In our case, we needed `extra_folder` in the root of our release, so we added...

```erlang
{overlay, [{copy, "extra_folder/", "."}]}.
```

Now, if we try to generate the release...

```bash
$ rebar3 release
===> Verifying dependencies...
===> Compiling my
===> Starting relx build process ...
===> Resolving OTP Applications from directories:
          /elbrujohalcon/relx-sample/_build/default/lib
          /.kerl/19.3/lib
===> Resolved my-1.0.0
===> Including Erts from /.kerl/19.3
===> release successfully created!
$ ls _build/default/rel/my/
bin              erts-8.3        extra_folder    lib             releases
```

**Perfect!** The folder is there.

What if we try to generate a tar file?

```bash
$ rebar3 tar
===> Verifying dependencies...
===> Compiling my
===> Starting relx build process ...
===> Resolving OTP Applications from directories:
          /elbrujohalcon/relx-sample/_build/default/lib
          /.kerl/19.3/lib
          /elbrujohalcon/relx-sample/_build/default/rel
===> Resolved my-1.0.0
===> Including Erts from /.kerl/19.3
===> release successfully created!
===> Starting relx build process ...
===> Resolving OTP Applications from directories:
          /elbrujohalcon/relx-sample/_build/default/lib
          /.kerl/19.3/lib
          /elbrujohalcon/relx-sample/_build/default/rel
===> Resolved my-1.0.0
*WARNING* Missing application sasl. Can not upgrade with this release
===> tarball /elbrujohalcon/relx-sample/_build/default/rel/my/my-1.0.0.tar.gz successfully created!
$ tar tf _build/default/rel/my/my-1.0.0.tar.gz
releases/my.rel
releases/1.0.0/start.boot
releases/1.0.0/my.rel
releases/1.0.0/sys.config
releases/start_erl.data
releases/RELEASES
bin/no_dot_erlang.boot
bin/my-1.0.0
bin/start_clean.boot
bin/my
lib/stdlib-3.3/...
...
lib/my-1.0.0/ebin/my.app
lib/my-1.0.0/src/my.app.src
lib/kernel-5.2/...
...
releases/1.0.0/vm.args
./releases/RELEASES
./releases/1.0.0/no_dot_erlang.boot
./releases/1.0.0/vm.args
./releases/1.0.0/my.rel
./releases/1.0.0/sys.config
./releases/1.0.0/my.script
./releases/1.0.0/start_clean.boot
./releases/1.0.0/my.boot
./releases/start_erl.data
./bin/no_dot_erlang.boot
./bin/my-1.0.0
./bin/start_clean.boot
./bin/my
./my-1.0.0.tar.gz
./lib/my-1.0.0/ebin/my.app
./lib/my-1.0.0/src/my.app.src
./extra_folder/some_script
./extra_folder/sub_folder/another_file
```

As you can see `extra_folder` is there, with all its subfolders and also `my-1.0.0.tar.gz` and a second version of the `releases` folder...

![wait what?](https://vignette.wikia.nocookie.net/deathbattle/images/0/0b/Wait-What-Meme-11.jpg/revision/latest?cb=20161005180110)

What is that `./my-1.0.0.tar.gz` doing inside my `my-1.0.0.tar.gz` file?

In case you are wondering, that file is not empty but it's also not a valid tar.gz file.

## The Diagnosis
After running `rebar3 shell` and using [`redbug`](https://hex.pm/packages/redbug) to trace what was happening when I run `r3:tar()`, I found this call to `erl_tar:create/3`:

```erlang
erl_tar:create(
  "/elbrujohalcon/relx-sample/_build/default/rel/my/my-1.0.0.tar.gz",
  [ { "releases"
    , "/var/folders/07/d0j3z4p56ts3w_vx54rs7bx00000gn/T/.tmp_dir139302618984/releases"
    }
  , { "releases/start_erl.data"
    , "/elbrujohalcon/relx-sample/_build/default/rel/my/releases/start_erl.data"
    }
  , { "releases/RELEASES"
    , "/elbrujohalcon/relx-sample/_build/default/rel/my/releases/RELEASES"
    }
  , { "bin"
    , "/elbrujohalcon/relx-sample/_build/default/rel/my/bin"
    }
  , { "lib"
    , "/var/folders/07/d0j3z4p56ts3w_vx54rs7bx00000gn/T/.tmp_dir139302618984/lib"
    }
  , { "releases/1.0.0/vm.args"
    , "/elbrujohalcon/relx-sample/_build/default/rel/my/releases/1.0.0/vm.args"
    }
  , { "."
    , "/elbrujohalcon/relx-sample/_build/default/rel/my/."
    }
  ], [dereference,compressed])
```

That last tuple is the culprit: `{ ".", "/elbrujohalcon/relx-sample/_build/default/rel/my/."}`. Since the compressed file is in `/elbrujohalcon/relx-sample/_build/default/rel/my/my-1.0.0.tar.gz`, and it's including `.` in it... well, it's including itself in the generated result. An intermediate version of itself, that is.

That list of files to compress comes from [this piece of relx code](https://github.com/erlware/relx/blob/9f9989167b04c1335c23fafe9945f490abbff799/src/rlx_prv_archive.erl#L111)

## The Hack
If you check that part of the code in relx, you'll find [another interesting bit](https://github.com/erlware/relx/blob/9f9989167b04c1335c23fafe9945f490abbff799/src/rlx_prv_archive.erl#L167).
So, to _workaround_ our issue, we can replace our overlay with...

```erlang
{overlay, [{copy, "extra_folder/", "bin/.."}]}.
```

But for that to work, we can't run just `rebar3 tar`, we need `rebar3 do release, tar`.

![don't try this at home](https://www.reactiongifs.us/wp-content/uploads/2014/06/dont_try_at_home_futurama.gif)
