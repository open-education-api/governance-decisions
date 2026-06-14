# Recommended approach for profiles and consumers

Profiles and consumers should be maintained in their own repository, or in a shared repository containing multiple profiles and consumers. They should not be added directly to the OEAPI repository.

Although OEAPI allows consumers to be defined independently, using a consumer without a profile is not recommended. A standalone consumer only defines consumer-specific additions and does not result in a complete specification. In OEAPI 6.0, the `consumer` property must be specialised by the profile to reference the consumer-specific schema.

Therefore, the recommended approach is to always define a profile, even when only consumer-specific extensions are required.

## Profiles

A profile should comply with the following principles:

- The profile contains a unique profile name.
- The profile contains its own version information, which is added to the resulting specification.
- The profile references the consumers it uses and the versions of those consumers, which are added to the resulting specification.
- The profile only contains overlay files relative to the referenced OEAPI version.
- The profile reuses the referenced OEAPI specification as its base.
- The profile acts as an overlay on top of the referenced OEAPI specification.

## Consumers

A consumer should comply with the following principles:

- The consumer references the minimum OEAPI version on which it is based.
- The consumer defines only the consumer-specific additions.
- The consumer contains its own version information, which is added to the resulting specification.
- The consumer does not duplicate OEAPI definitions.
- The consumer does not contain a complete copy of the resulting specification.
- The consumer acts as an overlay that can be applied by a profile.

## Specification overlay process

A resulting specification should comply with the following principles:

- The resulting specification is based on the referenced OEAPI version.
- The resulting specification includes the profile overlay.
- The resulting specification includes any consumer overlays referenced by the profile.
- The resulting specification contains the profile name, profile version and the included consumer versions.

This approach minimises duplication, simplifies maintenance, and makes it clear which profile version and consumer versions were used to create a particular specification.

## Referencing OEAPI

The recommended approach is to include the OEAPI specification as a Git submodule in a `base` directory. The submodule should reference a specific OEAPI release, tag or branch.

The contents of the submodule should be treated as read-only. Profile and consumer repositories should only contain overlay files relative to the referenced OEAPI specification.

Upgrading to a newer OEAPI version is performed by updating the submodule reference.

## Example repository structure

```text
oeapi-extensions/
├── README.md
├── base/
│   └── oeapi/                           (Git submodule)
├── rio/
│   ├── README.md
│   ├── profile.yaml
│   ├── source/
│   │   ├── consumers/
│   │   │   └── rio/
│   │   │       ├── consumer_version.yaml
│   │   │       ├── CourseRioConsumer.yaml
│   │   │       └── ProgrammeRioConsumer.yaml
│   │   ├── spec.yaml
│   │   ├── paths/
│   │   └── schemas/
│   │       ├── Course.yaml
│   │       └── Programme.yaml
│   └── generated/
│       ├── oeapi_rio.yaml
│       └── oeapi_rio.json
├── eduxchange/
│   ├── README.md
│   ├── profile.yaml
│   ├── source/
│   │   ├── consumers/
│   │   │   └── eduxchange/
│   │   │       ├── consumer_version.yaml
│   │   │       ├── CourseEduXchangeConsumer.yaml
│   │   │       └── ProgrammeEduXchangeConsumer.yaml
│   │   ├── spec.yaml
│   │   ├── paths/
│   │   └── schemas/
│   │       ├── Course.yaml
│   │       └── Programme.yaml
│   └── generated/
│       ├── oeapi_eduxchange.yaml
│       └── oeapi_eduxchange.json
└── ...
```

The `source` directory follows the same structure as the referenced OEAPI specification and only contains overlay files.

The `consumers` directory contains the consumer-specific overlay schemas used by the profile. These schemas should only define the additions required for that consumer.

## Example profile metadata

```yaml
# rio/profile.yaml

profileName: rio
profileVersion: 1.2.0

consumers:
  rio:
    consumerVersion: 2.1.0
    oeapiMinVersion: 6.0
```

## Example consumer metadata

```yaml
# rio/source/consumers/rio/consumer_version.yaml

consumerVersion: 2.1.0
oeapiMinVersion: 6.0
```

## Example profile overlay

```yaml
# rio/source/schemas/Course.yaml

required:
  - courseId
  - name
  - abbreviation
  - description
  - primaryCode
  - level
  - teachingLanguage
  - validFrom
  - educationSpecification
  - duration

properties:
  consumer:
    $ref: '../consumers/rio/CourseRioConsumer.yaml'
```

## Example consumer overlay

```yaml
# rio/source/consumers/rio/CourseRioConsumer.yaml

allOf:
  - $ref: './Consumer.yaml'
  - required:
      - educationOffererCode
      - consentParticipationSTAP
    properties:
      consumerKey:
        const: rio
      educationOffererCode:
        type: string
        description: |
          onderwijsaanbiedercode
          Een betekenisloze identifier voor een onderwijsaanbieder in de Registratie Instellingen en Opleidingen.
        pattern: ^(?:\d{3}A\d{3})$
      consentParticipationSTAP:
        type: string
        description: |
          toestemmingDeelnameSTAP
          Geeft aan of een AangebodenOpleiding beschikbaar is in het kader van de STAP-regeling.
        enum:
          - permission_granted
          - permission_not_granted
```

## Example spec.yaml overlay

```yaml
# rio/source/spec.yaml

info:
  x-profile-name: rio
  x-profile-version: 1.2.0

x-consumers:
  rio:
    version: 2.1.0
    oeapiMinVersion: 6.0
```

## Summary

A profile repository should contain:

- A reference to the OEAPI specification through a Git submodule in the `base` directory.
- A `profile.yaml` file containing the profile metadata.
- Overlay files in the `source` directory containing only profile-specific additions and modifications, such as making attributes required, replacing the `consumer` property with a consumer-specific schema reference, or disabling endpoints by setting them to `null`.
- Generated OpenAPI specifications in YAML and JSON format.

As a result, the repository remains compact and only contains the information required to overlay a specific OEAPI version with profile-specific and consumer-specific changes.

During the overlay process, the referenced OEAPI specification from the `base` submodule, the profile overlay and the referenced consumer overlays are applied to create a complete specification.

Generated specifications are artefacts and should not be maintained manually.