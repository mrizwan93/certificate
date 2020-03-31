# Certificate System Role - API Design proposal

Linux system role to issue and manage SSL certificates.

Basic usage:

```yaml

---
- hosts: webserver

  vars:
    certificate_requests:
      - name: mycert
        common_name: *.example.com
        self_signed: yes

  roles:
    - certificate
```

On a RPM-based system this will place the certificate in `/etc/pki/tls/certs/mycert.crt`
and the key in `/etc/pki/tls/private/mycert.key`.


## Parameters

| Parameter               | Description                                                                                                    | Type | Required | Default           |
|-------------------------|----------------------------------------------------------------------------------------------------------------|:----:|:--------:|-------------------|
| certificate_wait        | If the should wait for the certificate to be issued.                                                           | bool | no       | yes               |
| certificate_provider    | The underlying method used to request and manage the certificate. See [Providers and CAs](#providers-and-cas). | str  | no       | -                 |
| certificate_requests    | A list of dicts representing each certificate to be issued. See [certificate_requests](#certificate_requests). | list | no       | -                 |
| certificate_install_certmonger | Install certmonger and configure it to be a provider.                                                   | bool | no       | yes               |


### certificate_requests

| Parameter               | Description                                                                          | Type | Required | Default |
|-------------------------|--------------------------------------------------------------------------------------|:----:|:--------:|---------|
| name                    | Name of the certificate.                                                             | str  | yes      | -       |
| common_name             | Hostname to be used in the certificate subject.                                      | str  | yes      | -       |
| self_signed             | Issue a self-signed certificate. This option cannot be used if `ca` is also set.     | bool | no       | -       |
| key_size                | Generate keys with a specific keysize in bits.                                       | int  | no       | 2048    |
| auto_renew              | Indicates if the certificate should be renewed automatically before it expires.      | bool | no       | yes     |
| ca                      | CA that will issue the certificate. Options vary by [Provider](#providers-and-cas).  | str  | no       | -       |
| certificate_file        | Full path of certificate to be issued.                                               | str  | no       | -       |
| key_file                | Full path to private key file to be issued.                                          | str  | no       | -       |
| dns_alternative_names   | Domain names to be present in the certificate request in SAN field.                  | list | no       | -       |
| email_alternative_names | Email addresses to be present in the certificate request in SAN field.               | list | no       | -       |
| ip_alternative_names    | IP addresses to be present in the certificate request in SAN field.                  | list | no       | -       |
| country                 | Country field of certificate request.                                                | str  | no       | -       |
| state                   | State field of certificate request.                                                  | str  | no       | -       |
| locality                | Locality field of certificate request (usually city).                                | str  | no       | -       |
| organization            | Organization field of certificate (usually company name).                            | str  | no       | -       |
| organizational_unit     | Organizational unit field of certificate request.                                    | str  | no       | -       |
| email                   | Contact email that will be present in the certificate.                               | str  | no       | -       |
| key_usage               | Key Usage attributes to be present in the certificate request.                       | list | no       | -       |
| extended_key_usage      | Extended Key Usage attributes to be present in the certificate request.              | list | no       | -       |
| run_before              | Command that should run before saving the certificate.                               | str  | no       | -       |
| run_after               | Command that should run after saving the certificate.                                | str  | no       | -       |
| principal               | Kerberos principal.                                                                  | str  | no       | -       |


## Providers and CAs

| provider   | Available CA | CA description                                                                                   |
|------------|--------------|--------------------------------------------------------------------------------------------------|
| certmonger | local        | Issue self-signed certificates from a local CA.                                                  |
| certmonger | ipa          | Issue certificates using the FreeIPA CA. **Requires the node be enrolled in an FreeIPA server.** |


## Facts

This role provide facts that can be useful for conditional control in
playbooks or event for debuging.


### certificate_enabled_providers

A dict with the providers available in the host. Each provider has a
respective list of enabled CAs for that host.

If any CA is listed in [Providers and CAs](#providers-and-cas) but
it's not showing in the `certificate_enabled_providers` fact it means that,
this CA doesn't meet the requirements to be enabled.


For example if host **is not** enrolled in an IPA server:

```json
{
    "certificate_enabled_providers": {
        "certmonger": [
            "local"
        ]
    }
}
```

If host is enrolled in an IPA server:

```json
{
    "certificate_enabled_providers": {
        "certmonger": [
            "local",
            "ipa"
        ]
    }
}
```


## Examples:


### Minimal to get started - Self-sign certificate:

Issue a certificate for `*.example.com` and place it in the standard
directory for the distribution.


```yaml

---
- hosts: webserver

  vars:
    certificate_requests:
      - name: mycert
        common_name: *.example.com
        self_signed: yes

  roles:
    - certificate
```

An equivalent form would be:

```yaml

---
- hosts: webserver

  vars:
    certificate_provider: certmonger
    certificate_requests:
      - name: mycert
        common_name: *.example.com
        ca: local

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
        common_name: *.example.com
        self_signed: yes
        certificate_file: /another/path/other-cert-name.crt
        key_file: /another/path/other-cert-name.key

  roles:
    - certificate
```

### Certificates with one or more SAN (Subject Alternative Name)

```yaml

---
- hosts: webserver
  vars:
    certificate_requests:
      - name: mycert
        common_name: www.example.com
        self_signed: yes
        dns_alternative_names:
          - sub1.example.com
          - sub2.example.com
          - sub3.example.com
        ip_alternative_names:
          - 192.168.0.12
          - 192.168.0.65
        email_alternative_names:
          - sysadmin@example.com
          - support@example.com

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
        common_name: www.example.com
        self_signed: yes
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
        common_name: www.example.com
        self_signed: yes
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
        common_name: www.example.com
        self_signed: yes
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
        common_name: www.example.com
        self_signed: yes
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
        common_name: www.example.com
        self_signed: yes
        principal: HTTP/www.example.com

  roles:
    - certificate
```


### Choosing to not auto-renew a certificate

By default certificates generated by the role will be set for auto-rewal.
To disable that behavior set `auto_renew: no`.

```yaml

---
- hosts: webserver
  vars:
    certificate_requests:
      - name: mycert
        common_name: www.example.com
        self_signed: yes
        auto_renew: no

  roles:
    - certificate
```


### Using FreeIPA to issue a certificate

If your host is enrolled in an FreeIPA server you will also have the option
to use it's CA to issue your certificate. To do that just set `ca: ipa`.

Note: `ipa` CA is only available when using certmonger as
`certificate_provider`.

```yaml

---
- hosts: webserver
  vars:
    certificate_provider: certmonger
    certificate_requests:
      - name: mycert
        common_name: www.example.com
        ca: ipa

  roles:
    - certificate
```

In case you want to use FreeIPA but fallback to self-signed if the host
is not enrolled:

```yaml

- hosts: webserver
  vars:
    certificate_requests:
      - name: mycert
        common_name: www.example.com
        ca: "{{ 'ipa' if 'ipa' in certificate_enabled_providers.get("certmonger", []) else 'local' }}"
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
        common_name: www.example.com
        self_signed: yes
        run_before: systemctl stop webserver.service
        run_after: systemctl start webserver.service

  roles:
    - certificate
```

[modeline]: # ( vim: set ts=2 sts=2 et: )
