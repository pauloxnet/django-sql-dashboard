# Installation and configuration

## Install using pip

Install this library using `pip`:

    $ pip install django-sql-dashboard

## Configuration

Add `"django_sql_dashboard"` to your `INSTALLED_APPS` in `settings.py`.

Add the following to your `urls.py`:

```python
from django.urls import path, include
import django_sql_dashboard

urlpatterns = [
    # ...
    path("dashboard/", include(django_sql_dashboard.urls)),
]
```

## Setting up read-only PostgreSQL credentials

Create a new PostgreSQL user or role that is limited to read-only SELECT access to a specific list of tables.

If your read-only role is called `my-read-only-role`, you can grant access using the following SQL (executed as a privileged user):

```sql
GRANT USAGE ON SCHEMA PUBLIC TO "my-read-only-role";
```
This grants that role the ability to see what tables exist. You then need to grant `SELECT` access to specific tables like this:
```sql
GRANT SELECT ON TABLE
    public.locations_location,
    public.locations_county,
    public.django_content_type,
    public.django_migrations
TO "my-read-only-role";
```
Think carefully about which tables you expose to the dashboard - in particular, you should avoid exposing tables that contain sensitive data such as `auth_user` or `django_session`.

## Configuring the "dashboard" database alias

Django SQL Dashboard defaults to executing all queries using the `"dashboard"` Django database alias.

You can define this `"dashboard"` database alias in `settings.py`. Your `DATABASES` section should look something like this:

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "mydb",
        "USER": "read_write_user",
        "PASSWORD": "read_write_password",
        "HOST": "dbhost.example.com",
        "PORT": "5432",
    },
    "dashboard": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "mydb",
        "USER": "read_only_user",
        "PASSWORD": "read_only_password",
        "HOST": "dbhost.example.com",
        "PORT": "5432",
        "OPTIONS": {
            "options": "-c default_transaction_read_only=on -c statement_timeout=100"
        },
    },
}
```
In addition to the read-only user and password, pay attention to the `"OPTIONS"` section: this sets a statement timeout of 100ms - queries that take longer than that will be terminated with an error message. It also sets it so transactions will be read-only by default, as an extra layer of protection should your read-only user have more permissions that you intended.

Now visit `/dashboard/` as a staff user to start trying out the dashboard.

### Danger mode: configuration without a read-only database user

Some hosting environments such as Heroku charge extra for the ability to create read-only database users. For smaller projects with dashboard access only made available to trusted users it's possible to configure this tool without a read-only account, using the following options:

```python
    # ...
    "dashboard": {
        "ENGINE": "django.db.backends.postgresql",
        "USER": "read_write_user",
        # ...
        "OPTIONS": {
            "options": "-c default_transaction_read_only=on -c statement_timeout=100"
        },
    },
```
The `-c default_transaction_read_only=on` option here should prevent accidental writes from being executed, but note that dashboard users in this configuration will be able to access _all tables_ including tables that might contain sensitive information. Only use this trick if you are confident you fully understand the implications!

## Additional settings

You can customize the following settings in Django's `settings.py` module:

- `DASHBOARD_DB_ALIAS = "db_alias"` - which database alias to use for executing these queries. Defaults to `"dashboard"`.
- `DASHBOARD_ROW_LIMIT = 1000` - the maximum number of rows that can be returned from a query. This defaults to 100.
- `DASHBOARD_UPGRADE_OLD_BASE64_LINKS` - prior to version 0.8a0 SQL URLs used base64-encoded JSON. If you set this to `True` any hits that include those old URLs will be automatically redirected to the upgraded new version. Use this if you have an existing installation of `django-sql-dashboard` that people already have saved bookmarks for.
- `DASHBOARD_ENABLE_FULL_EXPORT` - set this to `True` to enable the full results CSV/TSV export feature. It defaults to `False`. Enable this feature only if you are confident that the database alias you are using does not have write permissions to anything.

## Custom templates

The templates used by `django-sql-dashboard` extend a base template called `django_sql_dashboard/base.html`, which provides Django template blocks named `title` and `content`. You can customize the appearance of your dashboard installation by providing your own version of this base template in your own configured `templates/` directory.
