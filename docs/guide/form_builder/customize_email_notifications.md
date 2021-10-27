# Customize email notifications [[% include 'snippets/experience_badge.md' %]] [[% include 'snippets/commerce_badge.md' %]]

Email is one of the Submit button options you can add to a form in the Form Builder.
Use it to configure a list of email addresses that get notifications about newly filled forms.

![Email notification](../img/email_notification.png)

## Override email template

To customize the form submission email, override the `form_builder/form_submit_notification_email.html.twig` template.
It contains two blocks: `subject` and `body`.
Each of them is rendered independently and consists of three sets of parameters.

|Parameter|Type|Description|
|---------|----|-----------|
|`content`|`eZ\Publish\API\Repository\Values\Content\Content`|Name of the form, its Content Type|
|`form`|`EzSystems\EzPlatformFormBuilder\FieldType\Model\Form`|Definition of the form|
|`data`|`EzSystems\EzPlatformFormBuilder\FieldType\Model\FormSubmission`|Sent data|  

## Configure sender details

Some email providers require a sender address to be set, so to avoid unsent emails when using Form Builder,
it is recommended to configure `sender_address` in `config/packages/swiftmailer.yaml`.
This email acts as a sender and return address for all bounced messages.
For more information, see [Symfony Mailer Configuration Reference.]([[= symfony_doc =]]/reference/configuration/swiftmailer.html#sender-address)

Add `sender_address` entry to `config/packages/swiftmailer.yaml`:

```yaml
swiftmailer:
    url: '%env(MAILER_URL)%'
    spool: { type: 'memory' }
    sender_address: '%env(MAILER_SENDER_ADDRESS)%'
```

In the `.env` file, define a new environment variable:
`MAILER_SENDER_ADDRESS=mail@example.com`
and configure your mail server connection details in the `MAILER_URL` environmental variable.