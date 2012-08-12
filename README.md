# Lascivious

This plugin simplifies the use of Kiss Metrics with Rails.

Kiss Metrics really works best with Javascript. The problem is that in Rails
the best place to decide whether to fire off an event is inside a Controller.

Using a flash mechanism this plugin provides a series of helper functions to
allow your Controller to inject the correct Javascript into any page.

It's a work-in-progress right now and is pretty dumb: you have to RTFM on when
to insert what. We will probably extend functionality to handle aliasing and
strong typing of users.


## Instructions

1. Install the gem by adding it to your Gemfile
2. Add an initializer in `config/initializers/kiss_metrics.rb` like:

```ruby
Lascivious.setup do |config|
  config.api_key = "0000000000000000000000000000000000000000"
end
```

3. Replace the zeroes with your API key from <https://www.kissmetrics.com/settings>
4. Add the kiss metrics tag to whichever layouts you want to use Kiss Metrics
with, usually all of them, e.g. here's what our header partial looks like:

```erb
<title><%= title %></title>
<%= csrf_meta_tag %>
<link rel="image_src" href="/images/facebook-icon.png"/>
<%= kiss_metrics_tag %>
```

5. Now all you have to do is add Kiss Metrics tags in controllers or views wherever you need something. For instance:

```ruby
class SomeController < ApplicationController
  def index
    kiss_record "SomeController loaded"
  end
end
```

Currently the following commands are provided:

* `kiss_set <message>` - adds a 'set' event, e.g. 'country: uk'
* `kiss_identify <user_id>` - adds an 'identity' event, associating the given
  user ID with the KISSmetrics identifier.
* `kiss_alias <value>` - adds an 'alias' event (weak identifier), e.g. user_id from a tracking cookie that may or may not be on a shared machine
* `kiss_record <message>` - adds a 'record' event with a message of 'message'
* `kiss_metric <event_type> <message>` - adds an event of type 'event_type' with a message of 'message'


## How to Integrate with your app

Our service is built on Rails 3 with Devise and Inherited Resources. This is
how we integrated Kiss Metrics into our app.

### Everywhere

In all cases add this to the HEAD of your layouts:

```erb
<%= kiss_metrics_tag %>
```

And define your keys via an initializer in `app/config/initializers/kiss_metrics.rb`
like this:

```ruby
Lascivious.setup do |config|
  if Rails.env == 'production'
    config.api_key = ENV['KISS_METRICS_API_KEY']
  else
    # Development/testing/staging account key...
    config.api_key = "1111111111111111111111111111111111111111"
  end
end
```

### Sign In

We use a Controller override in Devise as we want to handle a failed login very
specifically. So we have in `app/controllers/users/sessions_controller.rb`:

```ruby
class Users::SessionsController < Devise::SessionsController
  def create
    warden_opts = { :scope => resource_name, :recall => "#{controller_path}#new" }
    resource = warden.authenticate(warden_opts)
    if(resource.nil?)
      kind = :invalid
      resource = build_resource
      resource.errors[:base] = I18n.t("#{resource_name}.#{kind}", {
        scope: "devise.failure",
        default: [kind],
        resource_name: resource_name
      })
    else
      kiss_identify resource.email
      kiss_record "Signed In"
      set_flash_message(:notice, :signed_in) if is_navigational_format?
      sign_in(resource_name, resource)
    end
    respond_with resource, :location => redirect_location(resource_name, resource)
  end
end
```

A much simpler version would be an `after_sign_in_path_for` override in
`app/controllers/application_controller.rb`:

```ruby
private

def after_sign_in_path_for(resource_or_scope)
  kiss_identify resource_or_scope.email unless resource_or_scope.email.nil?
  kiss_record "Signed In"
  scope = Devise::Mapping.find_scope!(resource_or_scope)
  home_path = "#{scope}_root_path"
  respond_to?(home_path, true) ? send(home_path) : root_path
end
```

### Sign Out

We added this to `app/controllers/application_controller.rb`:

```ruby
private

### Record a sign out
def after_sign_out_path_for(resource_or_scope)
  kiss_record "Signed Out"
  new_user_session_path
end
```

The `new_user_session_path` is important: if you redirect to `root_path` you
will be redirected to the login page (as your user will now fail authorization)
and in the process your flash will be wiped clear.

### Email Open

Our Mailers typically look like this:

```ruby
def send_mail(user, recipient, bill_period, subject)
  @recipient ||= user
  @user = user
  @bill_period = bill_period
  @data = collate(@user, @bill_period)

  mail({
    to: formatted_address(@user, @recipient),
    subject: subject
  })
end
```

The `@recipient` variable is important to us, it allows us to send an email
even if we don't have a user setup.

Now inside the email partial we do:

```erb
... our email ERB template ...
  <%= kiss_metrics_email_beacon @recipient.email, "Summary" %>
</body>
</html>
```

Points to note here:

* This only works inside HTML emails and even then not all the time. If this
  doesn't make sense to you go Google 'email pixels' or 'email beacons'
* Change "Summary" to be whichever email variant you have, it could be 'Welcome
  Email' or 'Bill Reminder', etc.
* We've put the `kiss_metrics_email_beacon` at the bottom of the email, right
  before the closing `body` tag. This reduces the chance the pixel kills your
  layout and means the open is only triggered if the email is properly
  downloaded and parsed.

### General Activity

You get this for free on every page where you have included the
`kiss_metrics_tag` included in your layout.

### Identity

See the `kiss_identify` tag above. We use the email address but you could use
a hash of this or the user record ID if you don't want to put the email address
inside a page. We prefer the email address (it's easier to understand what's
happening on a user by user basis) but some folks don't like to put an email
address inside a web page.

### Activation & Sign Up

This gets a bit awkward. We have an `Invite` model that is a bit unusual.
Without going into the details this is what the controller looks like:

```ruby
class InvitesController < InheritedResources::Base
  respond_to :html
  actions :new, :create

  def new
    new! do
      kiss_record "Activated"
    end
  end

  def create
    create! do |success, failure|
      success.html do
        kiss_record "Signed Up"
        sign_in(@invite.user)
        kiss_identify current_user.email
        redirect_to first_page_in_your_post_sign_up_path
      end
    end
  end
end
```

Beyond this you're on your own here, sorry.

### Dev vs Prod

If you don't setup two sites - one for prod and the other for dev - your Prod
site will get polluted with your dev work. Or you can simply disable it. See
the example above for a template.

### Other Stuff

You can add other events to your app by simply stating:

```ruby
kiss_record "Some Other Event"
```

You might want to record for instance the first time someone returns to your
site after they have purchased your product. You can work these events into
Kiss Metrics really easily, with no setup required on the KM side.

See the API section for more details on what tools you have for doing this.


## Contributing to lascivious

* Check out the latest master to make sure the feature hasn't been implemented
  or the bug hasn't been fixed yet.
* Check out the issue tracker to make sure someone already hasn't requested it
  and/or contributed it.
* Fork the project.
* Start a feature/bugfix branch.
* Commit and push until you are happy with your contribution.
* Make sure to add tests for it. This is important so I don't break it in a
  future version unintentionally.
* Please try not to mess with the Rakefile, version, or history. If you want to
  have your own version, or is otherwise necessary, that is fine, but please
  isolate to its own commit so I can cherry-pick around it.

## Copyright

Copyright (c) 2011 Cloudability Inc.

See [[LICENSE]] for further details.

