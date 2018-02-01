# django-loginratelimit

A Django Middleware that detects failed login attempts and incurs an increasing rate-limit after a specified number of attempts. Uses only the cache and never hits the DB.


## Purpose

This Middleware detects failed login attempts per username and incurs a rate-limit specifically for the used credential after n specified attemps.

Out-of-the-package, django does not protect against repeated login attempts. This could be used as an attack vector to try to guess an administrator's account password (if their username is known) by scripting a large number of unnoticed login attempts with a password list. 

This middleware makes this attack almost infeasible by severely limiting the number of attempts, whithout being a great inconvenience for users.

## What it does

By default, the login on URL `/admin/login/` is watched. After 5 failed attempts using a username, no login on the same username can be attempted for 30 seconds per previously limited attempt, up until a maximum of 10 minutes.

During an active rate limit, this middleware prevents *any* authentication attempts on the limited username. This means that the credentials are not checked at all, so not even the correct credentials can be used to log in during a rate limit. Each successive failed login attempt after a rate limit period incurs an even longer rate-limit.
 
After a successful login on that username, all recorded failed attempts are reset.

## Installation

Copy the file `login_ratelimit_middleware.py` to a path in your project and add the following to your settings.py:

```
MIDDLEWARE_CLASSES = (
    ...,
    <path.in.your.project>.login_ratelimit_middleware.LoginRateLimitMiddleware,
)
```

## Settings

By default, this middleware only catches login attempts on the django admin. To monitor your custom login URL, set `LOGIN_RATELIMIT_LOGIN_URLS` in your settings.py and add your URLs there. URLs starting with the specified pattern are being matched. The value for each setting specifies the form field name of the user credential in the login form on that URL.

Example:

```
# {'URL': 'user_field_fieldname', ...}
LOGIN_RATELIMIT_LOGIN_URLS = {
    '/admin/login/': 'username',
    '/login/': 'username',
}
```

If your authentication method does not use `<UserModel>.username` as credential field, set `LOGIN_RATELIMIT_USERNAME_FIELD` in your settings.py!

`LOGIN_RATELIMIT_TRIGGER_ON_ATTEMPT` (default: 5), `LOGIN_RATELIMIT_LIMIT_DURATION_PER_ATTEMPT_SECONDS` (default: 30) and `LOGIN_RATELIMIT_LIMIT_DURATION_MAX_SECONDS` (default: 600) control the rate limit behaviour.

Other settings can be found in `login_ratelimit_middleware.default_settings`.

### Note: 

Unless you override 'admin/login.html' to include the django messages, the rate limit warning message will *not* be displayed in the django admin login interface!

## Other details:

* This middleware only uses the cache and doesn't hit the database at all.
* This middleware uses the `user_login_failed` and `user_logged_in` django signals to monitor failed and successful login attempts.
* The existence of accounts is not exposed because attempts for all user credentials are being rate limited, not only existing ones.

## Signals sent and logging

Signal sent after reaching the rate limit after `LOGIN_RATELIMIT_TRIGGER_ON_ATTEMPT` attempts for a username.

```
login_ratelimit_triggered = dispatch.Signal(providing_args=['username', 'ip'])
```

A logging warning is also issued after reaching the rate limit, check the `LOGIN_RATELIMIT_LOG_ON_LIMIT` and `LOGIN_RATELIMIT_LOGGER_NAME` settings for details.

## Compatibility

Unfortunately, I've only been able to test this with Django 1.8. If there are any incompatibilities with other Django versions, please let me know!
