# Certificate System Role - API Design proposal

Linux system role to issue and renew SSL certificates.

Basic usage:

```yaml

---
- hosts: webserver

  vars:
    certificate_requests:
      - name: mycert
        dns: www.example.com
        ca: self-sign

  roles:
    - certificate
```

On a RPM-based system this will place the certificate in `/etc/pki/tls/certs/mycert.crt`
and the key in `/etc/pki/tls/private/mycert.key`.


## Variables

| Parameter               | Description                                                                                                    | Type | Required | Default           |
|-------------------------|----------------------------------------------------------------------------------------------------------------|:----:|:--------:|-------------------|
| certificate_wait        | If the task should wait for the certificate to be issued.                                                      | bool | no       | yes               |
| certificate\_requests   | A list of dicts representing each certificate to be issued. See [certificate_requests](#certificate_requests). | list | no       | -                 |


### certificate_requests

**Note:** Be aware that the CA might not honor all the requested fields.
For example, even if a request include `country: US`, the CA might issue
the certificate without `country` in it's subject.

| Parameter            | Description                                                                                       | Type        | Required | Default                 |
|----------------------|---------------------------------------------------------------------------------------------------|:-----------:|:--------:|-------------------------|
| name                 | Name of the certificate.                                                                          | str         | yes      | -                       |
| ca                   | CA that will issue the certificate. See [CAs and Providers](#cas-and-providers).                  | str         | yes      | -                       |
| dns                  | Domain (or list of domains) to be included in the certficate.                                     | str or list | no       | -                       |
| email                | Email (or list of emails) to be included in the certficate.                                       | str or list | no       | -                       |
| ip                   | IP address (or list of IP addresses) to be included in the certficate.                            | str or list | no       | -                       |
| certificate\_file    | Full path of certificate to be issued.                                                            | str         | no       | -                       |
| key\_file            | Full path to private key file to be issued.                                                       | str         | no       | -                       |
| auto_renew           | Indicates if the certificate should be renewed automatically before it expires.                   | bool        | no       | yes                     |
| user                 | User name (or user id) for the certficate and key files.                                          | str         | no       | *User running Ansible*  |
| group                | Group name (or group id) for the certficate and key files.                                        | str         | no       | *Group running Ansible* |
| key\_size            | Generate keys with a specific keysize in bits.                                                    | int         | no       | 3072 - See [key_size](#key_size) |
| common\_name         | Common Name requested for the certificate subject.                                                | str         | no       | See [common_name](#common_name)  |
| country              | Country requested for the certificate subject.                                                    | str         | no       | -                       |
| state                | State requested for the certificate subject.                                                      | str         | no       | -                       |
| locality             | Locality requested for the certificate subject (usually city).                                    | str         | no       | -                       |
| organization         | Organization requested for the certificate subject.                                               | str         | no       | -                       |
| organizational_unit  | Organizational unit requested for the certificate subject.                                        | str         | no       | -                       |
| contact\_email       | Contact email requested for the certificate subject.                                              | str         | no       | -                       |
| key\_usage           | Allowed Key Usage for the certificate. For valid values see: [key\_usage](#key_usage).            | list        | no       | -                       |
| extended\_key\_usage | Extended Key Usage attributes to be present in the certificate request.                           | list        | no       | -                       |
| run\_before          | Command that should run before saving the certificate.                                            | str         | no       | -                       |
| run\_after           | Command that should run after saving the certificate.                                             | str         | no       | -                       |
| principal            | Kerberos principal.                                                                               | str         | no       | -                       |
| provider             | The underlying method used to request and manage the certificate.                                 | str         | no       | *Varies by CA*          |


### common_name

If `common_name` is not set the role will attempt to use the first
value of `dns` or `ip`, respectively, as the default. If `dns` and
`ip` are also not set, `common_name` will not be included in the certificate.

If you want to ensure that `common_name` doesn't appear in the
certificate you can set it to `null`. For example:

```yaml

---
- hosts: webserver

  vars:
    certificate_requests:
      - name: mycert
        dns: www.example.com
        ca: self-sign
        common_name: null

  roles:
    - certificate
```


### key_size

Recommended minimal-values for a certificate key size, from different
organizations, vary across time. In the attempt to provide safe settings,
the default minimal-value for `key_size` will be increased over time.

If you want your certificates to always keep the same `key_size` when
renewed, set this variable to the desired value.


### key_usage

Valid values for `key_usage` are:

* digitalSignature
* nonRepudiation
* keyEncipherment
* dataEncipherment
* keyAgreement
* keyCertSign
* cRLSign
* encipherOnly
* decipherOnly


## CAs and Providers

| CA               | Providers   | CA description                                  | Requirements                                    |
|------------------|-------------|-------------------------------------------------|-------------------------------------------------|
| self&#x2011;sign | certmonger* | Issue self-signed certificates from a local CA. |                                                 |
| ipa              | certmonger* | Issue certificates using the FreeIPA CA.        | Host needs to be enrolled in a FreeIPA server.  |

 *\* Default provider.*

CA represents the CA certificates that will be used to issue and sign the
requested certificate. Provider it's the method used to send the certificate
request to the CA and then retrieve the signed certificate.

If a user chooses `self-sign` CA, with `certmonger` as provider and, later on
decide to change the provider to `openssl`, the CA certificates used in both
cases needs to be the same. *Please note that `openssl` is **not yet a supported**
provider and it's only mentioned here as an example.*


## Facts

This role provide facts that can be useful for conditional control in
playbooks or event for debugging.

### certificate_available_cas

A list of available CAs in the host.

If any CA is listed in [CAs and Providers](#cas-and-providers) but
it's not showing in the `certificate_available_cas` fact, it means that
this CA doesn't meet the requirements to be enabled.


For example if host **is not** enrolled in an IPA server:

```json
{
    "certificate_available_cas": [
        "self-sign"
    ]
}
```

If host is enrolled in an IPA server:

```json
{
    "certificate_available_cas": [
        "self-sign",
        "ipa"
    ]
}
```


## Examples:


### Issue a self-signed certificate:

Issue a certificate for `*.example.com` and place it in the standard
directory for the distribution.


```yaml

---
- hosts: webserver

  vars:
    certificate_requests:
      - name: mycert
        dns: *.example.com
        ca: self-sign

  roles:
    - certificate
```

The directories for each distribution are:

* Debian/Ubuntu:
  * Certificates: `/etc/ssl/localcerts/certs/`
  * Keys: `/etc/ssl/localcerts/private/`

* RHEL/CentOS/Fedora:
  * Certificates: `/etc/pki/tls/certs/`
  * Keys: `/etc/pki/tls/private/`


### Choose where the certificates will be placed

Issue a certificate and key and place them in the specified location.

```yaml

---
- hosts: webserver
  vars:
    certificate_requests:
      - name: mycert
        dns: *.example.com
        ca: self-sign
        certificate_file: /another/path/other-cert-name.crt
        key_file: /another/path/other-cert-name.key

  roles:
    - certificate
```

### Certificates with multiple DNS, IP and Email

```yaml

---
- hosts: webserver
  vars:
    certificate_requests:
      - name: mycert
        dns:
          - www.example.com
          - sub1.example.com
          - sub2.example.com
          - sub3.example.com
        ip:
          - 192.168.0.12
          - 192.168.0.65
        email:
          - sysadmin@example.com
          - support@example.com
        ca: self-sign

  roles:
    - certificate
```


### Setting common subject options

```yaml

---
- hosts: webserver
  vars:
    certificate_requests:
      - name: mycert
        dns: www.example.com
        common_name: www.example.com
        ca: self-sign
        country: US
        state: NY
        locality: New York
        organization: Red Hat
        organizational_unit: platform
        email: admin@example.com
  roles:
    - certificate
```


### Setting the certificate key size

```yaml

---
- hosts: webserver
  vars:
    certificate_requests:
      - name: mycert
        dns: www.example.com
        ca: self-sign
        key_size: 4096
  roles:
    - certificate
```


### Setting the "Key Usage" and "Extended Key Usage" (EKU)


```yaml

---
- hosts: webserver
  vars:
    certificate_requests:
      - name: mycert
        dns: www.example.com
        ca: self-sign
        key_usage:
          - digitalSignature
          - nonRepudiation
          - keyEncipherment
        extended_key_usage:
          - 'id-kp-clientAuth'
          - 'id-kp-serverAuth'
  roles:
    - certificate
```


### Don't wait for the certificate to be issued

The certificate issuance can take several minutes depending on the CA.
Because of that it's also possible to request the certificate but not
actually wait for it.

```yaml

---
- hosts: webserver
  vars:
    certificate_wait: false
    certificate_requests:
      - name: mycert
        dns: www.example.com
        ca: self-sign
  roles:
    - certificate
```


### Set a principal

```yaml

---
- hosts: webserver
  vars:
    certificate_requests:
      - name: mycert
        dns: www.example.com
        ca: self-sign
        principal: HTTP/www.example.com

  roles:
    - certificate
```


### Choosing to not auto-renew a certificate

By default certificates generated by the role will be set for
auto-renewal. To disable that behavior set `auto_renew: no`.

```yaml

---
- hosts: webserver
  vars:
    certificate_requests:
      - name: mycert
        dns: www.example.com
        ca: self-sign
        auto_renew: no

  roles:
    - certificate
```


### Using FreeIPA to issue a certificate

If your host is enrolled in a FreeIPA server you will also have the option
to use it's CA to issue your certificate. To do that just set `ca: ipa`.

```yaml

---
- hosts: webserver
  vars:
    certificate_requests:
      - name: mycert
        dns: www.example.com
        ca: ipa

  roles:
    - certificate
```

In case you want to use FreeIPA but fallback to self-sign if the host
is not enrolled:

```yaml

- hosts: webserver
  vars:
    certificate_requests:
      - name: mycert
        dns: www.example.com
        ca: "{{ 'ipa' if 'ipa' in certificate_available_cas else 'self-sign' }}"
  roles:
    - certificate
```


### Running a command before or after a certificate is issued


Sometime is used to execute a command just before a certificate get
renewed and another command just after. In order to do that use
`run_before` and `run_after`.

```yaml

---
- hosts: webserver
  vars:
    certificate_requests:
      - name: mycert
        dns: www.example.com
        ca: self-sign
        run_before: systemctl stop webserver.service
        run_after: systemctl start webserver.service

  roles:
    - certificate
```

### Setting the certificate user and group

If you are using a certificate for a service, for example httpd,
it's likely that you'll need to set the certificate user and group
that will own the certificate. In the following example the
user and group are both set to httpd.


```yaml

---
- hosts: webserver
  vars:
    certificate_requests:
      - name: mycert
        dns: www.example.com
        ca: self-sign
        user: httpd
        group: httpd

  roles:
    - certificate
```

Please note that it's also possible to use UID and GID instead of
user and group names.


[modeline]: # ( vim: set ts=2 sts=2 et: )
