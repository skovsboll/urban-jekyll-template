---
layout: post
title: "Stop your users from choosing a compromised password - A Rails example"
date: 2018-10-01 01:36:23
categories: "Ruby security bloomed"
---
You run a website. Users sign up. They do so by entering their email address and a password. Well, not any password. A subset will enter `password123`, the same password they use on every website. Even though you bcrypt the password before storing it in your db, it's still easy for an attacker to lookup the credentials in one of the many lists of credentials previously breached.

Let's say the user with the insecure password is an administrator for other users in your website. That's when you realize their low security is **your** low security.

## What can you do?

Troy Hunt has released V2 of his list of [more than 500 million pwned (breached) passwords](https://haveibeenpwned.com/Passwords). This is a brilliant resource, one that's already making many sites more secure. To take advantage of them in for your own site for password validation, you have three options:

You can download the full 22GB file and put all 500M SHAs, or a subset of them, in a database table. They will take up at least the space that the original list takes. With an index, search times can be ... ok. But 22GB is still a price to pay.

Second option is to use [Troy Hunt's API](https://haveibeenpwned.com/API/v2#PwnedPasswords). Let the service worry about disk size and indices. This makes sense in many situations. There may be cases, however, where you don't want to take on a dependency on an external service.

## Enter the bloom filter

The third option is to use a [Bloom Filter](https://en.wikipedia.org/wiki/Bloom_filter). A bloom filter is a data structure that trades accuracy for lower memory consumption. The deal is: For the same amount of space, a bloom filter lets you store many times more the data than a normal index.

You pay for this benefit with a probability of _false positives_. Once in a while, the bloom filter will think it has seen a password before while in fact it has not. So a percentage of your users will try a password that is falsely rejected. On the other hand a bloom filter stores 100.000 breached passwords with a false positive probability of 0.001, taking only **194 kb of memory** to do so. That's few enough bytes that you may decide to keep in memory. You can configure a [bloom filter like this excellent one](https://github.com/mceachen/bloomer) with your specific trade off: Use more bytes to store more passwords and have fewer false positives? It's your call.

To get you going quickly, all of this has been packaged into a gem called [Bloomed](https://github.com/skovsboll/bloomed). It includes a few options to choose from: Top 10.000 or 100.000 passwords with a false positive probability of either 0.01, 0.001 or 0.0001. To keep the gem distribution of modest size, the larges pre-packaged filter weighs 253 kb.

If you want to add more options, there's a rake task that will download the full 22GB and pre-seed a bloom filter with your desired trade off between size and accuracy. For instance, a filter including the top 10M passwords with 1-in-1000 average false positives takes up just 17.5 Mb of disk and memory.

## Step by step implementation in Rails

Start by adding a couple of gems to your `Gemfile`:

```ruby

gem 'bcrypt'
gem 'bloomed'

```

The `User` model should hold a username and a password digest (a bcrypted hash).

```ruby

rails g model User username:string password_digest:string

```

Next, you'll need a custom validator that will invoke the Bloomed gem.

```ruby

class UncompromisedValidator < ActiveModel::EachValidator
  def validate_each(record, attribute, value)
    if Rails.application.config.bloomed.pwned? value
      record.errors.add(attribute, "has been compromised in an earlier password breach.
        To protect your data, we don't allow using passwords that might already be in
        the hands of people with bad intentions. This is a good time to choose a
        password that you have never used before.")
    end
  end
end

```

Inside your `config/application.rb` you'll need to prepare the shared instance of a `Bloomed::PW` class. It needs to be shared because it fills some space in memory with the bloom filter that the validator uses for lookups. Even if the memory size is significantly reduced with a bloom filter, loading does take some time. You only want to load this data once.

```ruby

config.bloomed = Bloomed::PW.new

```

Finally, modify `models/user.rb` to validate the password using your custom validator.

```ruby

class User < ApplicationRecord
  has_secure_password
  validates :password, uncompromised: true
end

```

That's all the code you need to check every password entry against the top 100.000 breached passwords with a false positive probability of 0.001, taking only 194 kb of memory to do so.

Let's try it out. Start a `rails console`.

```ruby

irb(main):001:0> u=User.create username: 'safe-user', password: 'correcthorsebatterystaple'
   (0.1ms)  begin transaction
  User Create (0.5ms)  INSERT INTO "users" ("username", "password_digest",
    "created_at", "updated_at") VALUES (?, ?, ?, ?)  [["username",
      "safe-user"], ["password_digest",
      "$2a$10$wIS7wVyjpXoVcAy6DTKP3O52.JReSEc.sDlatvzpE.gVISNvO7vDa"],
      ["created_at", "2018-09-29 21:29:09.993106"],
      ["updated_at", "2018-09-29 21:29:09.993106"]]
   (0.9ms)  commit transaction
=> <User id: 2, username: "safe-user", password_digest:
   "$2a$10$wIS7wVyjpXoVcAy6DTKP3O52.JReSEc.sDlatvzpE.g...",
   created_at: "2018-09-29 21:29:09", updated_at: "2018-09-29 21:29:09">

```

So that went well using `correcthorsebatterystaple`.
Let's try again, this time using `password123`.

```ruby

irb(main):002:0> u=User.create username: 'unsafe-user', password: 'password123'
   (0.0ms)  begin transaction
   (0.0ms)  rollback transaction

irb(main):003:0> u.errors.messages
=> {:password=>["has been compromised in an earlier password breach.
  To protect your data, we don't allow using passwords that might
  already be in the hands of people with bad intentions. This is a
  good time to choose a password that you have never used before."]}

```

There you go!
he world's most used password has been detected. At least no one registering with your site will ever again be able to jeopardize their account and your site by using a password that is already breached.

Next time I'll show you how to use the `rake seed` task to generate other combinations of *top N passwords* and *false positive probability* and ship the new generated bloom filter with your app.
