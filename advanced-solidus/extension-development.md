# Extension development

## Intro to extension development

We've already talked about how to use extensions in [Using extensions](../getting-started/extensions.md). As you may already know, extensions are Rails engines that augment your Solidus store with additional functionality such as payment gateways, WMS integrations, social login, and more. Using extensions when developing your store can be a huge time and money saver since it spares you the need to reimplement common (and often complicated) functionality.

But how would you go about creating your own extension? Perhaps you found an unsolved problem in the ecosystem, or you need to implement a feature or integration that you think others could benefit from. Perhaps, you would like to create an open-source extension or, maybe, you want to keep the extension private and only allow your team to access it.

Whatever the use case, this guide has got you covered. We'll see how to create an extension from scratch and release it as open-source on RubyGems or on a private gem server.

## Creating your first extension

In this example, we'll create an extension for integrating with Acme Fulfillment, a fulfillment provider — this is a very common use case in eCommerce. Our extension will do the following:

1. When an order is finalized, it will call the fulfillment partner's API to send all information about the order, so that it can be packaged and shipped to the customer.
2. It will store the fulfillment partner's shipment ID in the Solidus DB, for easier inspection and debugging.

Let's dive right in!

### Generating the skeleton

When working with extensions and Rails engines in general, there's a lot of boilerplate involved. Over the years, we've developed tools that simplify most of the tasks around extension creation and maintenance.

We've consolidated this tooling in [`solidus_dev_support`](https://github.com/solidusio/solidus\_dev\_support), a gem whose job is to make it a no-brainer to develop Solidus extensions. The gem provides static and runtime utilities that help you create a new extension, make it compatible with different Solidus versions, release it and maintain it over time.

{% hint style="info" %}
`solidus_dev_support` provides much more functionality than we'd be able to cover in this guide, so you should definitely check out its [documentation](https://github.com/solidusio/solidus\_dev\_support) to get an idea of how much it does.
{% endhint %}

The first step is to install the gem globally so that you can use it from your console:

```bash
$ gem install solidus_dev_support
```

You can now generate a new extension with the `solidus extension` command. Solidus extensions are generally named `solidus_`, and then the name of the feature or integration they provide, so we'll follow this convention for our Acme Fulfillment extension:

```bash
$ solidus extension solidus_acme_fulfillment
```

This will generate a `solidus_acme_fulfillment` directory in the current path. This directory will contain a lot of boilerplate that is required to make your extension play nice with Solidus and Rails conventions.

`solidus_dev_support` tries to be as smart as possible and use some sensible defaults for your extension, but we'll still need to adjust some values before we can proceed.

First of all, open `solidus_acme_fulfillment.gemspec` in your favorite text editor, and change the following lines to configure the required fields for your extension:

{% code title="solidus_acme_fulfillment.gemspec" %}
```diff
# ...
-  spec.authors = ['TODO: Write your name']
-  spec.email = 'TODO: Write your email address'
+  spec.authors = ['Alice']
+  spec.email = 'alice@mail.com'

- spec.summary = 'TODO: Write a short summary, because RubyGems requires one.'
+ spec.summary = 'A Solidus extension to integrate with the Acme Fulfillment API.'
- spec.description = 'TODO: Write a longer description or delete this line.'

# ...
```
{% endcode %}

Now that our gemspec doesn't have any TODOs in it, we can install the extension's bundle, which contains some useful tooling for working with the extension:

```
$ bundle install
```

Now that the gemspec is configured and the bundle installed, we're ready to write some code!

### Adding dependencies

In order to make API calls, we'll need a library that makes it easier to perform HTTP requests. We could use Ruby's `HTTP` module, but its API is kind of cumbersome and not really fun to work with. Instead, we'll go with the popular [`httparty`](https://github.com/jnunemaker/httparty) gem. In order to do that, let's add the dependency to our gemspec:

{% code title="solidus_acme_fulfillment.gemspec" %}
```diff
 Gem::Specification.new do |spec|
   # ...

   spec.add_dependency 'solidus_core', ['>= 2.0.0', '< 4']
   spec.add_dependency 'solidus_support', '~> 0.5'
+  spec.add_dependency 'httparty', '~> 0.20.0'
 
   # ...
 end
```
{% endcode %}

This will ensure `httparty` is also installed by any apps that install our extension. We can now reinstall the bundle to get the new gem:

```bash
$ bundle install
```

Finally, we'll need to require the `httparty` gem in our extension's main file, since gem dependencies are not autoloaded by Bundler when initializing the main app:

{% code title="lib/solidus_acme_fulfillment.rb" %}
```diff
+require 'httparty'
+
 require 'solidus_acme_fulfillment/configuration'
 require 'solidus_acme_fulfillment/version'
 require 'solidus_acme_fulfillment/engine'
```
{% endcode %}

We should now be able to access `httparty` everywhere in our extension's code!

### Accepting configuration values

A common pattern in extensions is to accept certain configuration values that the user can change in a Rails initializer. For this reason, the extension skeleton generated by `solidus_dev_support` ships with a sample configuration file where you can add any options that you want the user to be able to configure. The skeleton also contains an initializer which will be copied to the main app when the extension is installed, so that the user doesn't have to write the configuration code manually.

In the case of our Acme Fulfillment extension, we want to let the user configure their API key. In order to do this, let's edit the `lib/solidus_acme_fulfillment/configuration.rb` file as follows:

{% code title="lib/solidus_acme_fulfillment/configuration.rb" %}
```diff
 module SolidusAcmeFulfillment
   class Configuration
-    # Define here the settings for this extension, e.g.:
-    #
-    # attr_accessor :my_setting
+    attr_accessor :api_key
   end

   # ...
 end
```
{% endcode %}

We will also edit the sample initializer to let the user know about the new configuration option:

{% code title="lib/generators/solidus_acme_fulfillment/install/templates/initializer.rb" %}
```diff
 SolidusAcmeFulfillment.configure do |config|
-  # TODO: Remember to change this with the actual preferences you have implemented!
-  # config.sample_preference = 'sample_value'
+  # Set your Acme Fulfillment API key here.
+  config.api_key = 'my-api-key'
 end
```
{% endcode %}

When a user installs our extension, an initializer will be added to their main application under `config/initializers/solidus_acme_fulfillment.rb` which will contain our sample configuration.

### Writing your first feature

Customizing Solidus through an extension is very similar to customizing it in the main application, and the same rules and patterns apply: you can use extension hooks, the event bus, overrides, etc.

{% hint style="warning" %}
One important aspect to keep in mind when working on extensions is that you can't predict what other extensions the user will install, so you need to make sure your customizations play nice with other extensions.

For example, setting a configuration value in the Solidus configuration is usually discouraged in extensions, since other extensions may do the same and end up overriding your setting. Instead, you can change the value by altering the configuration in the main app through your installation initializer, or document that the value needs to be set in your readme, and let the user do it.
{% endhint %}

When we described our requirements, we mentioned we want to save the shipment ID that we get back from the 3PL's API when we create the order, so that we can easily access the shipment later. The best place to store this information would be an additional column in the `spree_orders` table, so let's first write a migration to create it:

```bash
$ bin/rails g migration AddAcmeFulfillmentShipmentIdToSpreeOrders \
    acme_fulfillment_shipment_id
```

{% hint style="info" %}
You may have noticed that we're not running `rails db:migrate` after generating the migration. This is because we're working in a Rails engine, not a Rails application. We don't have a database to work on.

All database migrations that you generate in your extension will be automatically imported into the main application when the extension is installed via its initializer.
{% endhint %}

Next, we need to implement the actual code to integrate with the fulfillment partner's API. An event subscriber seems like the best way to do this, so let's write one:

{% code title="app/subscribers/solidus_acme_fulfillment/order_subscriber.rb" %}
```ruby
module SolidusAcmeFulfillment
  class OrderSubscriber
    include Omnes::Subscriber
    
    handle :order_finalized, with: :send_to_3pl
    
    def send_to_3pl(event)
      order = event.payload.fetch(:order)
      
      response = HTTParty.post(
        'https://api.acmefulfillment.com/orders', 
        headers: {
          'Authorization' => "Bearer #{SolidusAcmeFulfillment.config.api_key}",
          'Content-Type' => 'application/json',
          'Accept' => 'application/json',
        },
        body: serialize_order(order).to_json,
      )
      
      order.update!(
        acme_fulfillment_shipment_id: response.parsed_response['id'],
      )
    end
    
    private
    
    def serialize_order(order)
      {
        # ...
      }
    end
  end
end
```
{% endcode %}

Our event subscriber is pretty simple: it listens to the `:order_finalized`event and, when it's published, it calls the Acme Fulfillment API with the configured API key and the serialized order information. It then parses the API response and sets the `acme_fulfillment_shipment_id` column on the order to the ID returned by the fulfillment partner's API.

{% hint style="warning" %}
In the real world, you'd want to move this block of code to a background job, so that it doesn't unnecessarily slow down your user's HTTP requests with API calls to your fulfillment partner. &#x20;
{% endhint %}

We still need to subscribe it to the Solidus event bus system. We can do it in our extension's engine file:

{% code title="lib/solidus_acme_fullfilment/engine.rb" %}
```diff
module SolidusAcmeFulfillment
  class Engine < Rails::Engine
    # ...
+
+    config.to_prepare do
+      SolidusAcmeFulfillment::OrderSubscriber.new.subscribe_to(Spree::Bus)
+    end
   end
 end
```
{% endcode %}

That's all we needed! The requirements have been satisfied, and it's now time to preview our work. In order to do that, we'll use the **sandbox app.**

### Using the sandbox app

Because extensions are Rails engines, they can't be previewed as easily as we'd do when customizing our main app: there's no underlying Rails/Solidus application to run the extension. You could install your extension in an existing Solidus app and preview it there, but this can be slow and cumbersome, especially when you're still actively working on the extension.

Luckily, `solidus_dev_support` provides a Rake task we can run to generate a "sandbox app", i.e. a barebones Rails + Solidus application with our extension already installed and configured. The sandbox app is extremely useful in extension development, and it's important to learn to make the best of it.

To generate the sandbox app, simply run the following command:

```
$ bin/sandbox
```

The generation might take a couple of minutes, so sit tight and relax! The process will also ask you a few times whether migrations should be run immediately or manually at a later stage — you want to run them immediately, which is also the default selection. This will save you a few seconds of work.

Once the process has been completed, you'll find a new `sandbox` directory at the root of your extension. This contains your new shiny sandbox app. Your extension has already been installed and configured inside the app: try looking for the `config/initializers/solidus_acme_fulfillment.rb` initializer.

{% hint style="warning" %}
The sandbox app is ephemeral and intended for development/test purposes only: the sandbox path is ignored by Git, so any changes you make here will be lost permanently if you remove the `sandbox` directory.
{% endhint %}

`solidus_dev_support` also allows you to run commands in your sandbox app from the root of your extension, just as you would do with a regular Rails application. Try spinning up a Rails server:

```
$ bin/rails server
```

This should boot your sandbox app and serve it at [http://localhost:3000](http://localhost:3000), so that you can preview your extension as you work on it!

{% hint style="info" %}
All `rails` commands will be delegated to the sandbox app.

One exception is the `rails g` /`rails generate` command, which will be run in your extension (since that's usually the intended behavior). If you need to run a generator in the sandbox app, you'll have to first `cd` into the `sandbox` directory.

As an alternative, you can use `bin/rails-engine` and `bin/rails-sandbox` to force a command to run in the engine or in the sandbox respectively.
{% endhint %}

### Releasing the extension

{% hint style="info" %}
This step is optional but recommended. You _could_ keep your extension in a private or public GitHub repository and download it directly from there, but you'd miss out on the benefits of properly versioning your extension, which makes it easier to maintain and upgrade it.
{% endhint %}

Like all gems, Solidus extensions can be released to any public or private gem servers, such as [RubyGems](https://rubygems.org/) or [Gemfury](https://gemfury.com/). Releasing your extension allows you to package it in a convenient way and follow an established versioning scheme much more easily than simply pulling code from GitHub. In general, you should release the first version of your extension as soon as you start using it in production.

By default, the extension skeleton generated by `solidus_dev_support` is configured to release your gem on RubyGems. If you're using a different gem server, they should provide instructions on how to properly configure your gemspec.

Once you've configured your gem server you can release your extension with the following command:

```
$ gem bump -v 1.0.0 -t # bump the extension version to 1.0.0
$ bin/rake changelog # generate the new release's changelog
$ git commit -a --amend # update the version bump commit
$ git push --follow-tags # push the version bump to GitHub
$ gem release # release the extension on the gem server
```

{% hint style="danger" %}
The `bin/rake changelog` command will work as long as you're hosting your repository on GitHub. Otherwise, you need to manually create it or use another tool.

You might also need to [generate a new GitHub token](https://github.com/settings/tokens/new) and provide it as an environment variable. E.g.: `CHANGELOG_GITHUB_TOKEN=my_token bin/rake changelog`\
See [github-changelog-generator](https://github.com/github-changelog-generator/github-changelog-generator) (used under the hood) for details.
{% endhint %}

{% hint style="info" %}
In the Solidus ecosystem, we follow [Semantic Versioning](https://semver.org/) for assigning version numbers to our releases. It is strongly recommended you do the same, in order not to break the expectations of experienced Solidus developers.
{% endhint %}

### Installing the extension

Once you've released your extension (or just pushed it to GitHub), you can install it in any Solidus store by following the [usual instructions](../getting-started/extensions.md#installing-an-extension). First of all, add the extension to your Gemfile:

{% code title="Gemfile" %}
```ruby
# ...

# Use this if you released your extension to a gem server:
gem 'solidus_acme_fulfillment'

# Use this if you simply host your extension on GitHub:
gem 'solidus_acme_fulfillment', github: 'your-org/solidus_acme_fulfillment'
```
{% endcode %}

Then install the bundle:

```
$ bundle install
```

Finally, run your extension's install generator to copy all relevant files to your application:

```
$ bin/rails g solidus_acme_fulfillment:install
```

The generator will ask you whether to run migrations immediately. If you choose not to do it, you can always do it yourself with `bin/rails db:migrate`.

As the last step, you may want to review and customize the files generated by your extension. In our example, you should set your API key in the `config/initializers/solidus_acme_fulfillment.rb` file.

That's it! Your extension is now fully installed and running in your app.

## Extension best practices

The following section contains some advanced recommendations for extension design, development, and maintenance. By following these best practices, you'll make your extension future-proof and compatible with the vast majority of Solidus applications.

### Don't override, extend

The first and most important rule of good extension design is to avoid [overrides](../customization/customizing-the-core.md#using-overrides) at all costs. Overrides in extensions have the same problems as overrides in the main app: because you're directly altering third-party code, they are hard to maintain and hard to test.

In extensions, overrides become even more of a problem, because multiple extensions may override the same pieces of Solidus! This can lead to a mess of tangled overrides that are in conflict with each other. Also, overrides don't play nice with IDE autocompletion, and they will make it more difficult for users of your extension to figure out your extension's API.

Sometimes, overrides are inevitable, but you should always look for alternatives. Solidus allows users to customize the vast majority of [service objects](../customization/customizing-the-core.md#using-built-in-hooks) used by the framework. Whenever possible, you should leverage those configuration options instead of overriding the existing service objects. The [event bus](../customization/subscribing-to-events.md) is another good option which you should learn to rely on.

Instead of altering the upstream version, try to find a way to provide the desired functionality with new code which extends or wraps the original implementation. This might mean writing a bit more code, but it will pay off greatly when you need to update your extension for compatibility with new Solidus versions, or when users need to understand how your extension interacts with the rest of their application.

We've already seen a good example of it. We could have overridden the [`Spree::Order#finalize`](https://github.com/solidusio/solidus/blob/4cc1d5aeb0bd0ef9cc045be1c1083d1c702b5264/core/app/models/spree/order.rb#L737)method and added the API call to Acme Fulfillment's API there. However, that would have been brittle, as there's no guarantee the method won't change in subsequent Solidus releases. We can't either anticipate if stores or other extensions are messing with it. Having added the feature as an event handler frees us from all those problems and leaves our code completely scoped to our own domain.

### Support internationalization

Many Solidus stores are international, or plan to be at some point in the future. It's important that your extension is completely translatable, so that users can easily translate the UI and any other content it provides into the languages their store supports.

There's really no magic when it comes to making Solidus extensions translatable, so we recommend checking out the [Rails Internalization guide](https://guides.rubyonrails.org/i18n.html).

### Avoid storefront code

We strongly discourage extensions from attempting to alter or extend the storefront in any way, since Solidus storefronts come in a lot of different shapes and forms: some storefronts are monolithic and rely on plain old ERB, SASS, and JS; others use React, Vue, or Stimulus + View Components; others still use Solidus as a headless solution, interacting with the framework through the REST or GraphQL API.

As you can imagine, it would be impossible for an extension to provide all the possible variations of storefront integrations. Furthermore, customizing the storefront code to fit the specific storefront's needs is almost always more work than attempting to integrate the extension in the storefront from scratch.

{% hint style="warning" %}
In the past, some extensions used to provide storefront code, either through Deface overrides or in the form of new views that the user could include in their main storefront. We have since moved away from this practice and such code should be considered deprecated.
{% endhint %}

What we do recommend is documenting your extension thoroughly, so that other developers can easily understand it and use it in different ways.

Sometimes, it's also useful to provide some examples of storefront integrations that users can copy-paste or use as inspiration when integrating their extension in their own store!

### Design extension hooks

When designing your extension, you should always be on the lookout for ways to make it more... extendable. Just like Solidus users customize the core to reach their goals, you should also expect that they will want to customize the behavior of your extension to fit their use case.

Often, you can let them do this through plain old configuration switches, but sometimes you can't anticipate all the possible use cases and it's simpler and more flexible to let users provide their own implementation for certain pieces of your extension.

In our [original example](extension-development.md#writing-your-first-feature), a good candidate for an extension hook would be the API serialization logic, which could be implemented as:

{% code title="app/subscribers/solidus_acme_fulfillment/order_subscriber.rb" %}
```ruby
module SolidusAcmeFulfillment
  class OrderSubscriber
    # ...
  
    def serialize_order(order)
      SolidusAcmeFulfillment::OrderSerializer.new(order).serialize
    end
  end
end
```
{% endcode %}

Users may want to pass custom fields to the 3PL API, or override the ones you're setting. To accomplish this easily, you can allow them to provide their own API serializer class.

The process is pretty simple. First of all, add a configuration option:

{% code title="lib/solidus_acme_fulfillment/configuration.rb" %}
```diff
 module SolidusAcmeFulfillment
   class Configuration
     # ...
+    attr_accessor :order_serializer_class
+
+    def initialize
+      # Set the default order serializer to our own implementation.
+      @order_serializer_class = 'SolidusAcmeFulfillment::OrderSerializer'
+    end
   end

   # ...
 end
```
{% endcode %}

Also, make sure to add the new option to your initializer template:

{% code title="lib/generators/solidus_acme_fulfillment/install/templates/initializer.rb" %}
```diff
 SolidusAcmeFulfillment.configure do |config|
+  # ....
+
+  # This class is used to serializer orders sent to the 3PL API.
+  # You can override it with your own implementation.
+  config.order_serializer_class = 'SolidusAcmeFulfillment::OrderSerializer'
 end
```
{% endcode %}

Next, implement the default serializer, by extracting it from the code you already have:

{% code title="app/serializers/solidus:acme:fulfillment/order:serializer.rb" %}
```ruby
module SolidusAcmeFulfillment
  class OrderSerializer
    def call(order)
      {
        # ...
      }
    end
  end
end
```
{% endcode %}

Finally, call the configured serializer from the subscriber you've implemented:

{% code title="app/subscribers/solidus_acme_fulfillment/order_subscriber.rb" %}
```diff
 module SolidusAcmeFulfillment
   class OrderSubscriber
     # ...
    
     def serialize_order(order)
-      SolidusAcmeFulfillment::OrderSerializer.new(order).serialize
+      SolidusAcmeFulfillment
+        .config
+        .order_serializer_class
+        .constantize
+        .new
+        .call(order)
     end
   end
 end
```
{% endcode %}

That's all you need! Users of your extension can now provide their own API serializer by implementing it in their app and setting the `order_serializer_class` configuration option.

Another great option to provide extensibility is to publish your own events on the Solidus' event bus. When you do that, it's a good practice to prefix their name with your extension's name.

For instance, we can do better at error handling and broadcast whenever we have a failure in our response:

{% code title="app/subscribers/solidus_acme_fulfillment/order_subscriber.rb" %}
```diff
-      order.update!(
-        acme_fulfillment_shipment_id: response.parsed_response['id'],
-      )
+      if response.success?
+        order.update!(
+          acme_fulfillment_shipment_id: response.parsed_response['id'],
+        )
+      else
+        Spree::Bus.publish(:"solidus_acme_fulfillment.response_error", response: response)
+      end
```
{% endcode %}

Don't forget to register the new event name in the engine file:

{% code title="" %}
```diff
config.to_prepare do
+  Spree::Bus.register(:"solidus_acme_fulfillment.response_error")
   SolidusAcmeFulfillment::OrderSubscriber.new.subscribe_to(Spree::Bus)
end
```
{% endcode %}

### Automate testing with CI

{% hint style="info" %}
[CircleCI](https://circleci.com/) is an extremely powerful platform, and an in-depth explanation of its architecture is out of the scope of this guide. The following paragraphs assume you are familiar with CircleCI and [CircleCI Orbs](https://circleci.com/docs/2.0/orb-intro/). If you are not, we recommend reading the relevant documentation first.
{% endhint %}

The Solidus ecosystem is extremely large and varied. For lots of stores with extensive customizations, upgrading as soon as a new version of Solidus is released is simply not feasible, as it would take too much work and distract the engineering department from other priorities. To give Solidus users a smooth upgrade path, we commit to maintaining all Solidus versions [for 18 months after their release](https://solidus.io/security). This ensures stores have plenty of time to upgrade their Solidus version.

Official extensions follow the same policy, while community-maintained extensions are expected to do the same. We aim to support all currently maintained Solidus versions so that users on older versions are not "left behind" as the ecosystem moves forward. This means that all extensions should be tested against all the currently supported Solidus versions so that no incompatible changes are inadvertently introduced in the extension's code.

We know that this can be a burden for extension maintainers, so we've developed a set of tools to help with the process, like the [`@solidusio/extensions` CircleCI orb](https://circleci.com/developer/orbs/orb/solidusio/extensions). The orb will automatically test your Solidus extension against the right Solidus versions, without the need for you to update the versions list manually. The orb will even periodically test your extension against the latest `master` branch of Solidus, so that you know whether your extension is compatible with the _upcoming_ version of Solidus!

{% hint style="info" %}
If you have generated your extension with `solidus_dev_support`, your extension is already configured for testing via CircleCI, and you just need to [follow the project](https://circleci.com/docs/2.0/project-build/#adding-projects) on CircleCI!
{% endhint %}

Here's a sample CircleCI configuration for a Solidus extension:

```yaml
version: 2.1

orbs:
  solidusio_extensions: solidusio/extensions@volatile

jobs:
  # Test with MySQL
  run-specs-with-mysql:
    executor: solidusio_extensions/mysql
    steps:
      - solidusio_extensions/run-tests
  # Test with PostgreSQL
  run-specs-with-postgres:
    executor: solidusio_extensions/postgres
    steps:
      - solidusio_extensions/run-tests

workflows:
  # Test all commits against the supported Solidus versions
  # and the latest master branch from Solidus
  Run specs on supported Solidus versions:
    jobs:
      - run-specs-with-postgres
      - run-specs-with-mysql
  # Weekly test the extension's master branch against the
  # supported Solidus versions and the latest master branch
  # from Solidus
  Weekly run specs against master:
    jobs:
      - run-specs-with-postgres
      - run-specs-with-mysql
    triggers:
      - schedule:
          cron: 0 0 * * 4
          filters:
            branches:
              only:
                - master
```

As you can read in the comments, the configuration above will:

* Test every commit in `master` and in other branches against the currently supported Solidus versions, as well as against the latest `master`, in order to ensure the correctness of any code changes you push to the extension.
* Test the current `master` weekly against the currently supported Solidus versions, as well as against the latest `master`, in order to ensure your extension's code is compatible with the upcoming Solidus release.

The tests will be run both with MySQL and PostgreSQL since Solidus supports both.

### Write engine-specific code

As you probably already know, Solidus doesn't come as a monolithic piece of code. Instead, you can switch the default frontend, backend, and API components for your own. When developing an extension, you can't just assume that the whole default ecosystem is present. The only thing you can be certain of is that the `solidus_core` gem is there.

However, you might want to extend the behavior of one of the Solidus sub-components. If `solidus_dev_support` is your friend when it comes to the development chores, another gem, [`solidus_support`](https://github.com/solidusio/solidus\_support), has your back covered when talking about your extension runtime and its compatibility with different Solidus installations and versions. By the way, if you created your extension with `solidus_dev_support`, you don't need to add `solidus_support` to your list of dependencies as it's already there.

Say that you'd like to add an API endpoint to return the associated partner's fulfillment id for a given order. Thanks to `solidus_support`, you can place your controller in `lib/controllers/api/` and it'll be automatically picked up.

{% code title="lib/controllers/api/spree/solidus_acme_fulfillment/shipments_controller.rb" %}
```ruby
# frozen_string_literal: true

class Spree::SolidusAcmeFulfillment::ShipmentsController < Spree::Api::BaseController
  def by_order_id
    id = params[:id]
    order = Spree::Order.find(id)

    respond_with({
      id: id,
      acme_fulfillment_shipment_id: order.acme_fulfillment_shipment_id
    })
  end
end
```
{% endcode %}

{% hint style="info" %}
By inheriting from `Spree::Api::BaseController`, you have access to some of Solidus API's conventions out of the box. For instance, your endpoint is automatically under authentication. See the [file definition](https://github.com/solidusio/solidus/blob/master/api/app/controllers/spree/api/base\_controller.rb) for more details.
{% endhint %}

All that is left is adding the corresponding route to your extension's routes file. However, we need to add it only when the API component is present:

{% code title="config/routes.rb" %}
```diff
 Spree::Core::Engine.routes.draw do
-  # Add your extension routes here
+  if SolidusSupport.api_available?
+    namespace :solidus_acme_fulfillment do
+      get '/shipments/by_order_id/:id', to: 'shipments#by_order_id'
+    end
+  end
 end
```
{% endcode %}

You can now try your new route, served by the sandbox application. After restarting the server, you can use `curl` to access it. You'll need a user's `spree_api_key` to authenticate them:

```
curl http://localhost:3000/solidus_acme_fulfillment/shipments/by_order_id/1 \
     --header "Authorization: Bearer 44abb4fda97e1a22da9a837d8705a3f492392453c37164c7" \
     --header "Accept: application/json"
```

{% hint style="info" %}
Note that you can also use the following path structures:

* `lib/views/{engine}` (e.g., `lib/views/api`)
* `lib/controllers/{engine}` (e.g., `lib/controllers/backend`)

`solidus_support` is smart enough to pick them up and only load the views/controllers when the corresponding Solidus engine has been loaded!
{% endhint %}

### Write backward-compatible extensions

One aspect to keep in mind when writing Solidus extensions is backwards compatibility: ideally, your extension should always be compatible with all the currently supported Solidus versions.

In some cases, this can be done with some simple tricks. In more complicated scenarios, `solidus_support` provides you with all the tools you need. We encourage you to check out [`solidus_support`'s source code](https://github.com/solidusio/solidus\_support) to familiarize with the helpers it offers.&#x20;

#### Migrations

By default, Rails generated the migration you created as being compatible with its latest version. However, Solidus also supports older releases. Your migrations should be compatible with the oldest Rails version still supported by Solidus:

{% code title="db/migrate/20220602090213_add_acme_fulfillment_shipment_id_to_spree_orders.rb" %}
```diff
-class AddAcmeFulfillmentShipmentIdToSpreeOrders < ActiveRecord::Migration[7.0]
+class AddAcmeFulfillmentShipmentIdToSpreeOrders < ActiveRecord::Migration[5.2]
```
{% endcode %}

#### Event bus

`solidus_support` is also your friend when you want to support past Solidus versions which are still maintained. The event subscriber we created is only valid for Solidus versions greater than 3.2. Before that, [a more basic messaging system was available](../customization/subscribing-to-events.md#upgrading-from-the-legacy-event-system).&#x20;

By including the `SolidusSupport::LegacyEventCompat::Subscriber` module, you'll be able to support both event subscribers:

{% code title="app/subscribers/solidus_acme_fulfillment/order_subscriber.rb" %}
```diff
 module SolidusAcmeFulfillment
-  class OrderSubscriber
-    include Omnes::Subscriber
+  module OrderSubscriber
+    include Spree::Event::Subscriber
+    include SolidusSupport::LegacyEventCompat::Subscriber

-    handle :order_finalized, with: :send_to_3pl
+    event_action :send_to_3pl, event_name: :order_finalized
```
{% endcode %}

Finally, you need to update your engine file to register your extension's events, but only when the new event subscriber is being used:

{% code title="lib/solidus_acme_fulfillment/engine.rb" %}
```diff
config.to_prepare do
-  Spree::Bus.register(:"solidus_acme_fulfillment.response_error")
-  SolidusAcmeFulfillment::OrderSubscriber.new.subscribe_to(Spree::Bus)
+  unless SolidusSupport::LegacyEventCompat.using_legacy?
+    Spree::Bus.register(:"solidus_acme_fulfillment.response_error")
+    SolidusAcmeFulfillment::OrderSubscriber.omnes_subscriber.subscribe_to(Spree::Bus)
+  end
end
```
{% endcode %}

{% hint style="info" %}
`#omnes_subscriber` returns the legacy subscriber transformed in the new format, so that it works with the new event bus.
{% endhint %}
