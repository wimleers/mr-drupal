# What?
This project proposes a novel, simpler way of managing Drupal sites.

It relies on only one command-line tool: `mr` ("a Multiple Repository management tool", <http://joeyh.name/code/mr/>), plus a Drupal plug-in for that tool.


# Why?

If you run Drupal sites, you need some way to manage them; some way to keep them up-to-date. As of Drupal 7, there's a built-in update manager, but it doesn't use a VCS.<br>Most likely, you want use a VCS to manage your Drupal site. You may be downloading tarballs and checking them into your VCS, or maybe you're using git submodules or even `git-subtree`.

In all of the above cases, there's a whole lot of process, a lot of steps, a lot to learn, and a lot of tricky things you have to think about *each time you want to update something*.

That is fine for big, commercial, team-backed sites. But it's a pain to do for smaller sites, that maybe only get updated once every few months. When you update rarely, it becomes very easy to forget about one of those tricky things. And then, for fear of doing X or Y wrong, which might break your site, causes you to update your site even less often.

It's *that niche* that this approach to Drupal site management attempts to simplify.


# How?

`mr` is a small, stable shell script (written in Perl, unfortunately) to help manage multiple (VCS) repositories simultaneously. By using the Drupal plug-in provided by this project, you can leverage a *subset* of `mr`'s functionality to simplify Drupal site management.

**The Drupal plugin offers you a simple, declarative syntax to automatically clone and update git repositories for each Drupal module you use from `git.drupal.org`.**

There's almost nothing to learn (except for to play around with it). Hence, there's almost nothing to forget. There is a single configuration file that is (mostly) declarative, so if you look at it again in 2, 6 or 12 months, it will still make sense.

**Essentially, you only need to know two things:**

1. the desired state is declared in the `.mrconfig` file
2. get to the desired state by running `mr update`

Note that the tool/approach described does not cover running database updates etc. (though it is possible to integrate that as well). I prefer to do this manually on smallish sites.


## Getting started

Steps to be repeated on each machine:

1. Install `mr`. (Package in Debian, unofficial RPM and on OS X via `brew`.)
2. Install the Drupal plug-in for `mr`: copy the `drupal` file in this repository into `/usr/share/mr/`.


## Creating a config file for your site

Steps to be repeated for each Drupal site:

1. Create a `.mrconfig` file, like this:
    ```ini
    # Use the Drupal plug-in for `mr`.
    [DEFAULT]
    include = cat /usr/share/mr/drupal
    
    # Drupal itself.
    [www]
    project = drupal
    version = 7.19
    # Set the proper file system permissions on all Drupal files. The owner is
    # set to the current user, the group is set to www-data.
    # See http://drupal.org/node/244924.
    fixups = drupal_set_permissions `whoami` 'www-data'

    # A branch of a Drupal module.
    [www/sites/all/modules/cdn]
    project = cdn
    version = 7.x-2.x

    # A tag of a Drupal module.
    [www/sites/all/modules/comment_notify]
    project = comment_notify
    version = 7.x-1.1

    # Custom themes live in a non-git.drupal.org repository. `mr` is smart enough
    # to automatically know this is a git repository, and knows how to update it.
    [www/sites/wimleers.com/themes]
    checkout = git clone git://github.com/wimleers/wimleers.com-themes.git themes
    ```
As you can see, it's easy to mix in custom modules/repositories.
2. Run `mr update`. Your Drupal site will be built based on your `.mrconfig` file, relative to where the `.mrconfig` file lives. Speed it up by telling it to work in parallel: `mr -j 10 update` would do up to 10 updates in parallel.
3. Optionally check in the `.mrconfig` file into a VCS, so you can roll back to previous versions. Preferable, but not essential.

We leveraged one "advanced" feature of `mr`: the ability to do "fixups" (after each checkout/update). In this case, to guarantee correct permissions.


## To update your site

1. Modify the `.mrconfig` file: change version numbers, add modules. Remove a module by adding a line like this: `deleted = true`.

2. Run `mr update`. That's it!

The only exception: those that are marked as deleted. `mr` does not actually delete them for you; assuming you've marked the `devel` and `drupad` modules as deleted and the `devel` module no longer exists, it'll output something like this:
```shell
$ mr -j 20 --stats update
mr update: /htdocs/wimleers.com/www/sites/all/modules/cdn
Updating 'cdn' to version 7.x-2.5 (from 7.x-2.3)...
Done.

mr error: /htdocs/wimleers.com/www/sites/all/modules/drupad/ should be deleted yet still exists

mr update: /htdocs/wimleers.com/www/sites/all/modules/cdn
Already at tag 7.x-2.5.

[…]

mr update: finished (29 ok; 1 failed; 1 skipped)
mr update: (skipped: /htdocs/wimleers.com/www/sites/all/modules/devel/)
mr update: (failed: /htdocs/wimleers.com/www/sites/all/modules/drupad/)
```
The `devel` module is skipped (no error!) because it's marked as deleted and doesn't exist. The `drupad` module is marked with an error message


# Advanced: environment-specific (Dev vs. Prod) actions

If the above doesn't look like it would cut it for you because you need to set up things differently for your local development setup and your production server, then you're right (i.e. [DTAP](http://en.wikipedia.org/wiki/Development,_testing,_acceptance_and_production)).

To easily deal with that too, this `drupal` extension for `mr` provides three functions:

1. `whoami()` — a simple wrapper around the `whoami` command.
2. `hostname()` — a simple wrapper around the `hostname` command.
3. `on()` — uses the two functions above to be able to write something like
   `on 'wim@wimleers.local'`, which evaluates to true if those are indeed the
   current user and host.

You can then upgrade the above example of

```ini
[DEFAULT]
include = cat /usr/share/mr/drupal

[www]
project = drupal
version = 7.19
fixups = drupal_set_permissions `whoami` 'www-data'
```

to something like this:

```ini
[DEFAULT]
include = cat /usr/share/mr/drupal
lib =
  get_dtap() {
    if on 'wim@wimleers.local'; then
      echo 'development'
    else
      echo 'production'
    fi
  }
  drupal_fs_group() {
    if [ "$(get_dtap)" = 'development' ]; then
      echo 'staff' # Mac OS X
    else
      echo 'www-data' # Linux
    fi
  }

[www]
project = drupal
version = 7.19
fixups = drupal_set_permissions `whoami` `drupal_fs_group`
```

But really, you can do whatever you want: anything that can be used in a shell script can also be used here.




# Conclusion

One file (`.mrconfig`) and one command (`mr update`) together cover 90% of the smallish Drupal site management needs.

Easy to understand, very few dependencies, extremely customizable.

Simple for simple deployments, complex for complex deployments. Instead of *always* complex.