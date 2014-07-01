# Week 1

To browse the resultant code of this week, see the [`week_1` tag](https://github.com/sweatypitts/workflowy_clone_project/tree/week_1)

## Create app

```shell
rails new workflowy_clone_project --skip-javascript --skip-test-unit
cd workflowy_clone_project
rake db:migrate # in order to create schema.rb, required for loading app
rvm current
atom
```

Open up `.gitignore` and add `.DS_Store` to the end if you're on a Mac.

Also add `/vendor/assets/lib` no matter which platform. You'll see why later.

## Update Gemfile

It should look thusly. After updating `bundle install`.

```ruby
source 'https://rubygems.org'

ruby '2.1.1'

gem 'rails', '4.0.3'

gem 'sqlite3'

gem 'coffee-rails'
gem 'haml-rails'
gem 'sass-rails'

gem 'therubyracer', platforms: :ruby

gem 'jbuilder'

gem 'debugger'

group :test do
  gem 'capybara'
  gem 'cucumber-rails', require: false
  gem 'database_cleaner'
  gem 'factory_girl_rails'
  gem 'ffaker'
  gem 'poltergeist'
  gem 'rspec-rails'
end
```

## Install RSpec

```shell
rails generate rspec:install
```

Add rspec to default task in `lib/tasks/test.rake`

```ruby
require 'rspec/core/rake_task'

RSpec::Core::RakeTask.new(:spec)

task :default => :spec
```

Run `rake`, observe lack of tests

Open `spec/spec_helper.rb`

Add the following requires:

```ruby
require 'factory_girl'
require 'database_cleaner'
```

And make the `RSpec.config` block look like so:

```ruby
RSpec.configure do |config|
  config.include FactoryGirl::Syntax::Methods
  config.mock_with :rspec
  config.order = "random"

  config.before(:suite) do
    DatabaseCleaner.strategy = :transaction
    DatabaseCleaner.clean_with(:truncation, pre_count: true)
  end

  config.before(:each) do
    DatabaseCleaner.start
  end

  config.after(:each) do
    DatabaseCleaner.clean
  end
end
```

## Install Cucumber

```shell
rails generate cucumber:install
```

Cucumber adds itself to the default rake task, so we should now be able to run `rake` and run both cukes and specs. Give it a try:

```shell
rake
```

## Write our first cuke

Add `features/user_views_items.feature` file, open it.

Now add this feature:

```gherkin
Feature: User views items
  In order to see the lists I've created
  As a user
  I want to see all of my lists and items

  @javascript
  Scenario: Success
    Given I have "3" items
    When I visit "/"
    Then I should see my items
```

Now `rake`. We should have a bunch of undefined steps now. Let's add them.

In `features/step_definitions/navigation_steps.rb`:

```ruby
When(/^I visit "(.*?)"$/) do |arg1|
  pending # express the regexp above with the code you wish you had
end
```

In `features/step_definitions/item_steps.rb`:

```ruby
Given(/^I have "(.*?)" items?$/) do |arg1|
  pending # express the regexp above with the code you wish you had
end

Then(/^I should see my items?$/) do
  pending # express the regexp above with the code you wish you had
end
```

Now if you run `rake` again, you should see different output. The first step should come up as pending, and the rest skipped. Let's start defining the step definitions now. We'll do them in the order in which they're called. Make the first one look like so:

```ruby
When(/^I visit "(.*?)"$/) do |path|
  visit path
end
```

And `item_steps.rb` like this:

```ruby
Given(/^I have "(\d+)" items?$/) do |kount|
  @items = create_list :item, kount.to_i
end

Then(/^I should see my items?$/) do
  @items.each do |i|
    expect(page).to have_content(i.content)
  end
end
```

Now `rake`. It should fail with this message:

```
No route matches [GET] "/" (ActionController::RoutingError)
./features/step_definitions/navigation_steps.rb:2:in `/^I visit "(.*?)"$/'
features/user/item/create.feature:7:in `When I visit "/"'
```

This is telling is that it can't find the route we told it to go to. This is good, because we haven't made that route yet. Let's do that now.

In `config/routes.rb` delete all the commented garbage and replace it with:

```ruby
root 'items#index'
resources :items
```

`rake` again. It will still fail, as it should. You are now in the middle of the TDD pattern. You write a test defining the behavior you want, then write the code to make the test pass.

The current failure should be telling us we need the `ItemsController`. Let's add that now. Create the file `app/controllers/items_controller.rb` and make it look like this:

```ruby
class ItemsController < ApplicationController

  def index
    @items = Item.all
  end
end
```

`rake` again. Now our template is missing. It can find the action but it doesn't know what to render. Create the file `app/views/items/index.html.haml`. At this point I think we're smart enough to anticipate what the next failure will be, so lets speed things up by making the next step pass too. In order to do that, we need to add the appropriate content to the view we just made, along with defining the next step def.

Make the view look like this:

```haml
%ul
  - @items.each do |i|
    %li= i.content
```

Observe `Item.all` in the controller. The `Item` object does not exist yet. It should be a model, but we haven't made it. Because of this, we know the test will definitely fail. We need to add some more stuff to make our second step pass. In your terminal do the following:

```shell
rails g model item content:string
```

If you've set up Rails as described above, this should only generate two files: the model file and the migration file. Let's run the migration now.

```shell
rake db:migrate
rails runner "Item.create(content: 'asdf')"
```

Now we need to fill in the step def for entering text in the form field. We're going to boot up the development rails server to take a peek at the ids on the form and field we're trying to fill in. Run the following in your terminal:

```shell
rails server
```

Now go to `localhost:3000` in your browser. You should see your item.

Go back to your terminal and shut down the server by doing Ctl+C , then `rake`. Green!

We're not done yet though. We want this page to use ember.js to display and manage items.

## Set up Ember.js

In your terminal run `bower init` and go through the interactive prompt. You can use the defaults, except you want to opt to keep this package private—we're not publishing it, it's for our use only.

Make a new file in the root directory of the project named `.bowerrc` and make it look like this:

```json
{
  "directory": "vendor/assets/lib"
}
```

What this does is save packages managed by bower in the defined directory rather than `./bower_components`, the default. This directory does not need to be checked into source control since they will be defined in `bower.json`. This is why we added this directory to the `.gitignore` earlier.

Now run the following commands:

```shell
bower install jquery#2.1.0 --save
bower install ember#1.6.0-beta.5 --save
bower install ember-data#1.0.0-beta.8 --save
```

Create the following directory: `app/assets/javascripts/app` and create a file called `application.js` in it, with the following contents:

```javascript
window.App = Ember.Application.create();

App.ApplicationStore = DS.Store.extend({
  adapter: DS.ActiveModelAdapter
});
```

This creates our Ember app and tells it what kind of data store to use. We're using the conventional `RESTAdapter`.

Create our Ember model in `app/assets/javascripts/app/models/item.js`:

```javascript
App.Item = DS.Model.extend({
  content: DS.attr('string')
});
```

This tells Ember what our model looks like.

And now our router in `app/assets/javascripts/app/router.js`:

```javascript
App.Router.map(function() {
  this.resource('items', { path: '/' });
});

App.ItemsRoute = Ember.Route.extend({
  model: function() {
    return this.store.find('item');
  }
});
```

Observe the variations on the word 'item' throughout the above files. If we follow convention, Ember makes it all work automagically.

This won't work yet though. We need to hook it all up now.

Rename `app/assets/javascripts/application.js` to `app/assets/javascripts/main.js` and make it look like this:

```javascript
//= require jquery/dist/jquery
//= require handlebars/handlebars
//= require ember/ember
//= require ember-data/ember-data
//= require app
```

This is requiring our bower packages and one more file that we still need to create, `app/assets/javascripts/app.js`:

```javascript
//= require app/application
//= require_directory ./app/models
//= require app/router
```

We are using this file to require all the Ember files.

Because we've included these files in this way, we only need to require one file in our layout view. Rename `app/views/layouts/application.html.erb` to `app/views/layouts/application.html.haml` and make it look like this:

```haml
!!!
%html
  %head
    %title Workflowy Clone Project
    = stylesheet_link_tag 'application', media: "all"
    = javascript_include_tag 'main'
    = csrf_meta_tags
  %body
    = yield
```

Notice how we're requiring the `main.js` file.

One last thing... our `app/views/items/index.html.haml` template needs to be modified to play with Ember:

```haml
%script{ type: 'text/x-handlebars', data: { template_name: 'items' } }
  %ul
    {{#each}}
    %li {{content}}
    {{/each}}
```

The script tag tells Ember what to render here. The double curly braces are a handlebars thing. If you look at it, the above is pretty self-explanatory.

If you fire up your server again, you should still see your item. Run `rake` again. Still green!