# redmine_acts_as_taggable_on

## NOTICE: this project is currently unmaintained.

See this fork instead, which is more up-to-date: https://github.com/alexbevi/redmine_acts_as_taggable_on

---

`redmine_acts_as_taggable_on` is a gem which allows multiple Redmine plugins to
use the tables provided by `acts-as-taggable-on` without stepping on each
others' toes.

## Why?

The problem we ran into when we discovered that both `redmine_tags` and
`redmine_knowledgebase` want to use the `acts-as-taggable-on` gem is that after
either is installed, the migration for the other fails, since the database
tables already exist.

Additionally, the plugins must choose between two less than ideal options when
uninstalling:

* Drop the tables, destroying data which may still be in use by another plugin
* Leave the tables there, violating the user's expectation that the database
  should be back to how it was before the plugin was installed

`redmine_acts_as_taggable_on` solves this issue by giving Redmine plugins a
mechanism to declare that they require these tables, and providing intelligent
migrations which only drop the tables when no other plugins are using them.

`redmine_acts_as_taggable_on` also provides a limited defence against plugins
which directly depend on `acts-as-taggable-on`. It does this by grepping
through their Gemfiles for the string `acts-as-taggable-on` -- if found, it
will treat the plugin as requiring the `acts-as-taggable-on` tables, and will
print a gentle(-ish) suggestion to use this gem instead.

## Status and Compatibility

`redmine_acts_as_taggable_on` should work with every 2.x release of Redmine.

To put your mind at ease: there is an automated test suite, which is run on
[Travis CI](https://travis-ci.org/hdgarrood/redmine_acts_as_taggable_on), on
each of the most recent releases of every 2.x branch and trunk, on every MRI
version supported by Redmine for that particular release (that's a build matrix
of 12 separate builds in total).

This gem is used by the following plugins:

* [redmine_blogs](https://github.com/ichizok/redmine_blogs)
* [redmine_knowledgebase](https://github.com/alexbevi/redmine_knowledgebase)
* [redmine_tags](https://github.com/ixti/redmine_tags)

It is not compatible with Redmine 1.x.

## Limitations

This gem cannot currently protect against situations where a plugin directly
using `acts-as-taggable-on` has put the generated migration into its db/migrate
directory. If a user tries to migrate the plugin down by executing `rake
redmine:plugins:migrate VERSION=0 NAME=redmine_foo_plugin`, the tables will
still be dropped, regardless of whether other plugins are still using them.

I'm in two minds about whether to fix this: on the one hand, it would require
some nasty monkey-patching. On the other hand, data loss is no fun at all.

## Setup

Add it to your plugin's Gemfile:

    gem 'redmine_acts_as_taggable_on', '~> 1.0'

**Note**: Please use the above line _verbatim_. This is to avoid being told by
Bundler that we aren't allowed to use multiple declarations of the same gem
with different version requirements.

I've chosen the above requirement to ensure that there are no accidental
breakages -- the public API (that is, the following paragraph) will remain the
same throughout version 1.0. If it does need to change, I'll bump the major
version. This should ensure that Bundler only refuses to bundle when there is
an actual incompatibility.

Add the migration:

    $ echo 'class AddTagsAndTaggings < RedmineActsAsTaggableOn::Migration; end'\
        > db/migrate/001_add_tags_and_taggings.rb

Declare that your plugin needs `redmine_acts_as_taggable_on` inside init.rb:

    require 'redmine_acts_as_taggable_on/initialize'

    Redmine::Plugin.register :my_plugin do
      requires_acts_as_taggable_on
      ...

That's it. Your plugin will now migrate up and down intelligently. For example:

    $ cd /path/to/redmine
    $ cp -r /some/plugin/using/this/gem       plugins/redmine_foo
    $ cp -r /another/plugin/using/this/gem    plugins/redmine_bar
    $ rake redmine:plugins:migrate
    Migrating redmine_bar (Redmine bar)...
    ==  AddTagsAndTaggings: migrating =========================================
    -- create_table(:tags)
       -> 0.0039s
    -- create_table(:taggings)
       -> 0.0007s
    -- add_index(:taggings, :tag_id)
       -> 0.0003s
    -- add_index(:taggings, [:taggable_id, :taggable_type, :context])
       -> 0.0003s
    ==  AddTagsAndTaggings: migrated (0.0059s) ================================

    Migrating redmine_foo (Redmine foo)...
    ==  AddTagsAndTaggings: migrating =========================================
    -- Not creating "tags" and "taggings" because they already exist
    ==  AddTagsAndTaggings: migrated (0.0006s) ================================

You can look at the [test script](test/test.bats) to see how other situations
are handled.
