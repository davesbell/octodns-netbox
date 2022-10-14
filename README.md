## A [NetBox](https://github.com/digitalocean/netbox) source for [octoDNS](https://github.com/github/octodns/)

NetBox is not intended to be used as a full DNS management application. However, with this project, you will have complete control over your DNS records with Netbox!

⚠️ This is a **source** for octoDNS! We can only serve to populate records into a zone, cannot be synced **to** Netbox.

Based on IP address information managed by NetBox, this automatically creating A/AAAA records and their corresponding PTR records. In order to generate these records, Netbox must have the correct information on what FQDNs correspond to IP addresses. We use the `description` field as a comma-separated list of hostnames (FQDNs).

For instance, if you have `192.0.2.1/26` in your Netbox and `en0.host1.example.com,host1.example.com` is entered in its description field, you can use this source as shown in the sample configuration below, you will get three records. That is `en0.host1.example.com. A 192.0.2.1`, `host1.example.com. A 192.0.2.1` and `1.0/26.2.0.192.in-addr.arpa. PTR en0.host1.example.com.`! 🎉

Starting with [Netbox v2.6.0](https://github.com/netbox-community/netbox/issues/166), IPAddress now has a `dns_name` field. If you want to utilize this field, just set `field_name: dns_name` to your configuration. However, [I don't know why](https://github.com/netbox-community/netbox/discussions/9232), this field can only store **single** FQDN. So you would probably be more comfortable using the `description` field as a comma-separated list, since it is not possible to do complex DNS record management.

### Example Configuration

The following config will combine the records in `./config/example.com.yaml` and the dynamically looked up addresses at NetBox.

You must configure `url` and `token` to work with the [NetBox API](https://netbox.readthedocs.io/en/latest/api/overview/).

```yaml
providers:

  config:
    class: octodns.provider.yaml.YamlProvider
    directory: ./config

  netbox:
    class: octodns_netbox.NetboxSource
    # Your Netbox URL
    url: https://ipam.example.com
    # Your Netbox Access Token (read-only)
    token: env/NETBOX_TOKEN
    # The TTL of the generated records (Optional, default: 60)
    ttl: 60
    #
    # !!!!! Advanced Parameters !!!!!
    # Just ignore below and no need to write these lines in your yaml.
    #
    # Generate records including subdomains (Optional, default: `true`)
    # If `false`, only records that belong directly to the zone (domain) will be generated.
    # If you are seeing a lot of `SubzoneRecordException` in your logs, change this to `false`.
    populate_subdomains: true
    # FQDN field name (Optional, default: `description`)
    # The `dns_name` field on Netbox is provided to hold only a single name,
    # but typically one IP address will correspond to multiple DNS records (FQDNs).
    # The `description` does not have any limitations so by default
    # we use the `description` field to store multiple FQDNs, separated by commas.
    # Tested: `description`, `dns_name`
    field_name: description
    # Tag Name (Optional)
    # By default, all records are retrieved from Netbox, but it can be restricted
    # to only IP addresses assigned a specific tag.
    populate_tags:
      - tag_name
      - passing multiple values will result in a logical AND operation
    # VRF ID (Optional)
    # By default, all records are retrieved from Netbox, but it can be restricted
    # to only IP addresses assigned a specific VRF ID.
    # Note that VRF ID here does not refer to RD, but to an number in Netbox.
    populate_vrf_id: 1
    # VRF Name (Optional)
    # VRF can also be specified by name.
    # If there are multiple VRFs with the same name, it would be better to use `populate_vrf_id`.
    populate_vrf_name: mgmt

  route53:
    class: octodns_route53.Route53Provider
    access_key_id: env/AWS_ACCESS_KEY_ID
    secret_access_key: env/AWS_SECRET_ACCESS_KEY

zones:
  example.com.:
    sources:
      - config
      - netbox  # will add A/AAAA records
    targets:
      - route53

  0/26.2.0.192.in-addr.arpa.:
    sources:
      - netbox  # will add PTR records (corresponding to A records)
    targets:
      - route53

  0.8.b.d.0.1.0.0.2.ip6.arpa:
    sources:
      - netbox  # will add PTR records (corresponding to AAAA records)
    targets:
      - route53
```

## Contributing
See [the contributing guide](CONTRIBUTING.md) for detailed instructions on how to get started with our project.

## License
[MIT](https://choosealicense.com/licenses/mit/)
