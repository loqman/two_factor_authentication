# Two factor authentication for Devise

[![Build Status](https://travis-ci.org/Houdini/two_factor_authentication.svg?branch=master)](https://travis-ci.org/Houdini/two_factor_authentication)
[![Code Climate](https://codeclimate.com/github/Houdini/two_factor_authentication.png)](https://codeclimate.com/github/Houdini/two_factor_authentication)

This fork is made to work with mongoid.

## Features

* configurable OTP code digit length
* configurable max login attempts
* customizable logic to determine if a user needs two factor authentication
* customizable logic for sending the OTP code to the user
* configurable period where users won't be asked for 2FA again
* option to encrypt the OTP secret key in the database, with iv and salt

## Configuration

### Initial Setup

In a Rails environment, require the gem in your Gemfile:

    gem 'two_factor_authentication'

Once that's done, run:

    bundle install

Note that Ruby 2.0 or greater is required.

### Installation

#### Automatic initial setup
Not possible with mongoid for the moment. However, the manual setup is pretty simple!

#### Manual initial setup
If you prefer to set up the model and migration manually, add the
`:two_factor_authentication` option to your existing devise options, such as:

```ruby
devise :database_authenticatable, :registerable, :recoverable, :rememberable,
       :trackable, :validatable, :two_factor_authenticatable
```

Then add the needed fields and index to you model:
```ruby
# Two factor authenticable
field :second_factor_attempts_count, type: Integer, default: 0
field :last_otp_lock, type: Time
field :encrypted_otp_secret_key
field :encrypted_otp_secret_key_iv
field :encrypted_otp_secret_key_salt
index({ encrypted_otp_secret_key: 1 }, { unique: true })
```

#### Complete the setup
Create the index with:

    bundle exec rake db:mongoid:create_indexes

Add the following line to your model to fully enable two-factor auth:

    has_one_time_password(encrypted: true)

Set config values in `config/initializers/devise.rb`:

```ruby
config.max_login_attempts = 3  # Maximum second factor attempts count.
config.unlock_otp_after_seconds = 0 # Time to wait before lock timeout. 0 disables automatic unlock.
config.allowed_otp_drift_seconds = 30  # Allowed time drift between client and server.
config.otp_length = 6  # OTP code length
config.remember_otp_session_for_seconds = 30.days  # Time before browser has to enter OTP code again. Default is 0.
config.otp_secret_encryption_key = ENV['OTP_SECRET_ENCRYPTION_KEY']
```
The `otp_secret_encryption_key` must be a random key that is not stored in the
DB, and is not checked in to your repo. It is recommended to store it in an
environment variable, and you can generate it with `bundle exec rake secret`.

Override the method to send one-time passwords in your model. This is
automatically called when a user logs in:

```ruby
def send_two_factor_authentication_code
  # use Model#otp_code and send via SMS, etc.
end
```

### Customisation and Usage

By default, second factor authentication is required for each user. You can
change that by overriding the following method in your model:

```ruby
def need_two_factor_authentication?(request)
  request.ip != '127.0.0.1'
end
```

In the example above, two factor authentication will not be required for local
users.

This gem is compatible with [Google Authenticator](https://support.google.com/accounts/answer/1066447?hl=en).
You can generate provisioning uris by invoking the following method on your model:

```ruby
user.provisioning_uri # This assumes a user model with an email attribute
```

This provisioning uri can then be turned in to a QR code if desired so that
users may add the app to Google Authenticator easily.  Once this is done, they
may retrieve a one-time password directly from the Google Authenticator app as
well as through whatever method you define in
`send_two_factor_authentication_code`.

#### Overriding the view

The default view that shows the form can be overridden by adding a
file named `show.html.erb` (or `show.html.haml` if you prefer HAML)
inside `app/views/devise/two_factor_authentication/` and customizing it.
Below is an example using ERB:


```html
<h2>Hi, you received a code by email, please enter it below, thanks!</h2>

<%= form_tag([resource_name, :two_factor_authentication], :method => :put) do %>
  <%= text_field_tag :code %>
  <%= submit_tag "Log in!" %>
<% end %>

<%= link_to "Sign out", destroy_user_session_path, :method => :delete %>

```

#### Updating existing users with OTP secret key

If you have existing users that need to be provided with a OTP secret key, so
they can use two factor authentication, create a rake task. It could look like this one below:

```ruby
desc 'rake task to update users with otp secret key'
task :update_users_with_otp_secret_key  => :environment do
  User.find_each do |user|
    user.otp_secret_key = ROTP::Base32.random_base32
    user.save!
    puts "Rake[:update_users_with_otp_secret_key] => OTP secret key set to '#{key}' for User '#{user.email}'"
  end
end
```
Then run the task with `bundle exec rake update_users_with_otp_secret_key`

#### Adding the OTP encryption option to an existing app

1. Follow the manual installation
2. Create and run a task to create users secret keys, such as
   ```ruby
   desc 'rake task to update users with otp secret key'
   task update_users_with_otp_secret_key: :environment do
     User.each do |user|
       user.otp_secret_key = ROTP::Base32.random_base32
       user.save!
     end
   end
   ```

#### Executing some code after the user signs in and before they sign out

In some cases, you might want to perform some action right after the user signs
in, but before the OTP is sent, and also right before the user signs out. One
scenario where you would need this is if you are requiring users to confirm
their phone number first before they can receive an OTP. If they enter a wrong
number, then sign out or close the browser before they confirm, they won't be
able to confirm their real number. To solve this problem, we need to be able to
reset their unconfirmed number before they sign out or sign in, and before the
OTP code is sent.

To define this action, create a `#{user.class}OtpSender` class that takes the
current user as its parameter, and defines a `#reset_otp_state` instance method.
For example, if your user's class is `User`, you would create a `UserOtpSender`
class, like this:
```ruby
class UserOtpSender
  def initialize(user)
    @user = user
  end

  def reset_otp_state
    if @user.unconfirmed_mobile.present?
      @user.update(unconfirmed_mobile: nil)
    end
  end
end
```
If you have different types of users in your app (for example, User and Admin),
and you need different logic for each type of user, create a second class for
your admin user, such as `AdminOtpSender`, with its own logic for
`#reset_otp_state`.

### Example App

[TwoFactorAuthenticationExample](https://github.com/Houdini/TwoFactorAuthenticationExample)
