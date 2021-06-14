# Before Running

Before running the core playbook some variables need to be modified:

- `domain`: Should be changed to i-on core DNS name
- `certbot_email_address`: The email address that certbot is going to use for relevant notifications about the generated certificate

We also need to created a `cloudflare_credentials.ini` file in the core resources folder (`ansible/res/core`), that has a Cloudflare API KEY with the necessary scopes to edit the DNS records of the i-on core domain. An example file is as follows:

```ini
# Cloudflare API token used by Certbot
# Generate API KEY in https://dash.cloudflare.com/profile/api-tokens
dns_cloudflare_api_token = cloudflare api key here
```

To connect with the database we must rename `core-db-secrets.yml.example` to `core-db-secrets.yml` (located in `ansible/vars`) and change the necessary values:

```yml
database_user: core
database_db: core
database_password: test
```