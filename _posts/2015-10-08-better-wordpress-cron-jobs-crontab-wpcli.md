---
layout: post
title:  "Better Wordpress Cron Jobs with crontab and WP CLI"
date:   2015-10-08
tags: wordpress, cron, wpcli
published: true
---

WP Cron jobs can be powerful tools, allowing you to schedule tasks that need to happen automatically. Unfortunately, by default, this only runs on a page visit. 

That means two things for you:

- Jobs only run when there's visitors actively on the site when its supposed to run
- A log running job will hold up the page render for the unlucky visitors that trigger it

### So, how do you do that better?

Crontab is a great solution for running cron jobs, its what your server does for its own jobs. But getting it to execute tasks can be a little tricky. Theres workaround requiring your server to call `curl http://yoursite.com/wp-cron.php`, but that relies on curl and that domain is resolvable from the server.

### WP-CLI to the rescue

WP CLI has the ability to run cron jobs itself. It does this intelligently, obeying the set schedule even when its run outside the task's window. So that means we can safely run that command on a minute interval.

This command runs a cron job, but you notice, it needs the hook name and can only run one at a time

`wp cron event run cron_job_hook_name`

This command lists all cron jobs available

`wp cron event list`

So, we take that and only get the hook, no formatting.

`wp cron event list --fields=hook --format=ids`

Then we're going to wrap that in a bash loop so that each hook that gets returned runs the cron job.

`for i in $(wp cron event list --fields=hook --format=ids); do wp cron event run $i; done`

That'll run them all! But since this is a cron job, it'll run outside our normal bash environment. So, we need to specify where the site is with: `--path=/full/path/to/site`

`for i in $(wp cron event list --fields=hook --format=ids --path=/full/path/to/site); do wp cron event run $i --path=/full/path/to/site; done`

Notice how we have to do that in the list command _and_ the run command.

Sometimes, depending on how you set up WP-CLI, you'll need to specify it's location too, like `/usr/local/bin/`

`for i in $(/usr/local/bin/wp cron event list --fields=hook --format=ids --path=/full/path/to/site); do /usr/local/bin/wp cron event run $i --path=/full/path/to/site; done`

Awesome, so this should run great, lets put it in a cron job.

That runs all WP cron jobs, every 5 minutes (*/12 is 60 minutes/12 = 5 minutes)

`*/12 * * * * for i in $(/usr/local/bin/wp cron event list --fields=hook --format=ids --path=/full/path/to/site); do /usr/local/bin/wp cron event run $i --quiet --path=/full/path/to/site; done`

Note I added `--quiet` to the `run` command so that I don't get an email every time this runs.

### root problems
Sometimes you'll have to run this as a root user, which WP CLI will warn you about. If you're comfortable with running this under root, WP-CLI has `--allow-root` as well

`*/12 * * * * for i in $(/usr/local/bin/wp cron event list --fields=hook --format=ids --path=/full/path/to/site --allow-root); do /usr/local/bin/wp cron event run $i --quiet --path=/full/path/to/site --allow-root; done`

Again, note that you must `--allow-root` on both commands.

There ya go, now you can disable WP's cron runner by adding this to your wp-config.php

`define( 'DISABLE_WP_CRON', true );`

Hope that helps!
