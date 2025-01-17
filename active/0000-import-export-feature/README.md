- Start Date: 2024-11-12
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Implementation PR(s): 

# Import/Export Feature

## Summary

This feature will add the ability to both export datasets to CSV files and import them back into DataHub from those CSV files, using the UI. Code is already implemented for this feature, though further work may need to be done. This RFC details the Implementation in its current state.

## Motivation

This feature was developed with the intention to mimic the import/export functionality present in Collibra. It can be used for moving datasets between instances of DataHub, which may be useful for enterprise-level users. Though it is not a strictly necessary feature, the DataHub team has expressed interest in adding it to the DataHub project.

## Requirements

This feature as it is currently implemented is only intended to support:
- Export to CSV of individual datasets.
- Export to CSV of all datasets within a container.
- Import from CSV of previously exported data.

## Non-Requirements

This feature is not intended to add a REST API for import/export like that of Collibra. It is only intended for use through the UI.

## Detailed design

This feature will add three new options to the existing `SearchExtendedMenu` dropdown. One to export all datasets within a container, one to export individual datasets, and one to import previously exported data into DataHub. The export options create CSV files from data existing in DataHub, while the import option adds new data to DataHub from CSV files. 

Below is a list of the column names used in the CSV files for this feature. Within the CSV files, each row describes an individual dataset or schema field.

``` csv
resource,asset_type,subresource,glossary_terms,tags,owners,ownership_type,description,domain
```

Here is information on how these CSV columns are used, and how the data stored within them is formatted:

- Resource: The URN of the dataset. In the case of schema fields, this is the URN of the dataset which contains the schema field.
- asset_type: What type of assset is contained in the row. This is either a dataset or schema field.
- subresource: The name of the schema field. This is unused by rows containing datasets.
- glossary_terms: A semicolon-separated list of glossary term URNs. This column is currently unused, but is planned to be used by both dataset and schema field rows.
- tags: A semicolon-separated list of tag URNs. This column is currently unused, but is planned to be used by both dataset and schema field rows.
- owners: A semicolon-separated list of owner URNs. Currently, this is populated on export, but unused on import.
- ownership_type: A list of mappings from owner URN to ownership type. Currently, this column is unused, and its format has yet to be determined.
- description: The description of a given asset. This is used by both dataset and schema field rows.
- domain: The URN of a domain associated with the dataset. This is unused by rows containing schema fields.

### Export

Within the `SearchExtendedMenu` dropdown, the container-level export option is only available when a container is being viewed. At all other times, it is grayed out and cannot be pressed. This is done using a react effect, which greys out the button unless the URL of the current page contains the word "container".

When either export option is selected, it opens a modal which prompts the user to enter the name of the CSV file to be created. For dataset-level export, the user is also prompted to enter the data source, database, schema, and table name of the dataset to be exported. Notably, these fields assume a specific number of containers to be present, which may not be the case for every data source. As such, this modal may need to be altered. This is what the fields presently refer to:
- Data source: The name of the data platform containing the dataset.
- Database: A container representing a database within the data source.
- Schema: A container representing a schema within the source database.
- Table name: The name of the dataset.

Upon entry, the following steps occur:

1. The modal is made invisible, but continues executing code for the export process. A notification is created to inform the user that the export process is ongoing.
2. The URN of the dataset or container is determined, by either:
    - Pulling from [`EntityContext`](https://github.com/datahub-project/datahub/blob/master/datahub-web-react/src/app/entity/shared/EntityContext.ts) in the case of container-level export.
    - Manually constructing the URN from data entered into the modal in the case of dataset-level export.
3. The modal uses the URN as input into GraphQL queries, which are used to fetch the metadata for the datasets to be exported.
    - Container-level export will first execute a GraphQL query to determine how many datasets are present in the container. If no datasets are present, execution will end early, and a notification is sent to the user informing them of such. Datasets are not searched for recursively.
    - Additionally, container-level export will only fetch 50 datasets per GraphQL execution. If more than 50 datasets are present in the container, this query will be executed multiple times, with each execution producing and downloading separate CSV files.
4. The metadata returned from the GraphQL query is transformed into a CSV-compatible JSON object using a shared function, `convertToCSVRows`. Each row in this JSON object contains the columns described in the prior section.
5. The existing `downloadRowsAsCsv` function in [`csvUtils`](https://github.com/datahub-project/datahub/blob/master/datahub-web-react/src/app/search/utils/csvUtils.ts) is used to create the download.

#### GraphQl queries

These GraphQL queries are used for container-level export and dataset-level export, respectively:

``` graphql
query getDatasetByUrn($urn: String!, $start: Int!, $count: Int!) {
    search(input: { type: DATASET, query: $urn, start: $start, count: $count }) {
        start
        count
        total
        searchResults {
            entity {
                ... on Dataset {
                    urn
                    type
                    name
                    platform {
                        urn
                    }
                    domain {
                        associatedUrn
                        domain {
                            urn
                            type
                        }
                    }
                    properties {
                        name
                        description
                    }

                    editableProperties {
                        description
                    }

                    ownership {
                        owners {
                            owner {
                                ... on Entity {
                                    urn
                                }
                            }
                            ownershipType {
                                urn
                            }
                        }
                    }
                    tags {
                        tags {
                            associatedUrn
                        }
                    }
                    glossaryTerms {
                        terms {
                            associatedUrn
                        }
                    }
                    editableSchemaMetadata {
                        editableSchemaFieldInfo {
                            description
                            fieldPath
                            tags {
                                tags {
                                    associatedUrn
                                }
                            }
                        }
                    }
                    schemaMetadata {
                        name
                        fields {
                            description
                            type
                            fieldPath
                            nativeDataType
                            tags {
                                tags {
                                    tag {
                                        name
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}

query getTable($urn: String!, $start: Int!, $count: Int!) {
    search(
        input: {
            type: DATASET
            query: "*"
            start: $start
            count: $count
            orFilters: { and: [{ field: "urn", values: [$urn], condition: EQUAL }] }
        }
    ) {
        start
        count
        total
        searchResults {
            entity {
                ... on Dataset {
                    urn
                    type
                    name
                    platform {
                        urn
                    }
                    domain {
                        associatedUrn
                        domain {
                            urn
                            type
                        }
                    }
                    properties {
                        name
                        description
                    }

                    editableProperties {
                        description
                    }

                    ownership {
                        owners {
                            owner {
                                ... on Entity {
                                    urn
                                }
                            }
                            ownershipType {
                                urn
                            }
                        }
                    }
                    tags {
                        tags {
                            associatedUrn
                        }
                    }
                    glossaryTerms {
                        terms {
                            associatedUrn
                        }
                    }
                    editableSchemaMetadata {
                        editableSchemaFieldInfo {
                            description
                            fieldPath
                            tags {
                                tags {
                                    associatedUrn
                                }
                            }
                        }
                    }
                    schemaMetadata {
                        name
                        fields {
                            description
                            type
                            fieldPath
                            nativeDataType
                            tags {
                                tags {
                                    tag {
                                        name
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}
```

### Import

In the case of import, the button first opens a prompt to upload a file, using the following snippet of code.

``` jsx
<input id="file" type="file" onChange={changeHandler} style={{ opacity: 0 }} />
```

The `papaparse` library is used to parse the CSV file and iterate over each row present within it. The data is then fed into GraphQL mutations to create datasets. Notably, a new GraphQL mutation had to be created to allow the upserting of schema metadata. Here is the specification for that new mutation:

``` graphql
type Mutation {
    upsertDataset(urn: String!, input: DatasetUpsertInput!): Dataset
}

input DatasetUpsertInput {
    name: String!
    description: String
    schemaMetadata: SchemaMetadataInput!
    globalTagUrns: [String],
    domainUrn: String
}

input SchemaMetadataInput {
    version: Int!,
    schemaName: String!,
    platformUrn: String!,
    fields: [SchemaFieldInput]!
}

input SchemaFieldInput {
    fieldPath: String!
    type: SchemaFieldDataType!
    nativeDataType: String!
    globalTagUrns: [String]
    description: String
}
```

Alongside these new GraphQL mutations, the necessary mappers and resolvers have been added to `datahub-graphql-core` to properly send the input to GMS. It's noteworthy that there are several fields required by the GraphQL mutation that are not present in the CSV Schema. Such fields are filled with these values on import:

- `name`: The name is extracted from the dataset URN stored in the `resrource` CSV field.
- `version`: 0
- `schemaName`: An empty string.
- `platformUrn`: The platform URN is extracted from the dataset URN stored in the `resrource` CSV field.
- `type`: `SchemaFieldDataType.Null`
- `nativeDataType`: "Unknown Type"

Presently, the `glossary_terms`, `tags`, `owners`, and `ownership_type` CSV fields are unused in the import of datasets.

## How we teach this

A user guide should be written on how to use the feature. In particular, we will need to highlight:
- Under what circumstances the container-level export button becomes available.
- How to fill out the form presented in the dataset-level export modal.

This feature would be best presented as a wholly new functionality. Though it is presently possible to download search results as CSV, the format of the resulting CSV files differs significantly from that of this import/export feature. As such, the files cannot be used as import input. 

The GraphQL documentation may also need to be updated to include the new mutation added alongside this feature, should the DataHub team decide to make the mutation available via the UI.

## Drawbacks and Alternatives

As mentioned before, this feature is only intended for use within the UI. As the code has currently been written, it would not be possible to extend the import and export functionality to a different API (i.e., REST), as all the code is written in React. 

A possible alternative to this would be to move the code that performs the CSV data mutations to the Metadata Service or the Frontend Server, so that it is accessible from throughout the DataHub stack through a REST API or similar. This does not come without drawbacks, however. Namely, we would need to re-write the existing code entirely, and we'd be introducing additional complexity through the API endpoints.

It's also notable that because the format of the CSV files is so different from those produced by the existing functionality of downloading search results, existing CSV files cannot be used to import datasets. This may cause confusion among users, and may be worth remediating.

It's also notable that a extension for DataHub does exist which adds very similar functionality ([link](https://datahubproject.io/docs/0.13.1/generated/ingestion/sources/csv/)). This has not been investigated in detail, but if this is a duplicate feature, it may not be worth integrating into DataHub.

## Rollout / Adoption Strategy

This feature does not change or break any existing functionality, and therefore no specific migration strategy is required.

## Future Work

Out of the required GraphQL fields that are presently missing on import, the `type` and `nativeDataType` fields are the only ones that have a noticeable visual impact within DataHub when absent. Because of this, it is an absolute necessity that we add corresponding columns to the CSV schema, so that we can populate those fields. Notably, if we add a `type` column, it could also be uased to store the sub types of datasets.

Additionally, the `glossary_terms`, `tags`, and `ownership_type` CSV columns are presently unused. It would be fairly simple to add the functionality to fill those columns in, however, as we are already fetching the necessary information for these fields in the search results of our GraphQL query. At the same time, the import code should be updated to make use of the `owners` column. To upsert this data to DataHub, either the `upsertDataset` mutation would need to be updated to handle the new information, or additional GraphQL mutations would need to be performed during import using existing mutations.

As talked about in further detail below, the dataset-level export will also need to be refactored to be more flexible, as at present, it was designed to only work with data sources with two laters of containers in DataHub. It remains to be decided how it should be redesigned.

## Unresolved questions

It's notable that the dataset-level export component of this feature was designed specifically for data sources with two layers of containers in DataHub. This is unlikely to always be the case, and as such, this component will likely need to be refactored to be more flexible. We will need to determine what shape the component should take before performing this refactoring.

Additionally, this feature would end up adding the ability to create Datasets through GraphQL as a side effect. It will need to be evaluated whether this is an acceptable outcome, or if it is acceptable, whether it should be made accessible through the GraphiQL interface.