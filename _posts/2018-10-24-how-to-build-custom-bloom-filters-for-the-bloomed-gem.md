---
layout: post
title: "How to build custom bloom filters for the Bloomed gem"
date: 2018-10-24 00:00:00
categories: "Ruby security bloomed"
show_comments: true
---
In the previous post I showed you how to validate passwords in Rails, rejecting those that are already known to have been part of a major breach. We used an in-memory model based on bloom filters. The major trade off was between memory consumption and accuracy. If you want less false positives, you pay with memory consumption. If you want to include more breached passwords, your pay with memory consumption.

To be specific, here are some examples of how the trade off translates into file sizes:

     21G  pwned-passwords-ordered-by-count.txt

    114M  pwned_top_100000000_one_in_100.msgpk
    171M  pwned_top_100000000_one_in_1000.msgpk
    229M  pwned_top_100000000_one_in_10000.msgpk

     11M  pwned_top_10000000_one_in_100.msgpk
     17M  pwned_top_10000000_one_in_1000.msgpk
     23M  pwned_top_10000000_one_in_10000.msgpk

    1.1M  pwned_top_1000000_one_in_100.msgpk
    1.7M  pwned_top_1000000_one_in_1000.msgpk
    2.3M  pwned_top_1000000_one_in_10000.msgpk

    --- binaries below this line are included in the gem ---

    117K  pwned_top_100000_one_in_100.msgpk
    176K  pwned_top_100000_one_in_1000.msgpk
    234K  pwned_top_100000_one_in_10000.msgpk

     12K  pwned_top_10000_one_in_100.msgpk
     18K  pwned_top_10000_one_in_1000.msgpk
     23K  pwned_top_10000_one_in_10000.msgpk

I wanted to keep the gem distribution size down and left out the larger binaries. You can get decent protection even with one of the built in filters.

If you want a lower rate of false positives, or if you want to include more passwords, you can do so. It takes a while to generate the larger bloom filters. Be prepared to brew a lot of coffee. (Don't even try adding them all unless you have tons of RAM.)

There are two ways to generate the filters you need: With `rake "bloomed:seed[large]"` or using a Ruby repl like [Pry](https://github.com/pry/pry).

## Generating filters using Rake

Bloom filters are binary by nature and I've chosen [MessagePack](https://github.com/msgpack/msgpack) as the format. Therefore, filters end with `.msgpk`.

Note:
> You'll need to install [p7zip](https://www.7-zip.org/download.html) before proceeding. On macos, it's `brew install p7zip`.


The rake task is perfect for generating all of the filters in the list above.
Before seeding bloom filters, the rake task will download the 21GB 7zip file and unpack it in your current dir.

```bash
rake "bloomed:seed[large]"
```
Remember the qoutes when you run rake.

At this point, if you `ls` you should see a 21GB text file and all of the msgpk files listed at the top of this post.

If you want more fine grained control, use Pry:

## Generating filters using Pry

In order for this to work, you need to have the 21 gb password txt file unpacked in your current working dir. See the previous section.

The principle is simple. Whenever you instantiate a filter, it will first look for the binary. If it can't find it, it will look for `pwned-passwords-ordered-by-count.txt` in the current dir and attempt to generate (and save) the msgpk file. If it can't find either, it will raise an exception.


```bash
$> mkdir custom_filters

$> pry -r bloomed

[1] pry(main)> pw=Bloomed::PW.new(top: 20000, \
    false_positive_probability: 0.0001, \
    cache_dir: './custom_filters')

    [...]

[1] pry(main)>
```

Notice the `cache_dir` argument? That's the key to storing and retrieving your filters.

You can package this newly generated (47kb) file, `pwned_top_20000_one_in_10000.msgpk`, along with your app.

When you instantiate your `Bloomed::PW` on your server, remember to use the `cache_dir` argument again, pointing at the place where you copied `pwned_top_20000_one_in_10000.msgpk`.

With this procedure, you can create a custom bloomed filter with the accuracy you want, deploy the filter with your app and make sure that your users do not pick an insecure password.

Cross posted to [medium.com](https://medium.com/@Skovsboll/how-to-build-custom-bloom-filters-for-the-bloomed-gem-bd6c881fd4d3)
