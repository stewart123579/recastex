# Recast Annex

Take files and podcasts captured by git-annex and re-podcast them.


##### What is `git-annex`?
    git-annex allows managing files with git, without checking the file contents into git.

Thus, we can have *links* to postcast shows locally without actually having the content.  So your laptop can track *all* your podcasts without storing all the episodes locally.


##### Why `recastex`?

With recastex (RECAST annEX) you can now re-podcast the shows you have locally (to, say, your phone).  This reduces network usage (brilliant for travelling when network costs are expensive) and improves privacy (see below).


Recasing isn't limited to podcasts.  recastex casts all *locally available* media.

Have at it.  Listen to whatever you want.  Whenever you want.  On whichever device you want.



## What do the arguments do?


### `recastex getinfo`

`recastex getinfo` parses your `feeds` file, downloads the feeds, gets description, title, image, etc. and stores the information in `feed_details.yml` and the images (renamed to be the same as the feed).

This script **accesses the internet**, but only needs to be run when you add a new feed.

**Note:** By default you will need to use the `--talk-to-the-internet` flag to get this script to actually call out to the internet.  This is easily changed by setting `PRIVACY = False` at the top of the script.

*For the privacy minded, you can run this on your internet exposed host and add all files into git/git-annex.  Then you can fetch the data to your other hosts via git-annex over private VPN/SSH tunnels.*


### `recastex generate`

`recastex generate` collects the metadata on postcast shows that are present on the local machine.  It then generates RSS and OPML feeds for podcast managers to access.

This script **does not access the internet**, it is entirely local.

By default metadata regeneration occurs (it needs git-annex).  If you want to use a cached version use the `--noregenmetadata` flag.



## How to run


### *Step 1* - Setting up your `git-annex` as a pod-catcher (requires external network)

It's easist (and most up-to-date) if you read the git-annex tips page on [downloading podcasts](https://git-annex.branchable.com/tips/downloading_podcasts/), however this tool is best able to help is you follow **storing the podcast list in git**.  That is:

- create a list of podcast urls and check into git.  Just make a file named feeds and add one podcast url per line.  I use lines starting with a `#` for comments
- Run git-annex on all the feeds:
    ``` sh
    grep -v '^#' feeds| xargs git-annex importfeed --relaxed
    ```

For example, my `feeds` file looks like

    # NPR - Wait Wait... Don't Tell Me!
    https://www.npr.org/rss/podcast.php?id=35
    # NPR - Car Talk
    https://www.npr.org/rss/podcast.php?id=510208


### *Step 1.a* - Get some content

This is a git-annex step.  Try reading the fine manual.  Effectively you want something like:

```sh
git annex get Wait_Wait..._Don_t_Tell_Me_
```


### *Step 2* - Generating local copy of feed info (requires external network)

Simply run `recastex getinfo` from the directory in which you have your `feeds` file.

It will create a directory `recastex` and populate it with:

1. `feed_details.yml` - A YaML file with info about your feeds; and
2. Images for the different feeds.

It is probably a good idea to commit these files to git annex too:
``` sh
git add recastex/feed_details.yml
git annex add recastex/*.png recastex/*.jpg
git commit -m "Added feed details used by recastex"
```


### *Step 3* - Generating podcasts feeds (does not require external network)

Running `recastex generate` will generate
- All the individual feeds that have *local* files (so `recastex/Wait_Wait..._Don_t_Tell_Me_.xml` for example)
- An OPML file for all your feeds: `recastex/all.opml`
- `index.html` - a bare-bones index file linking to your feeds and OPML file


### *Step 4* - Run webserver

I use docker and the `kyma/docker-nginx` container for simplicity.

```sh 
THISHOST=$(ifconfig | grep 192.168 | awk '{print $2}') \
	&& echo "My IP is : ${THISHOST}" \
    && docker run -d --rm --name webhost \
    	-p ${THISHOST}:8080:80 \
        -p ${THISHOST}:443:443 \
        -v $(pwd):/var/www:ro \
        kyma/docker-nginx
```

**Note:** The URL here `${THISHOST}:8080` should point to the same location as the URL used in recastex (`recastex --url ...`).  I say *should* as Docker requires an IP address, whilst `recastex` can take an IP address (`192.168.0.3`) or a domain name (`mylappy.local` / `my.host.com`), but the addresses should be for the same machine (i.e. `mylappy.local == 192.168.0.3`).


### *Step 5* - Connect your podcast player
Open up `http://192.168.x.y:8080` in your browser to see the various feeds that are available.

I use [AntennaPod](http://antennapod.org/) on Android and it is as simple as opening `http://192.168.x.y:8080/recastex/all.opml` in my browser, selecting AntennaPod to open the OPML file and all my feeds are available.



## Requires

My install is:
- python 3.6.4
- PyYAML 3.12
- feedgen 0.6.1
- feedparser 5.2.1



## Benefits


### Non-podcast media

You have some files you want to listen to (like Cory Doctorow's [Car Wars](http://archive.org/download/CarWars/Car%20Wars.mp3))? 

Just add them to your podcast repository.  Wherever.  It doesn't matter.

Recastex casts all *locally available* media.  If it can't associate a feed it appears in `Unknown.xml`.

Have at it.  Listen to whatever you want.  Whenever you want.  On whichever device you want.


### Privacy

As git-annex can be configured (like git) to be a distributed network you can have all your internet facing activities occur on a single machine (like a VPS, EC2 instance, etc.) and then all your recasting done elsewhere.  That way the IP address connecting (and being logged) to the podcast sources will not be your personal/home network, but a machine your choose.

The privacy comes from using the `git-annex get` functionality and **[disabling the web remote](https://git-annex.branchable.com/tips/disabling_a_special_remote/)**:

``` sh
git config remote.web.annex-ignore false
```

In this case, the *local* git-annex will only call out to known annexes and not the web for content.

Then any re-casting will be privacy respecting (assuming no leaks from the data files, i.e. JPG/PNG/MP3).


### Network usage

(At least here in Australia) Mobile data is expensive, network connections are often slow and unreliable.

Using recastex with git-annex you download the feed information once over the broader network and then it is available (and cast) locally.



## To-Do

1. Have an 'update' option for get_feed_info.py - i.e. only get info for new feeds, not everything.
2. Repackage for PyPI?
3. Testing!
3. Better README, how-to, etc.
4. Update to use SSL



## Notes / Limitations

1. If the feed icon is not at least 1400x1400 pixels then it may not display - this is an iTunes requirement.



## Licence

    Copyright (C) 2018 Stewart V. Wright <stewart@vifortech.com>
    Supported by Vifortech Solutions.

recastex is [Free Software](https://www.gnu.org/philosophy/free-sw.en.html), written in [Python](https://www.python.org/). You can [contribute](https://github.com/stewart123579/recastex)!

recastex is licensed under the [GPL](https://github.com/stewart123579/recastex/blob/master/LICENSE), version 3 or higher.

What that means is this program comes with ABSOLUTELY NO WARRANTY.  If it breaks, you get to keep all the parts.
