# Carbon Accounting Data Interchange Specification (CADI) RFC - DRAFT

## Abstract

This is a draft specification concerning the discovery of carbon intensity data in a given organisation's supply chain.

For carbon accounting solutions to be useful, accurate and complete, and to help organisations fully understand their supply chain, it is essential that data can flow freely between providers of carbon accounting tools and services. This avoids a "walled garden" approach to carbon accounting, where data is locked in proprietary systems and cannot be used by other tools. This specification is intended to provide a common way for tools to discover and exchange carbon accounting data.

If you have any questions about this spec, please contact the maintainer, Tom Hallam (@tom-carbontrail), at hello@carbontrail.co.

## Updates and suggestions

Updates and suggestions are welcomed. Please open a pull request with your change and it will be discussed.

## Notational Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

## Authentication

It is essential that data is only shared with authorised parties.

We reccomend that the OAuth2 `client_credentials` grant type is used, where the client ID and secret are provided in the request body. This is the simplest way to authenticate, and is suitable for machine-to-machine communication.

## Privacy

Providers shall make membership of the CADI network optional to their customers. If a customer chooses not to participate in the CADI network, the provider shall not disclose any information about that customer to other members of the network.

## Identification (CADI-ID)

The CADI exchange is used to produce a static identifier for organisations participating in the network. This identifier is used to identify the organisation in the CADI exchange and for Providers to identify supply chain members for a given organisation.

Providers shall assign a unique CADI-ID to each organisation that participates in the CADI exchange. The CADI-ID format is described below. The CADI-ID shall be assigned to the organisation, not to a specific user of the Provider's system.

To communicate with the CADI exchange, Providers should authenticate using a known `client_id` and `client_secret` in line with the OAuth 2.0 specification.

### Registration

To register a new organisation with the CADI exchange, an authenticated Provider Client shall communicate to the CADI exchange like so:

```
POST https://exchange.cadi-wg.org/v1/organisations
```

With the following information:

- Normalised Organisation Name
- Organisation Category (see Organisation Categories for more information. In NZ, this would be the ANZIC code of the company)
- Organisation Registration Number (Optional. In NZ, this would be the NZBN)

The CADI exchange will then generate a CADI-ID in the form:

```
<CADI-ID-VERSION>:<COUNTRY>:<ANZIC>:<UUID>
```

For example:

```
V1:NZ:2619:bfcd57a9-c16b-4069-8b0d-db976728f1c5
```

The CADI-ID must be stable, given the same inputs, for a given CADI-ID version.

### Query

Participants in the CADI network can query the CADI exchange with a `CADI-ID` to retrieve the organisation and provider details.

```
GET https://exchange.cadi-wg.org/v1/organisations?cadiId=<CADI-ID>
```

The CADI exchange shall respond with the following information, encoded as a JSON object:

```
{
    "version": 1,
    "cadiId": "V1:NZ:2619:bfcd57a9-c16b-4069-8b0d-db976728f1c5",
    "organizationName: "Organization Name",
    "organizationCategory": "2619",
    "organizationRegistrationNumber": "123456789",
    "provider": {
        "name": "Provider Name",
        "cadiId": "V1:NZ:2619:bfcd57a9-c16b-4069-8b0d-db976728f1c5",
        "url": "https://provider.com",
        "logoUrl": "https://provider.com/logo.png"
    }
}
```

## Emissions Factor Discovery

If opted into the CADI network, the Provider must make a file publically available at the following URL:

```
https://<provider-base-url>/.well-known/cadi
```

For example:

```
https://app.carbontrail.co/.well-known/cadi
```

The Data Provider MUST implement a method to filter the returned organisation list to those that the Peer Provider is interested in. If the header `x-cadi-ids` is specified, the list should only contain CADI-IDs that were requested, or an empty list if no matching organisations could be found.

The Provider SHOULD make use of the `If-Modified-Since` request header to only return information that has been modified since the requested date. If this header is present, the Provider MUST only return reporting periods that have been modified since that date.

### Querying Peers

Each Peer Provider SHOULD query each provider that it is aware of (through the CADI-ID query when ingesting a matching organisation). If there is a match, and the data has been updated more recently than the current cache, the Provider may contact the Peer Provider directly to retrieve more in-depth data, if more is required.

Peer Providers MUST NOT query this document more than twice every 24 hours to limit the load on the Peer Provider's infrastructure.

### Discovery Document Format

The file shall enumerate the opted-in companies, their CADI numbers, and the reporting periods that the Provider has information for.

The Provider may this file as it could result in large database queries. If the Provider chooses to cache the file, it must be invalidated daily to ensure the data remains fresh.

For each line, the schema should be as follows:

```
CADI-ID \t ReportingPeriodStart (ISO 8601) \t ReportingPeriodEnd (ISO 8601) \t EmissionsFactor (kgCO2e/$) \t Status
```

An example of such a file is shown below:

```
V1:NZ:0000:bfcd57a9-c16b-4069-8b0d-db976728f1c5 2020-01-01 2020-12-31 0.10 FINALISED
V1:NZ:0000:1fcd57a9-c16b-4069-8b0d-db976728f1c5 2021-01-01 2021-12-31 0.02 INTERIM
```

## Outstanding questions

- Should we just have the Peer Providers update the Exchange on a schedule, so there's one place to go? Or does this centralise too much?
- What happens if multiple providers appear to be serving the same NZBN?
