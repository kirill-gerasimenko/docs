---
description: Import MFA enrollments for your existing users.
topics:
  - mfa
contentType:
  - how-to
useCase:
  - import-mfa
---
# Import Multi-Factor Authenticators

You can import a user's MFA enrollments with [automatic migration](/users/guides/configure-automatic-migration) and [bulk user imports](/users/guides/bulk-user-imports). The supported enrollment types are:

* Email: for [email](/mfa/concepts/mfa-factors#email-notifications) verification.
* Phone: for [SMS](/mfa/concepts/mfa-factors#sms-notifications) verification.
* TOTP: for [One-Time Passwords (OTP)](/mfa/concepts/mfa-factors#one-time-passwords) used with authenticator applications, such as Google Authenticator.

Importing MFA enrollments provides a seamless user experience, since users won't have to re-enroll after migration.

## Schema

The schema applies to MFA factors for both of the aforementioned workflows.

```json
{
    "type": "array",
    "items": {
        "type": "object",
        "properties": {
            "totp": {
                "type": "object",
                "properties": {
                "secret": {
                    "type": "string",
                        "pattern": "^[A-Z2-7]+$",
                        "description": "The OTP secret is used for MFA authentication with Google Authenticator type apps. It must be supplied in un-padded Base32 encoding, such as: JBTWY3DPEHPK3PNP"
                    },
                },
                "additionalProperties": false,
                "required": ["secret"],
            },
            "phone": {
                "type": "object",
                "properties": {
                "value": {
                    "type": "string",
                    "pattern": "^\\+[0-9-()\\s]{8,}$",
                    "description": "The phone number for SMS MFA. The phone number should include a country code and begin with +, such as: +1 (212) 555-0001"
                },
                },
                "additionalProperties": false,
                "required": ["value"],
            },
            "email": {
                "type": "object",
                "properties": {
                    "value": {
                        "type": "string",
                        "format": "email",
                        "description": "The email address for MFA"
                    },
                },
                "additionalProperties": false,
                "required": ["value"],
            },
        },
        "maxProperties": 1,
        "additionalProperties": false,
    },
    "minItems": 1,
    "maxItems": 10
}
```

## Bulk User Import

To begin, prepare a `users.json` file as described [here](/users/guides/bulk-user-imports). Be sure to include any existing MFA enrollments for each user. Next, start a bulk user import (described in more detail [here](/users/guides/bulk-user-imports#request-bulk-user-import)). You can _update_ the factors of any existing users by enabling the `upsert` option in your initial request.

Once the import job completes, check the response for any errors. If any of the users' MFA factors failed to import, you will see errors such as: 

```json
{
  "code": "MFA_FACTORS_FAILED",
  "message": "Unable to import factors"
}
```

When using the `upsert` option, any non-MFA related updates to existing users will have been applied to the user's profile. For example, the following error summary shows the user's `picture` attribute was successfully set to `http://example.org/jdoe.png`, however we were unable to import the provided MFA factors. In cases like this it is safe to retry the import for failed users.

```json
[
  {
    "user": {
      "email": "antoinette@contoso.com",
      "picture": "http://example.org/jdoe.png",
      "mfa_factors": [
        {
          "totp": {
            "secret": "2PRX8ZWZAYYDAWCD"
          }
        },
        {
          "phone": {
            "value": "+1 (555) 111 2233"
          }
        },
        {
          "email": {
            "value": "antoinette@antoinette.biz"
          }
        }
      ]
    },
    "errors": [
      {
        "code": "MFA_FACTORS_FAILED",
        "message": "Unable to import factors"
      }
    ]
  }
]
```

## Automatic Migration

MFA enrollments can also be imported during an [automatic migration](/connections/database/custom-db/overview-custom-db-connections#automatic-migration-scenario). This can be accomplished by providing any existing enrollments in the `mfa_factors` field of the user that is provided to the callback at the end of your custom DB [login script](/connections/database/custom-db/templates/login).

Any failures will appear in your tenant logs as failed logins, and will be distinguishable from other failures by their description: `Unable to import MFA factors`.